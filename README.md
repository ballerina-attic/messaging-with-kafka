[![Build Status](https://travis-ci.org/ballerina-guides/messaging-with-kafka.svg?branch=master)](https://travis-ci.org/ballerina-guides/messaging-with-kafka)

# Messaging with Kafka

Kafka is a distributed, partitioned, replicated commit log service. It provides the functionality of a publish-subscribe messaging system with a unique design. Kafka mainly operates based on a topic model. A topic is a category or feed name to which records get published. Topics in Kafka are always multi-subscriber.

> This guide walks you through the process of messaging with Apache Kafka using Ballerina language. 

The following are the sections available in this guide.

- [What you'll build](#what-youll-build)
- [Prerequisites](#prerequisites)
- [Implementation](#implementation)
- [Testing](#testing)
- [Deployment](#deployment)
- [Observability](#observability)

## What you’ll build
To understanding how you can use Kafka for publish-subscribe messaging, let's consider a real-world use case of a product management system. This product management system consists of a product admin portal using which the product administrator can update the price for a product. 
This price update message should be consumed by a couple of franchisees and an inventory control system to take appropriate actions. Kafka is an ideal messaging system for this scenario.
In this particular use case, once the admin updates the price of a product, the update message is published to a Kafka topic called `product-price` to which the franchisees and the inventory control system subscribed to listen. The following diagram illustrates this use case.


![alt text](/images/messaging-with-kafka.svg)


In this example, the Ballerina Kafka Connector is used to connect Ballerina to Apache Kafka. With this Kafka Connector, Ballerina can act as both message publisher and subscriber.

## Prerequisites
- [Ballerina Distribution](https://ballerina.io/learn/getting-started/)
- [Apache Kafka 1.1.0](https://kafka.apache.org/downloads)
  * Download the binary distribution and extract the contents
- [Ballerina Kafka Connector](https://github.com/wso2-ballerina/package-kafka/releases/download/v0.96.0/ballerina-kafka-connector-0.96.0.zip)
  * After downloading the ZIP file, extract it and copy the containing JAR files into the <BALLERINA_HOME>/bre/lib folder
- A Text Editor or an IDE 

### Optional Requirements
- Ballerina IDE plugins ([IntelliJ IDEA](https://plugins.jetbrains.com/plugin/9520-ballerina), [VSCode](https://marketplace.visualstudio.com/items?itemName=WSO2.Ballerina), [Atom](https://atom.io/packages/language-ballerina))
- [Docker](https://docs.docker.com/engine/installation/)
- [Kubernetes](https://kubernetes.io/docs/setup/)

## Implementation

> If you want to skip the basics, you can download the git repo and directly move to the "Testing" section by skipping "Implementation" section.    

### Create the project structure

Ballerina is a complete programming language that supports custom project structures. Use the following package structure for this guide.
```
messaging-with-kafka
 └── guide
      ├── franchisee1
      │    └── franchisee1.bal
      ├── franchisee2
      │    └── franchisee2.bal
      ├── inventory_control_system
      │    └── inventory_control_system.bal
      └── product_admin_portal
           ├── product_admin_portal.bal
           └── tests
                └── product_admin_portal_test.bal

```

- Create the above directories in your local machine and also create empty `.bal` files.

- Then open the terminal and navigate to `messaging-with-kafka/guide` and run Ballerina project initializing toolkit.
```bash
   $ ballerina init
```

### Implementation

Let's get started with the implementation of a Kafka service, which is subscribed to the Kafka topic `product-price`. Let's consider the `inventory_control_system` for example.
First, let's see how to add the Kafka configurations for a Kafka subscriber written in Ballerina language. Refer to the code segment attached below.

##### Kafka subscriber configurations
```ballerina
// Kafka consumer endpoint
endpoint kafka:SimpleConsumer consumer {
    bootstrapServers: "localhost:9092, localhost:9093",
    // Consumer group ID
    groupId: "inventorySystemd",
    // Listen from topic 'product-price'
    topics: ["product-price"],
    // Poll every 1 second
    pollingInterval:1000
};
```

A Kafka subscriber in Ballerina needs to consist of a `kafka:SimpleConsumer` endpoint in which you specify the required configurations for a Kafka subscriber. 

The `bootstrapServers` field provides the list of host and port pairs, which are the addresses of the Kafka brokers in a "bootstrap" Kafka cluster. 

The `groupId` field specifies the Id of the consumer group. 

The `topics` field specifies the topics that must be listened by this consumer. 

The `pollingInterval` field is the time interval that a consumer polls the topic. 

Let's now see the complete implementation of the `inventory_control_system`, which is a Kafka topic subscriber.

##### inventory_control_system.bal
```ballerina
import ballerina/log;
import wso2/kafka;

// Kafka consumer endpoint
endpoint kafka:SimpleConsumer consumer {
    bootstrapServers: "localhost:9092, localhost:9093",
    // Consumer group ID
    groupId: "inventorySystemd",
    // Listen from topic 'product-price'
    topics: ["product-price"],
    // Poll every 1 second
    pollingInterval:1000
};

// Kafka service that listens from the topic 'product-price'
// 'inventoryControlService' subscribed to new product price updates from
// the product admin and updates the Database.
service<kafka:Consumer> kafkaService bind consumer {
    // Triggered whenever a message added to the subscribed topic
    onMessage(kafka:ConsumerAction consumerAction, kafka:ConsumerRecord[] records) {
        // Dispatched set of Kafka records to service, We process each one by one.
        foreach record in records {
            blob serializedMsg = record.value;
            // Convert the serialized message to string message
            string msg = serializedMsg.toString("UTF-8");
            log:printInfo("New message received from the product admin");
            // log the retrieved Kafka record
            log:printInfo("Topic: " + record.topic + "; Received Message: " + msg);
            // Mock logic
            // Update the database with the new price for the specified product
            log:printInfo("Database updated with the new price for the product");
        }
    }
}
```

In the above code, resource `onMessage` will be triggered whenever a message published to the topic specified.

To check the implementations of the other subscribers, see [franchisee1.bal](https://github.com/ballerina-guides/messaging-with-kafka/blob/master/guide/franchisee1/franchisee1.bal) and 
[franchisee2.bal](https://github.com/ballerina-guides/messaging-with-kafka/blob/master/guide/franchisee2/franchisee2.bal).


Let's next focus on the implementation of the `product_admin_portal`, which acts as the message publisher. It contains an HTTP service using which a product admin updates the price of a product. 

First, let's see how to add the Kafka configurations for a Kafka publisher written in Ballerina language. Refer to the code segment attached below.

##### Kafka producer configurations
```ballerina
// Kafka producer endpoint
endpoint kafka:SimpleProducer kafkaProducer {
    bootstrapServers: "localhost:9092",
    clientID:"basic-producer",
    acks:"all",
    noRetries:3
};
```

A Kafka producer in Ballerina needs to consist of a `kafka:SimpleProducer` endpoint in which you specify the required configurations for a Kafka publisher. 

Let's now see the complete implementation of the `product_admin_portal`, which is a Kafka topic publisher. Inline comments added for better understanding.

##### product_admin_portal.bal
```ballerina
import ballerina/http;
import wso2/kafka;

// Constants to store admin credentials
@final string ADMIN_USERNAME = "Admin";
@final string ADMIN_PASSWORD = "Admin";

// Kafka producer endpoint
endpoint kafka:SimpleProducer kafkaProducer {
    bootstrapServers: "localhost:9092",
    clientID:"basic-producer",
    acks:"all",
    noRetries:3
};

// HTTP service endpoint
endpoint http:Listener listener {
    port:9090
};

@http:ServiceConfig {basePath:"/product"}
service<http:Service> productAdminService bind listener {

    @http:ResourceConfig {methods:["POST"], consumes:["application/json"],
        produces:["application/json"]}
    updatePrice (endpoint client, http:Request request) {
        http:Response response;
        json reqPayload;
        float newPriceAmount;

        // Try parsing the JSON payload from the request
        match request.getJsonPayload() {
            // Valid JSON payload
            json payload => reqPayload = payload;
            // NOT a valid JSON payload
            any => {
                response.statusCode = 400;
                response.setJsonPayload({"Message":"Not a valid JSON payload"});
                _ = client -> respond(response);
                done;
            }
        }

        json username = reqPayload.Username;
        json password = reqPayload.Password;
        json productName = reqPayload.Product;
        json newPrice = reqPayload.Price;

        // If payload parsing fails, send a "Bad Request" message as the response
        if (username == null || password == null || productName == null ||
            newPrice == null) {
            response.statusCode = 400;
            response.setJsonPayload({"Message":"Bad Request - Invalid payload"});
            _ = client->respond(response);
            done;
        }

        // Convert the price value to float
        var result = <float>newPrice.toString();
        match result {
            float value => {
                newPriceAmount = value;
            }
            error err => {
                response.statusCode = 400;
                response.setJsonPayload({"Message":"Invalid amount specified"});
                _ = client->respond(response);
                done;
            }
        }

        // If the credentials does not match with the admin credentials,
        // send an "Access Forbidden" response message
        if (username.toString() != ADMIN_USERNAME ||
            password.toString() != ADMIN_PASSWORD) {
            response.statusCode = 403;
            response.setJsonPayload({"Message":"Access Forbidden"});
            _ = client->respond(response);
            done;
        }

        // Construct and serialize the message to be published to the Kafka topic
        json priceUpdateInfo = {"Product":productName, "UpdatedPrice":newPriceAmount};
        blob serializedMsg = priceUpdateInfo.toString().toBlob("UTF-8");

        // Produce the message and publish it to the Kafka topic
        kafkaProducer->send(serializedMsg, "product-price", partition = 0);
        // Send a success status to the admin request
        response.setJsonPayload({"Status":"Success"});
        _ = client->respond(response);
    }
}
```

## Testing 

### Invoking the service

- First, start the `ZooKeeper` instance with the default configurations by entering the following command in a terminal from `<KAFKA_HOME_DIRECTORY>`.
 ```bash
    $ bin/zookeeper-server-start.sh -daemon config/zookeeper.properties
 ```

- Start a single `Kafka broker` instance with the default configurations by entering the following command  in a terminal from `<KAFKA_HOME_DIRECTORY>`.
```bash
   $ bin/kafka-server-start.sh -daemon config/server.properties
```
  
  Here we started the Kafka server on `host:localhost` and `port:9092`. Now we have a working Kafka cluster.

- Create a new topic `product-price` on Kafka cluster by entering the following command in a terminal from `<KAFKA_HOME_DIRECTORY>`.
```bash
   $ bin/kafka-topics.sh --create --topic product-price --zookeeper \
   localhost:2181 --replication-factor 1 --partitions 2
```
   Here we created a new topic that consists of two partitions with a single replication factor.
   
- Start the `productAdminService`, which is an HTTP service that publishes messages to the Kafka topic by entering the following command from `messaging-with-kafka/guide` directory.
```bash
   $ ballerina run product_admin_portal
```
- Navigate to `messaging-with-kafka/guide` and run the following command in separate terminals to start the topic subscribers.
```bash
   $ ballerina run <subscriber_package_name>
```
   
- Invoke the `productAdminService` by sending a valid POST request.
```bash
   curl -v -X POST -d \
   '{"Username":"Admin", "Password":"Admin", "Product":"ABC", "Price":100.00}' \
   "http://localhost:9090/product/updatePrice" -H "Content-Type:application/json"
```

- The `productAdminService` sends a response similar to the following.
```bash
   < HTTP/1.1 200 OK
   {"Status":"Success"}
```

- Sample log messages in Kafka subscriber services:
```bash
  INFO  [<All>] - New message received from the product admin 
  INFO  [<All>] - Topic:product-price; Message:{"Product":"ABC","UpdatedPrice":100.0} 
  INFO  [franchisee1] - Acknowledgement from Franchisee 1 
  INFO  [franchisee2] - Acknowledgement from Franchisee 2 
  INFO  [inventory_control_system] - Database updated with the new price of the product
```

### Writing unit tests 

In Ballerina, the unit test cases should be in the same package inside a folder named as 'tests'.  When writing the test functions the below convention should be followed.
- Test functions should be annotated with `@test:Config`. See the below example.
```ballerina
   @test:Config
   function testProductAdminPortal() {
```
  
This guide contains unit test for the 'product_admin_portal' service implemented above. 

To run the unit tests, navigate to `messaging-with-kafka/guide` and run the following command. 
```bash
   $ ballerina test
```

When running the unit tests, make sure that `Kafka` is up and running.

To check the implementation of the test file, refer to the [product_admin_portal_test.bal](https://github.com/ballerina-guides/messaging-with-kafka/blob/master/guide/product_admin_portal/tests/product_admin_portal_test.bal).


## Deployment

Once you are done with the development, you can deploy the services using any of the methods that we listed below. 

### Deploying locally

As the first step, you can build Ballerina executable archives (.balx) of the services that we developed above. Navigate to `messaging-with-kafka/guide` and run the following command.
```bash
   $ ballerina build
```

- Once the .balx files are created inside the target folder, you can run them using the following command. 
```bash
   $ ballerina run target/<Exec_Archive_File_Name>
```

- The successful execution of a service will show us something similar to the following output.
```
   ballerina: initiating service(s) in 'target/product_admin_portal.balx'
   ballerina: started HTTP/WS endpoint 0.0.0.0:9090
```

### Deploying on Docker

You can run the service that we developed above as a docker container.
As Ballerina platform includes [Ballerina_Docker_Extension](https://github.com/ballerinax/docker), which offers native support for running ballerina programs on containers,
you just need to put the corresponding docker annotations on your service code.
Since this guide requires `Kafka` as a prerequisite, you need a couple of more steps to configure it in docker container.   

- Run the below command to start both `Zookeeper` and `Kafka` with default configurations in the same docker container. 
```bash
   $ docker run -p 2181:2181 -p 9092:9092 --env \
   ADVERTISED_HOST=`docker-machine ip \`docker-machine active\`` \
   --env ADVERTISED_PORT=9092 spotify/kafka
```

- Check whether the `Kafka` container is up and running using the following command.
```bash
   $ docker ps
```

Now let's see how we can deploy the `product_admin_portal` we developed above on docker. We need to import  `ballerinax/docker` and use the annotation `@docker:Config` as shown below to enable docker image generation during the build time. 

##### product_admin_portal.bal
```ballerina
import ballerinax/docker;
// Other imports

// Constants to store admin credentials

// Kafka producer endpoint

@docker:Config {
    registry:"ballerina.guides.io",
    name:"product_admin_portal",
    tag:"v1.0"
}

@docker:CopyFiles {
    files:[{source:<path_to_Kafka_connector_jars>,
            target:"/ballerina/runtime/bre/lib"}]
}

@docker:Expose{}
endpoint http:Listener listener {
    port:9090
};

@http:ServiceConfig {basePath:"/product"}
service<http:Service> productAdminService bind listener {
``` 

- `@docker:Config` annotation is used to provide the basic docker image configurations for the sample. `@docker:CopyFiles` is used to copy the Kafka connector jar files into the ballerina bre/lib folder. You can provide multiple files as an array to field `files` of CopyFiles docker annotation. `@docker:Expose {}` is used to expose the port. 

- Now you can build a Ballerina executable archive (.balx) of the service that we developed above, using the following command. This will also create the corresponding docker image using the docker annotations that you have configured above. Navigate to `messaging-with-kafka/guide` and run the following command.  
  
```
   $ballerina build product_admin_portal
  
   Run following command to start docker container: 
   docker run -d -p 9090:9090 ballerina.guides.io/product_admin_portal:v1.0
```

- Once you successfully build the docker image, you can run it with the `` docker run`` command that is shown in the previous step.  

```bash
   $ docker run -d -p 9090:9090 ballerina.guides.io/product_admin_portal:v1.0
```

   Here we run the docker image with flag`` -p <host_port>:<container_port>`` so that we use the host port 9090 and the container port 9090. Therefore you can access the service through the host port. 

- Verify docker container is running with the use of `` $ docker ps``. The status of the docker container should be shown as 'Up'. 

- You can access the service using the same curl commands that we've used above.
```bash
   curl -v -X POST -d \
   '{"Username":"Admin", "Password":"Admin", "Product":"ABC", "Price":100.00}' \
   "http://localhost:9090/product/updatePrice" -H "Content-Type:application/json"
```


### Deploying on Kubernetes

- You can run the service that we developed above, on Kubernetes. The Ballerina language offers native support for running a ballerina programs on Kubernetes, with the use of Kubernetes annotations that you can include as part of your service code. Also, it will take care of the creation of the docker images. So you don't need to explicitly create docker images prior to deploying it on Kubernetes. Refer to [Ballerina_Kubernetes_Extension](https://github.com/ballerinax/kubernetes) for more details and samples on Kubernetes deployment with Ballerina. You can also find details on using Minikube to deploy Ballerina programs. 

- Since this guide requires `Kafka` as a prerequisite, you need an additional step to create a pod for `Kafka` and use it with our sample.  

- Navigate to `messaging-with-kafka/resources` directory and run the below command to create the Kafka pod by creating a deployment and service for Kafka. You can find the deployment descriptor and service descriptor in the `./resources/kubernetes` folder.
```bash
   $ kubectl create -f ./kubernetes/
```

- Now let's see how we can deploy the `product_admin_portal` on Kubernetes. We need to import `` ballerinax/kubernetes; `` and use `` @kubernetes `` annotations as shown below to enable kubernetes deployment.

##### product_admin_portal.bal

```ballerina
import ballerinax/kubernetes;
// Other imports

// Constants to store admin credentials

// Kafka producer endpoint

@kubernetes:Ingress {
  hostname:"ballerina.guides.io",
  name:"ballerina-guides-product-admin-portal",
  path:"/"
}

@kubernetes:Service {
  serviceType:"NodePort",
  name:"ballerina-guides-product-admin-portal"
}

@kubernetes:Deployment {
  image:"ballerina.guides.io/product_admin_portal:v1.0",
  name:"ballerina-guides-product-admin-portal",
  copyFiles:[{target:"/ballerina/runtime/bre/lib",
                  source:<path_to_Kafka_connector_jars>}]
}

endpoint http:Listener listener {
    port:9090
};

@http:ServiceConfig {basePath:"/product"}
service<http:Service> productAdminService bind listener {
``` 

- Here we have used ``  @kubernetes:Deployment `` to specify the docker image name which will be created as part of building this service. `copyFiles` field is used to copy the Kafka connector jar files into the ballerina bre/lib folder. You can provide multiple files as an array to this field.
- We have also specified `` @kubernetes:Service `` so that it will create a Kubernetes service, which will expose the Ballerina service that is running on a Pod.  
- In addition we have used `` @kubernetes:Ingress ``, which is the external interface to access your service (with path `` /`` and host name ``ballerina.guides.io``)

- Now you can build a Ballerina executable archive (.balx) of the service that we developed above, using the following command. This will also create the corresponding docker image and the Kubernetes artifacts using the Kubernetes annotations that you have configured above.
  
```
   $ ballerina build product_admin_portal
  
   Run following command to deploy kubernetes artifacts:  
   kubectl apply -f ./target/product_admin_portal/kubernetes
```

- You can verify that the docker image that we specified in `` @kubernetes:Deployment `` is created, by using `` docker images ``. 
- Also the Kubernetes artifacts related our service, will be generated under `` ./target/product_admin_portal/kubernetes``. 
- Now you can create the Kubernetes deployment using:

```bash
   $ kubectl apply -f ./target/product_admin_portal/kubernetes 
 
   deployment.extensions "ballerina-guides-product-admin-portal" created
   ingress.extensions "ballerina-guides-product-admin-portal" created
   service "ballerina-guides-product-admin-portal" created
```

- You can verify Kubernetes deployment, service and ingress are running properly, by using following Kubernetes commands. 

```bash
   $ kubectl get service
   $ kubectl get deploy
   $ kubectl get pods
   $ kubectl get ingress
```

- If everything is successfully deployed, you can invoke the service either via Node port or ingress. 

Node Port:
```bash
  curl -v -X POST -d \
  '{"Username":"Admin", "Password":"Admin", "Product":"ABC", "Price":100.00}' \
  "http://localhost:<Node_Port>/product/updatePrice" -H "Content-Type:application/json"
```

Ingress:

Add `/etc/hosts` entry to match hostname. 
``` 
   127.0.0.1 ballerina.guides.io
```

Access the service 
```bash
   curl -v -X POST -d \
   '{"Username":"Admin", "Password":"Admin", "Product":"ABC", "Price":100.00}' \
   "http://ballerina.guides.io/product/updatePrice" -H "Content-Type:application/json" 
```


## Observability 
Ballerina is by default observable. Meaning you can easily observe your services, resources, etc.
However, observability is disabled by default via configuration. Observability can be enabled by adding following configurations to `ballerina.conf` file in `messaging-with-kafka/guide/`.

```ballerina
[b7a.observability]

[b7a.observability.metrics]
# Flag to enable Metrics
enabled=true

[b7a.observability.tracing]
# Flag to enable Tracing
enabled=true
```

NOTE: The above configuration is the minimum configuration needed to enable tracing and metrics. With these configurations default values are load as the other configuration parameters of metrics and tracing.

### Tracing 

You can monitor ballerina services using in built tracing capabilities of Ballerina. We'll use [Jaeger](https://github.com/jaegertracing/jaeger) as the distributed tracing system.
Follow the following steps to use tracing with Ballerina.

- You can add the following configurations for tracing. Note that these configurations are optional if you already have the basic configuration in `ballerina.conf` as described above.
```
   [b7a.observability]

   [b7a.observability.tracing]
   enabled=true
   name="jaeger"

   [b7a.observability.tracing.jaeger]
   reporter.hostname="localhost"
   reporter.port=5775
   sampler.param=1.0
   sampler.type="const"
   reporter.flush.interval.ms=2000
   reporter.log.spans=true
   reporter.max.buffer.spans=1000
```

- Run Jaeger docker image using the following command
```bash
   $ docker run -d -p5775:5775/udp -p6831:6831/udp -p6832:6832/udp -p5778:5778 \
   -p16686:16686 p14268:14268 jaegertracing/all-in-one:latest
```

- Navigate to `messaging-with-kafka/guide` and run the `product_admin_portal` using following command 
```
   $ ballerina run product_admin_portal/
```

- Observe the tracing using Jaeger UI using following URL
```
   http://localhost:16686
```

### Metrics
Metrics and alerts are built-in with ballerina. We will use Prometheus as the monitoring tool.
Follow the below steps to set up Prometheus and view metrics for product_admin_portal service.

- You can add the following configurations for metrics. Note that these configurations are optional if you already have the basic configuration in `ballerina.conf` as described under `Observability` section.

```ballerina
   [b7a.observability.metrics]
   enabled=true
   provider="micrometer"

   [b7a.observability.metrics.micrometer]
   registry.name="prometheus"

   [b7a.observability.metrics.prometheus]
   port=9700
   hostname="0.0.0.0"
   descriptions=false
   step="PT1M"
```

- Create a file `prometheus.yml` inside `/tmp/` location. Add the below configurations to the `prometheus.yml` file.
```
   global:
     scrape_interval:     15s
     evaluation_interval: 15s

   scrape_configs:
     - job_name: prometheus
       static_configs:
         - targets: ['172.17.0.1:9797']
```

   NOTE : Replace `172.17.0.1` if your local docker IP differs from `172.17.0.1`
   
- Run the Prometheus docker image using the following command
```
   $ docker run -p 19090:9090 -v /tmp/prometheus.yml:/etc/prometheus/prometheus.yml \
   prom/prometheus
```
   
- You can access Prometheus at the following URL
```
   http://localhost:19090/
```

NOTE:  Ballerina will by default have following metrics for HTTP server connector. You can enter following expression in Prometheus UI
-  http_requests_total
-  http_response_time


### Logging

Ballerina has a log package for logging to the console. You can import ballerina/log package and start logging. The following section will describe how to search, analyze, and visualize logs in real time using Elastic Stack.

- Start the Ballerina Service with the following command from `messaging-with-kafka/guide`
```
   $ nohup ballerina run product_admin_portal/ &>> ballerina.log&
```
   NOTE: This will write the console log to the `ballerina.log` file in the `messaging-with-kafka/guide` directory

- Start Elasticsearch using the following command

- Start Elasticsearch using the following command
```
   $ docker run -p 9200:9200 -p 9300:9300 -it -h elasticsearch --name \
   elasticsearch docker.elastic.co/elasticsearch/elasticsearch:6.2.2 
```

   NOTE: Linux users might need to run `sudo sysctl -w vm.max_map_count=262144` to increase `vm.max_map_count` 
   
- Start Kibana plugin for data visualization with Elasticsearch
```
   $ docker run -p 5601:5601 -h kibana --name kibana --link \
   elasticsearch:elasticsearch docker.elastic.co/kibana/kibana:6.2.2     
```

- Configure logstash to format the ballerina logs

i) Create a file named `logstash.conf` with the following content
```
input {  
 beats{ 
     port => 5044 
 }  
}

filter {  
 grok{  
     match => { 
	 "message" => "%{TIMESTAMP_ISO8601:date}%{SPACE}%{WORD:logLevel}%{SPACE}
	 \[%{GREEDYDATA:package}\]%{SPACE}\-%{SPACE}%{GREEDYDATA:logMessage}"
     }  
 }  
}   

output {  
 elasticsearch{  
     hosts => "elasticsearch:9200"  
     index => "store"  
     document_type => "store_logs"  
 }  
}  
```

ii) Save the above `logstash.conf` inside a directory named as `{SAMPLE_ROOT}\pipeline`
     
iii) Start the logstash container, replace the {SAMPLE_ROOT} with your directory name
     
```
$ docker run -h logstash --name logstash --link elasticsearch:elasticsearch \
-it --rm -v ~/{SAMPLE_ROOT}/pipeline:/usr/share/logstash/pipeline/ \
-p 5044:5044 docker.elastic.co/logstash/logstash:6.2.2
```
  
 - Configure filebeat to ship the ballerina logs
    
i) Create a file named `filebeat.yml` with the following content
```
filebeat.prospectors:
- type: log
  paths:
    - /usr/share/filebeat/ballerina.log
output.logstash:
  hosts: ["logstash:5044"]  
```
NOTE : Modify the ownership of filebeat.yml file using `$chmod go-w filebeat.yml` 

ii) Save the above `filebeat.yml` inside a directory named as `{SAMPLE_ROOT}\filebeat`   
        
iii) Start the logstash container, replace the {SAMPLE_ROOT} with your directory name
     
```
$ docker run -v {SAMPLE_ROOT}/filbeat/filebeat.yml:/usr/share/filebeat/filebeat.yml \
-v {SAMPLE_ROOT}/guide/product_admin_portal/ballerina.log:/usr/share\
/filebeat/ballerina.log --link logstash:logstash docker.elastic.co/beats/filebeat:6.2.2
```
 
 - Access Kibana to visualize the logs using following URL
```
   http://localhost:5601 
```
  
