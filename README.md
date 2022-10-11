# kafka-docker-environment
Help with setting up a docker environment for local testing and learning.

The goal here is to create an kafka environment quickly with very little install (except Docker) to be able to test out your code and other software that interacts with kafka.

## Kafka On Docker Setup

Make sure you have the ability to run docker containers locally.

Save this docker-compose.yml to a file named **docker-compose.yml**:
```
version: "3"
services:
  zookeeper:
    image: 'bitnami/zookeeper:latest'
    container_name: zookeeper
    networks:
      - kafka-nw
    ports:
      - '2181:2181'
    environment:
      - ALLOW_ANONYMOUS_LOGIN=yes
  kafka:
    image: 'bitnami/kafka:latest'
    container_name: kafka
    networks:
      - kafka-nw
    ports:
      - '9092:9092'
    environment:
      - KAFKA_BROKER_ID=1
      - KAFKA_CFG_LISTENERS=PLAINTEXT://:9092
      - KAFKA_CFG_ADVERTISED_LISTENERS=PLAINTEXT://kafka:9092
      - KAFKA_CFG_ZOOKEEPER_CONNECT=zookeeper:2181
      - ALLOW_PLAINTEXT_LISTENER=yes
    depends_on:
      - zookeeper

networks:
  kafka-nw:
    name: kafka-nw
    driver: bridge
```

You can run the docker compose above with this command:

` docker-compose up -d`

This docker-compose will create a kafka broker and zookeeper containers with no topics.
The kafka service will start as a container named: "kafka" and everything will be on the "kafka-nw" network.

Once this is done, you will have kafka and zookeeper running in Docker and next will need to create a topic.

## Creating a Topic

To create a topic we are going to execute a command on the kafka broker to create a topic.  The command below creates the topic named **input-topic**, but you can change the name as needed.

```
docker exec -it -e KAFKA_CFG_ZOOKEEPER_CONNECT=zookeeper:2181 kafka kafka-topics.sh --create --topic input-topic  --bootstrap-server localhost:9092
```
This command will create the topic.

You can run the following command at anytime to list the topics on the broker:
```
docker exec -it -e KAFKA_CFG_ZOOKEEPER_CONNECT=zookeeper:2181 kafka kafka-topics.sh --list  --bootstrap-server localhost:9092
```

## Producing Messages

We are going to use a utility called [kcat](https://github.com/edenhill/kcat) to interact with Kafk for simple producing and consuming of messages.
We are going to run kcat in a docker container as well.  Here's the command to produce messages:
```
docker run -it --network=kafka-nw edenhill/kcat:1.7.1 -b kafka:9092 -K : -P -t input-topic
```
This command assumes you will run it and then it will wait for you to type a KEY followed by a ':' and then the message you want to send.
You can hit Enter between each message and then hit "CTRL+D" to send the messages.
Example:
```
C:\devl\kafka\DockerKafka>docker run -it --network=kafka-nw edenhill/kcat:1.7.1 -b kafka:9092 -K , -P -t input-topic
1:hello
2:goodbye
```

Producing messages using a file:
```
docker run -it -v c:/tmp/messages:/tmp --network=kafka-nw edenhill/kcat:1.7.1 -b kafka:9092 -P -t barry-topic /tmp/mymessage.txt
```
This command uses the **-v** option to mount your local c:\tmp\messages folder to the containers /tmp folder.  This would assume the "mymessage.txt" is a file in that folder and therefore would be accessible in the container at /tmp/mymessage.txt.  Of course you can change the paths and file names as you see fit.
**NOTE:** Do NOT try to use the **-v** option in windows while using gitbash or some other Bash emulator.  It confuses the paths and causes issues.  If using Windows, it's recommended to do volume mounts with docker commands in the command prompt or powershell.

## Consuming messages
 
 We will also use kcat to consume messages:
 ```
 docker run -it --network=kafka-nw edenhill/kcat:1.7.1 -b kafka:9092 -C -t input-topic
 ```
 This command will read messages from the topic named "input-topic" and then output the message to the standard out.
Example:
```
 C:\devl\kafka\DockerKafka>docker run -it --network=kafka-nw edenhill/kcat:1.7.1 -b kafka:9092 -C -t input-topic
1:hello
2:goodbye
% Reached end of topic input-topic [0] at offset 2

 ```
Hit "CTRL+C" to break out of the consumer.

## Update Hosts file for local consumers/producers

In order to allow consumers or producers running on your local machine and not in Docker to access the broker, you need to update your host file with the following entry.
Hosts file can be found at this location: C:\Windows\System32\drivers\etc  NOTE: you will need admin rights(elevated privledges) on your machine to edit the hosts file.
```
#For kafka in docker
127.0.0.1 kafka
```


## Useful Commands

```
# Get offsets for a topic
docker exec -it -e KAFKA_CFG_ZOOKEEPER_CONNECT=zookeeper:2181 kafka kafka-get-offsets.sh --topic input-topic  --bootstrap-server localhost:9092

#List current consumer groups
docker exec -it -e KAFKA_CFG_ZOOKEEPER_CONNECT=zookeeper:2181 kafka kafka-consumer-groups.sh --list --bootstrap-server localhost:9092

# Get details about specific consumer group and their offsets
docker exec -it -e KAFKA_CFG_ZOOKEEPER_CONNECT=zookeeper:2181 kafka kafka-consumer-groups.sh --describe --group barry-demo  --bootstrap-server localhost:9092
```

