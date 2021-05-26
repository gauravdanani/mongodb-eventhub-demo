# mongodb-eventhub-demo
# Overview
This POC demonstrates how to set up the MongoDB Connector for Apache Kafka and use it within the Azure Event Hub environment. In the end, we will write data to a MongoDB collection, the source connector will see the change and push the data into an Azure Event hub. From there the sink connector will notice the new arrival of data in the event hub and push this data into another MongoDB collection.
# Prerequisites
**Docker** - If you don’t have Docker installed check out the [Docker website](https://docs.docker.com/get-docker/).

**MongoDB Atlas cluster** - In this demo, we will be using a MongoDB Cluster as a source and sink to the Azure Event Hub. You will need to add DB user and whitelist Azure Event Hub IP or put 0.0.0.0 to allow access from any IP.

![image](images/DbUser.png?raw=true "DBUser")

![image](images/NetworkAccess.png?raw=true "DBUser")

**Azure CLI tool** - Install latest version of [Azure CLI](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli?view=azure-cli-latest).

# Demo Setup
## Creating the Azure Resources
I have already created a test EventHub namespace [wow-dev-lm-hub-gauravtest](https://portal.azure.com/#@woolworthsgroup.com.au/resource/subscriptions/f5261f82-adb5-4a60-8bde-ca6690ba8737/resourceGroups/wow-dev-lastmile-aae/providers/Microsoft.EventHub/namespaces/wow-dev-lm-hub-gauravtest/overview) under wow-dev-lastmile-aae resource group for this POC. But, in case if you want to setup new EventHub topic then follow below steps to create one.

Login using Z-account credentials,
```
az login
```
Set to WOWDEVTEST subscription,
```
az account set --subscription WOWDEVTEST
```
Create EventHub namespace,
```
az eventhubs namespace create --name <EVENT HUB NAMESPACE NAME> --resource-group <RESOURCE GROUP> -l australiaeast
```
Create EventHub topic,
```
az eventhubs eventhub create --name <TOPIC NAME> --resource-group <RESOURCE GROUP> --namespace-name <EVENT HUB NAMESPACE NAME>
```

**NOTE**: Copy connection string to EventHub, this can be found under Shared Access Policies.

## Additional step if you have created your own Event Hub
Update Event Hub server and connection details in src/.env to match your configuration.

## Using Docker to spin up Kafka Connect container
Run *docker-compose up* in the *src* directory from terminal. This will launch an instance of Kafka Connect and connect to the Azure Event Hub. If you run into any issues, double-check you pasted the right credentials in the .env file.

## Configuring the MongoDB Connector for Apache Kafka
Once the Kafka Connector is running we can make our REST API calls to define the source and sink. Note you can view the Kafka Connect logs when it's running in a Docker container by issuing *docker logs {container id}* where container id can be obtained from docker ps command.

### Defining the source
Log in to your MongoDB Atlas cluster and obtain the connection string. Replace it in the connection.uri and issue the following command:
```
curl -X POST -H "Content-Type: application/json" --data '
  {"name": "mongo-source-demodata",
   "config": {
     "tasks.max":"1",
     "connector.class":"com.mongodb.kafka.connect.MongoSourceConnector",
     "key.converter":"org.apache.kafka.connect.json.JsonConverter",
     "key.converter.schemas.enable":true,
     "value.converter":"org.apache.kafka.connect.json.JsonConverter",
     "value.converter.schemas.enable":true,
     "publish.full.document.only": true,
     "connection.uri":"mongodb+srv://username-password@cluster0.mongodb.net/dbname?retryWrites=true&w=majority",
     "topic.prefix":"demo",
     "database":"EventHubDemo",
     "collection":"Source"
}}' http://localhost:8083/connectors -w "\n"
```
The command above will tell the Kafka Connector that we want to create a source to the Source collection in the EventHubDemo database. The topic will be created for us and we specified a topic.prefix of “demo” thus our full topic (i.e. Azure Event Hub name) will be “demo.EventHubDemo.Source”.

### Defining the sink
Replace connection string in the connection.uri and issue the following command:
```
curl -X POST -H "Content-Type: application/json" --data '
  {"name": "mongo-atlas-sink",
   "config": {
     "connector.class":"com.mongodb.kafka.connect.MongoSinkConnector",
     "tasks.max":"1",
     "topics":"demo.EventHubDemo.Source",
     "connection.uri":"mongodb+srv://username-password@cluster0.mongodb.net/dbname?retryWrites=true&w=majority",
     "database":"EventHubDemo",
     "collection":"Sink",
     "key.converter":"org.apache.kafka.connect.json.JsonConverter",
     "key.converter.schemas.enable":false,
     "value.converter":"org.apache.kafka.connect.json.JsonConverter",
     "value.converter.schemas.enable":false
}}' http://localhost:8083/connectors -w "\n"
```
In the above configuration, the MongoDB Connector will write data from the topic, “demo.EventHubDemo.Source” into the “Sink” collection in the “EventHubDemo” database.

## Running the Demo
Run Kafka Connect container and manually insert documents to the source collection. Sink connector should copy this new document to Sink collection. 
