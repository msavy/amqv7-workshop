## High Availability of Brokers in AMQ 7 Broker (HA)    

This worksheet covers HA AMQ7 Brokers. 
By the end of this you should know:

1. Shared Store HA concepts of AMQ7
   * Live/Backup pairs
   * Shared Store HA
   * Non Shared Store HA (replication)
   
2. How to configure HA
   * Configuring Shared Store HA
   * Configuring Clients
   * Configuring Replication 


### AMQ 7 HA Concepts

We define high availability as the *ability for the system to continue
functioning after failure of one or more of the servers*.

A part of high availability is *failover* which we define as the
*ability for client connections to migrate from one server to another in
event of server failure so client applications can continue to operate*.

The AMQ 7 Broker does this by using live/backup pairs where a backup is a
passive broker that will recover the state of a live broker if it crashes or
fails for some reason and take over its duties. There are 2 types of HA with AMQ 7
shared store and replication.

This demo will firstly cover shared store HA to teach the basic concepts of how HA works 
and how clients interact with HA brokers and then a deeper dive into replicated HA.

#### Shared Store HA

Lets start by creating a live/backup pair of brokers using the CLI, to create the live run:

      $ARTEMIS_HOME/bin/artemis create --allow-anonymous --user admin --password password --cluster-user admin --cluster-password --clustered --shared-store --host localhost --data ../shared-store live1
      
and to create the backup run:

      $ARTEMIS_HOME/bin/artemis create --allow-anonymous --user admin --password password --cluster-user admin --cluster-password --clustered --shared-store --host localhost --data ../shared-store --slave --port-offset 100 backup1

The important thing to note here is that the data directory points to the same location, shared store works by sharing
a file lock that the current live broker holds. Once this is released by a server crash then the backup gains the lock 
and loads the journal and becomes live. 

Now start both brokers, you should see the live start and the backup stay passive. You should see something like the following
in the backups log:

```bash
11:29:51,227 INFO  [org.apache.activemq.artemis.core.server] AMQ221031: backup announced
```

This means that backup has connected to the cluster (of 1) and broadcast its topology, the live brokers then send
this info to clients. You can also see the backup in the hawtIO console, make sure you enable slave brokers in the 
view drop down on the right hand side of the diagram tab, it should look like:

![alt text](etc/backup.png "Broker Diagram")

Now lets look at the configurations of the brokers, the first thing you will notice is that both the configurations
both have clustered configs, broadcast-groups, discovery-groups and cluster-connections. Even though this is not specifically
a clustered topology the same mechanics are used in sharing the topology with the master node, it also simplifies the
configuration if multiple live/back pairs are deployed in a cluster.

The extra configuration needed by both live and backup is an HA policy, Brokers are given a specific role of either a 
live or backup rather than deciding this at startup time like AMQ 6 does, that is first broker up is live.

The live broker is configured with a shared store policy with the role of master:

```xml
<ha-policy>
  <shared-store>
    <master>
       <failover-on-shutdown>false</failover-on-shutdown>
    </master>
  </shared-store>
</ha-policy>
```
      
The backup broker is also configured with a shared store policy but with the role of slave:

```xml
<ha-policy>
   <shared-store>
      <slave>
         <failover-on-shutdown>false</failover-on-shutdown>
      </slave>
   </shared-store>
</ha-policy>
```
Both brokers have 'failover-on-shutdown' to false, thi smeans that if the live broker is stopped cleany either by
ctrl-c or the CLI then the backup will stay passive. Setting this to true will allow failover ona clean shutdown.

Try playing around with stopping or killing the live broker to see the backup take over.
 
Also notice that if you restart a live broker, failback will occur, that is the backup will fall back to passive mode
and allow the live ti restart. This can be configured on the backup bu setting the 'allow-failback' config, like so:
 
 ```xml
<ha-policy>
   <shared-store>
      <slave>
         <failover-on-shutdown>true</failover-on-shutdown>
         <allow-failback>false</allow-failback>
      </slave>
   </shared-store>
</ha-policy>
```

#### Clients with HA

Before we start lets configure the live and backup with the same example queue, like so:

```xml
<address name="exampleQueue">
   <anycast>
      <queue name="exampleQueue"/>
   </anycast>
</address>
```

Client Support for HA is only available in the core JMS client and the qpid JMS client. The core JMS client has HA built 
in to the protocol where as qpid JMS leverages what is available in the AMQP protocol so it's feature set is limited. We 
 will concentrate on the core JMS client mostly.
 
##### Core JMS Client

HA is configured on the connection factory, typically by JNDI in the jndi.properties file:

```bash
java.naming.factory.initial=org.apache.activemq.artemis.jndi.ActiveMQInitialContextFactory
connectionFactory.ConnectionFactory=tcp://localhost:61616?ha=true&retryInterval=1000&retryIntervalMultiplier=1.0&reconnectAttempts=-1
queue.queue/exampleQueue=exampleQueue
```

The important configuration is ha=true, this tells the connection that when disconnected it should retry with the live and its backup.
The rest of the parameters specify how long and at what interval it should keep retrying.

Now lets run an example sender and receiver by running the following commands:

``bash
mvn verify -PHAMessageReceiver
``

and

```bash
mvn verify -PHAMessageSender
```

The sender will send 10 messages every 1 second within a local transaction, the receiver will consume 10 messages again in a transaction.

Now kill the live broker and see what happens, you should see the clients log a warning, pause for a while
and then continue. something like

```bash
WARN: AMQ212037: Connection failure has been detected: AMQ119015: The connection was disconnected because of server shutdown [code=DISCONNECTED]
```

The failover of the client is completely transparent, however from a JMS API point of view some errors need handling, 
you may have seen the following error in the sending client:

```bash
tx was rolled back after fail over
```

This error handling has been built into the Client, if you take a look at the 'com.redhat.workshop.amq7.ha.Sender' class
you will see the following:

```java
try {
   session.commit();
} catch (TransactionRolledBackException e) {
   System.out.println("tx was rolled back after fail over");
}
```

This happens because the client failed over during the lifecycle of the transaction, so somewhere between the 1st
message being sent and the transaction being committed. This is because the transaction may have contained non persistent
messages and because the messages are sent async so guarantees can't be held.

Another exception you might see is a JMS Exception with the error code of 'ActiveMQExceptionType.UNBLOCKED'. This happens 
when failover occurs in the middle of a blocking call, such as a commit for instance or sending a persistent message.
Since there is no way of knowing if the call made it to the broker the application needs to decide what to do in this case.


#### qpid JMS client

The qpid JMS client can also be configured to support HA on the connection factory via JNDI:

```bash
java.naming.factory.initial = org.apache.qpid.jms.jndi.JmsInitialContextFactory
connectionfactory.ConnectionFactory = failover:(amqp://localhost:61616)
queue.queue/exampleQueue=exampleQueue
```

The qpid JMS client will transparently reconnect to the backup altho unlike the core JMS client you won't see
any warnings. Similarly the qpid JMS client will always rollback transactions that exist when failover occured.
 
One major difference is that qpid JMS will not throw an exception on blocking sends, instead it will try to resend.
So for instance when sending a persistent message the client will try to resend it if failover occurs which could
cause duplicates. The same also applies for consumed messages depending on the acknowledgement mode.

make sure the previous core JMS receiver client is running and run the qpid JMS Sender using the command:

```bash
mvn verify -PHAQpidMessageSender
```

Again try killing and restarting the brokers to see what happens.