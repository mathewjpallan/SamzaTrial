# Trying Samza

Samza 0.14 trial (All the below steps have been tried on Ubuntu 16.04 LTS)

## Pre-requisites to get the stack installed (ahead of trying the source in this repo)

1. Install jdk 8
2. Download and unzip Zookeeper 3.4.10. The default Zookeeper package comes with a sample cfg file in the conf folder. Rename the file to zoo.cfg and run bin/zkServer.sh start. This should start zookeeper on the default port 2181. Kafka requires zookeeper to store configuration.
3. Download and unzip Kafka 0.11 and run bin/kafka-server-start.sh config/server.properties. The server.properties has the port for zookeeper and you need to re-confirm that it points to the zookeeper port.
4. Download and unzip hadoop 2.61 and execute the following to start the YARN resource manager and node manager
   sbin/yarn-daemon.sh start resourcemanager
   sbin/yarn-daemon.sh start nodemanager
5. Clone the samza repo (git clone git://git.apache.org/samza.git) and run ./gradlew -PscalaVersion=2.11 clean publishToMavenLocal from the clone folder 


## Notes on Samza

Samza concepts are well explained in the Samza documentation, but here is a quick summary for the impatient!
- Any job in Samza has to implement the StreamTask interface and have a property file with the job configuration.
- Zamza uses kafka as the stream by default - a stream is a topic in Kafka that can have partitions. Partitions help with parallelism.
- A job will run in a container (by default) and there will be as many StreamTask instances as partitions. Each streamtask instance will only read from 1 partition by default.
- Each container is a single threaded event loop and hence if you need more throughput, you should increase the number of containers. We can have as many containers as streamtasks (which is equal to the number of partitions in the input topic) and each streamtask instance will run in one of the containers which samza will schedule.
- Each SamzaContainer typically runs as an indepentent Java virtual machine.
- The sample job that is included in this trial has a stream job that just outputs the message to an output topic, but you can enhance the sample to include your processing.

## How to run the sample
**The pre-requisites have to be complete ahead of running the sample**

- Start Zookeeper, Kafka, YARN
- Clone this repo git clone https://github.com/mathewjpallan/SamzaTrial.git
- The event-validator.properties contains the input stream (tasks.input) and is set to events-in. Also the EventValidatorTask outputs the event to the events-valid stream. Create kafka topics events-in and events-valid
        bin/kafka-topics.sh --create --zookeeper localhost:2181 --replication-factor 1 --partitions 6 --topic events-in
        bin/kafka-topics.sh --create --zookeeper localhost:2181 --replication-factor 1 --partitions 6 --topic events-valid
- Use kafka-console-producer to input messages into the events-in topic 
        bin/kafka-console-producer.sh --broker-list localhost:9092 --topic events-in
        You can also use the kafka-producer-perf-test.sh to input millions of message for any testing very quickly. Eg - bin/kafka-producer-perf-test.sh --topic events-in --num-records 50000000 --record-size 100 --throughput 50000000 --producer-props bootstrap.servers=localhost:9092
- Navigate to the cloned source base and execute ./exec.sh. This will execute a maven build, generate the artefact and submit the job to YARN. 
- The exec.sh generates one artefact which contains a single job. This job is submitted to YARN and it spawns a single container that contains 6 (as we created topic events-in with 6 partitions) StreamTask instances. You can change the number of containers for more throughput (Samza will distribute the streamtask instances across containers) by updating the yarn.container.count variable in the event-validator.properties
- The job should show up (after a minute or so) on the YARN portal in RUNNING state at http://localhost:8088/cluster. The final state will be UNDEFINED and that is fine.
- Use the console consumer to see if the events from events-in topic are being streamed to the events-valid topic
        bin/kafka-console-consumer.sh  --zookeeper localhost:21000 --topic events-valid --from-beginning
- To stop the job, execute ./exec.sh stop
- You can now change the input and output topics and also include processing that you want to include. The input topic needs to be updated in the event-validator.properties and the output/processing customization goes into the EventValidatorTask.

## Performance test summary
On a workstation (4 cores, 16 GB RAM, with SSD running Ubuntu 16) this job was giving a throughput of a 100K messages processed in a second with a single container utilizing around 50% of 4 cores. By increasing the containers to 3, it was giving a throughput of 150K messages a second and CPU was maxing out which means that it would have given better throughput if there was spare CPU.