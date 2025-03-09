# Description

This repo is meant to provide a short overview of a deployed Eclipse Hono and Ditto instance.
For this purpose we will be using the following setup:

- A local deployment of the services using k3d

# Dependencies
Here we will go through the list of necessary dependencies you need to install in order to get this demo running.

## Docker
There lis more than enough documentation for docker already available, but I will do the courtesy of leaving a link
to the [installation guide in their documentation](https://docs.docker.com/engine/install/)

## Kubectl
Next up is our trusted friend kubectl.
Again the story will be the same and I will leave a link to their [installation doc](https://kubernetes.io/docs/tasks/tools/).

## Helm
After docker and kubectl we have our package manager for kubernetes.
It is the same story -> Go to their [documentation](https://helm.sh/docs/intro/install/) and install the tools as described for your OS.

## k3d
In order to avoid having to install [k3s](https://k3s.io/) directly onto your system it is wise to use a tool like k3d.
It will take care and actually containerize your k3s server inside docker giving you the benefit of having a sandbox k3s.
You can create and destroy the cluster any amount of times and even have multi-node setups.

Here I would like to emphasize on removing the `traefik` loadbalancer it gets installed with as it can cause problems if
you have stuff like nginx install on your system and start fighting for port 8080.

To not have this issue please create your cluster like this:

```shell
  k3d cluster create --k3s-arg "--disable=traefik@server:0"
```

## Mosquitto_pub and Mosquitto_sub
The popular mosquitto MQTT clients installation guide can be found [here](https://mosquitto.org/download/).

# Installation of Eclipse Packages
[Eclipse Packages](https://github.com/eclipse/packages) is a wonderful project where you can get your hands on a Helm chart
that includes both [Eclipse Hono](https://projects.eclipse.org/projects/iot.hono) and [Eclipse Ditto](https://eclipse.dev/ditto/).
The helm chart will install all the necessary services and setup everything for you:
- Installs Hono and Ditto
- Configures Kafka as a message broker between them
- Setups a connection Hono <-> Kafka and Ditto <-> Kafka
- Creates necessary certificates for the Hono connection adapters
- Creates a demo device with the necessary credentials to connect to Hono and that way to Ditto respectively

## Installation Steps for Eclipse Packages
Note that here we are using a NodePort approach in kubernetes as opposed to LoadBalancer.
Since this is something normally supported by managed Kubernetes inside cloud providers we will skip it.
The focus is to have a working demo system locally and not go full on production mode!

```shell
  NS=cloud2edge
  kubectl create namespace $NS
```

For the helm chart install we will be removing the limits for the ditto swagger-ui pod currently this causes the pod to break.
I've created a local values.yaml to fix the issue as the cloud2edge helm chart isn't very well-supported nowadays.
```shell
  RELEASE=c2e
  helm install -n $NS -f values.yaml --wait --timeout 20m $RELEASE eclipse-iot/cloud2edge
```

# Preparing the Environment
## Local Envs
Here we will be downloading the [setCloud2EdgeEnv script](https://www.eclipse.org/packages/packages/cloud2edge/scripts/setCloud2EdgeEnv.sh)
in order to set up all local environment variables that will make running our commands easier

```shell
  curl https://www.eclipse.org/packages/packages/cloud2edge/scripts/setCloud2EdgeEnv.sh \
    --output setCloud2EdgeEnv.sh
  chmod u+x setCloud2EdgeEnv.sh

  RELEASE=c2e
  NS=cloud2edge
  # file path that the Hono example truststore will be written to
  TRUSTSTORE_PATH=/tmp/c2e_hono_truststore.pem
  eval $(./setCloud2EdgeEnv.sh $RELEASE $NS $TRUSTSTORE_PATH)
```
Now your environment should be ready to go :)
You can see them using the `env` command in the shell.
A small test you can do is by executing the following command:
```shell
  echo $DITTO_API_BASE_URL
```
Go to the URL displayed in your browser.You should see the Ditto UI opening:
![Ditot UI](/Assets/ditto-ui.png)

## Ditto UI Overview and Config
As shown in the image above we have 2 hyperlinks - one leading to the Swagger-UI of Ditto and to the Ditto UI itself.
Since swagger should be a straightforward concept to readers here we will leave it as it is - if anyone wants to look and
see all available Ditto API and dive deeper they are very welcome to do so :)

### Ditto UI Environment
Once in the UI we will have to Authorize ourselves and for that reason we have to do the following:
1. Click on the Environments Tab
2. Select Local Ditto as an environment
3. Click on Edit
4. Copy the IP with the port inside the browser
5. Click on Update

The completed setup should look like this:
![Ditto UI Environments](/Assets/ditto-ui-environments.png)

### Ditto UI Authorization
After that is done we have to Authorize ourselves against Ditto.
The current setup uses basic auth so the steps are quite straighforward:

1. Click the Authorize button in the UI top right corner
2. For basic auth of the ditto user please use user `ditto` and password `ditto`
3. For devops user you can find it out by calling `echo $DITTO_DEVOPS_PWD` and inputting that output
4. Afterward click on Authorize

![Ditto UI Authorize](/Assets/ditto-ui-authorize.png)

### Ditto UI Walkthrough
Then you just have to click through the different tabs like ***Things***, ***Policies*** and ***Connections***.
1. You will respectively see the configured digital twin(Thing)
![Ditto UI Thing](/Assets/ditto-ui-thing.png)
You can see its different aspects like the **Attributes** and **Features**
2. You can copy the `policyId` in the top right corner and go to the ***Policies*** tab where you can paste it
and load its contents:
![Ditto UI Thing](/Assets/ditto-ui-policies.png)
We can see the policies consists of two *Policy Entries* which actually are the access to the thing we gave **DEFAULT**(nginx)
which does our basic auth allowing for HTTP calls to read/write on the Thing and **HONO** which allows the connection to
kafka to read/write for the Thing.
3. Clicking on the ***Connections*** we will see our configured Kafka connection:
![Ditto UI Kafka](/Assets/ditto-ui-kafka-1.png)
This picture shows the Configured *Sources* and *Targets* that respectively consume and push data to Kafka.
As this is a very broad topic I will restrain myself with only this information as more information can be found
in the official [Ditto Documentation](https://eclipse.dev/ditto/basic-connections.html)

## MonsterMock Server Set up(mmock)
For one of our scenarios we will set up a Monster Mock http server that will give us responses, and we will push out data
from Ditto to it in order to simulate an environment where you push data from Ditto to your own backends/services.

Run the following command inside the repository folder:
```shell

kubectl apply -f mmock/mmock.yaml
```


# Sending Messages from Hono to Ditto
Now that the basics are out of the way, configured and working we can move on to sending messages and updating our digital
twin. The flow of messages we will be observing will be:
- from the device(MQTT) to Hono which will relay to Kafka and the messages will be consumed by Ditto
- First message will have a QoS 0(Quality of Service) -> QoS 0 is an at most once delivery!
- Second message will be with a QoS 1 -> this is an at least once delivery!
- We will send a command and listen for it from the device side
- We will create a HTTP Push connection to our deployed MonsterMock server inside the kubernetes cluster
- We will observe the message being pushed out outside of Ditto and two the HTTP server

A sample architecture diagram of the described flow can be found below:
![Hono Ditto Message Flow](/Assets/ditto-hono-architecture)

## Publishing Telemetry Data (QoS 0)
Here we can either use HTTP or MQTT to push the data from a potential device to Hono and respectively Ditto.


The demo device’s digital twin supports a temperature property which will be set to 45 by means of the following command:
```shell

curl -i -k -u demo-device@org.eclipse.packages.c2e:demo-secret -H 'Content-Type: application/json' --data-binary '{
  "topic": "org.eclipse.packages.c2e/demo-device/things/twin/commands/modify",
  "headers": {},
  "path": "/features/temperature/properties/value",
  "value": 45
}' ${HTTP_ADAPTER_BASE_URL:?}/telemetry
```

In order to publish the telemetry data via MQTT, the `mosquitto_pub` command can be used:
```shell

mosquitto_pub -d -h ${MQTT_ADAPTER_IP} -p ${MQTT_ADAPTER_PORT_MQTTS} -u demo-device@org.eclipse.packages.c2e -P demo-secret --cafile /tmp/c2e_hono_truststore.pem --insecure -t telemetry -m '{
"topic": "org.eclipse.packages.c2e/demo-device/things/twin/commands/modify",
"headers": {},
"path": "/features/temperature/properties/value",
"value": 46
}'
```

## Retrieving the Digital Twin state
Here we can make use of the HTTP API and make sure that the new temperature value is actually there:
```shell

curl -u ditto:ditto -w '\n' ${DITTO_API_BASE_URL:?}/api/2/things/org.eclipse.packages.c2e:demo-device
```
The other way would be to go to the Ditto UI and check the ***Things*** tab.

## Pushing Telemetry Data (QoS 1)
QoS 1 is very important as some data is more important than other data!
That is why the both Hono and Ditto have concepts for this.
If we now send the data via MQTT with a QoS 1 on the **event** topic of Hono it will deliver the message to a different
Kafka topic. This will force Ditto to only commit the offset of the Kafka Consumer once the data is persisted!

```shell

mosquitto_pub -d -q 1 -h ${MQTT_ADAPTER_IP} -p ${MQTT_ADAPTER_PORT_MQTTS} -u demo-device@org.eclipse.packages.c2e -P demo-secret --cafile /tmp/c2e_hono_truststore.pem --insecure -t event -m '{
"topic": "org.eclipse.packages.c2e/demo-device/things/twin/commands/modify",
"headers": {},
"path": "/features/temperature/properties/value",
"value": 47
}'
```
And of course we can either check in the UI or do a HTTP call: 
```shell

curl -u ditto:ditto -w '\n' ${DITTO_API_BASE_URL:?}/api/2/things/org.eclipse.packages.c2e:demo-device
```
Same response but very different implications!!!

## Sending a Command to the Device via its Digital Twin

The command is sent to Ditto HTTP API, forwarded through the Kafka connection, consumed by Hono and published on a topic there.

1. First we subscribe the device for such messages inside Hono:
```shell

mosquitto_sub -v -h ${MQTT_ADAPTER_IP} -p ${MQTT_ADAPTER_PORT_MQTTS} -u demo-device@org.eclipse.packages.c2e -P demo-secret --cafile /tmp/c2e_hono_truststore.pem --insecure -t command///req/#
```
2. We send the command to the Ditto HTTP API by going to the UI and click on **Send Message** tab.
We use the subject `start-watering` a timeout of 0 and a simple JSON message `{"water-amount": "3liters"}`:
![Ditto UI Message](/Assets/ditto-ui-send-message.png)
And the respective response in the mosquitto_sub tab should read as follows:
```
command///req//start-watering {"topic":"org.eclipse.packages.c2e/demo-device/things/live/messages/start-watering","headers":{"version":2,"referer":"http://172.18.0.2:32319/ui/","x-ditto-pre-authenticated":"nginx:ditto","origin":"http://172.18.0.2:32319","dnt":"1","priority":"u=0","accept":"*/*","x-real-ip":"10.42.0.1","sec-gpc":"1","x-forwarded-user":"ditto","channel":"live","timeout":"0","response-required":false,"ditto-originator":"nginx:ditto","requested-acks":[],"ditto-message-direction":"TO","ditto-message-subject":"start-watering","ditto-message-thing-id":"org.eclipse.packages.c2e:demo-device","content-type":"application/json","timestamp":"2025-03-08T16:05:04.379337710+01:00","correlation-id":"a4c80869-6248-40ec-8aad-14c55377beff"},"path":"/inbox/messages/start-watering","value":{"water-amount":"3liters"}}
```
This confirms that our message arrived safely!

## Creating a HTTP Connection
Here we will make use of our MonsterMock(mmock) server and configure to connect to it.
The mmock server is already configured to reply with a `204 Accepted` status code on the `mmock/dummy` endpoint. 
Afterward we will send a MQTT message from the device and observe it going through Ditto and reaching the HTTP server.

In order to create the connection we will have to run the following commands:

1. Using the UI -> Go to the ***Connections*** tab, deselect the current connection and in the top right corner you should
see a **Create** button.
![Ditto UI Create Connection](/Assets/ditto-ui-create-connection.png)
2. Paste in the following JSON and click on `Create`
```json
{
    "id": "http-example-connection-mmock",
    "connectionType": "http-push",
    "connectionStatus": "open",
    "failoverEnabled": true,
    "uri": "http://mmock:80/",
    "specificConfig": {
      "parallelism": "2"
    },
    "sources": [],
    "targets": [
      {
        "address": "POST:/dummy",
        "topics": [
          "_/_/things/twin/events",
          "_/_/things/live/messages"
        ],
        "authorizationContext": ["pre-authenticated:hono-connection-org.eclipse.packages.c2e"],
        "headerMapping": {
          "content-type": "application/json"
        }
      }
    ]
}
```
3. You should see after a while the connection status as `succeeded`
![Ditto UI Connection Status Succeeded](/Assets/ditto-ui-connection-status-succeeded.png)
4. Before we do anything else we need to enable the logs of the Connection we just created.
Just scroll down and click the `Enable` button in the UI
![Ditto UI Enable Connection Logs](/Assets/ditto-ui-enable-connection-logs.png)
5. Now we are ready to push data from our device using the MQTT mosquitto_pub to do so.

```shell

mosquitto_pub -d -q 1 -h ${MQTT_ADAPTER_IP} -p ${MQTT_ADAPTER_PORT_MQTTS} -u demo-device@org.eclipse.packages.c2e -P demo-secret --cafile /tmp/c2e_hono_truststore.pem --insecure -t event -m '{
"topic": "org.eclipse.packages.c2e/demo-device/things/twin/commands/modify",
"headers": {},
"path": "/features/temperature/properties/value",
"value": 48
}'
```
6. Click the refresh button in the connection logs and you should see several message ending with a `published success`
![Ditto UI Connection Logs Publish Success](/Assets/ditto-ui-connection-logs-successful-send.png)

# Wrap up
Now we got a feel for how to work with Eclipse Hono and Ditto.
This was just a bare-bones demo and if you want more in depth scenario you can ping me :)

This was using Eclipse Packages repository which is currently not very well-supported -> which is why I highly recommend
doing separate deployments if you want to use Eclipse Hono and Ditto in production!!!



