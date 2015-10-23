# wildfly-8.2-ei
#How to Use Out of Process ActiveMQ with WildFly
This document explains how to use an out of process ActiveMQ to send and receive JMS messages inside WildFly Application Server.

##Start standalone ActiveMQ
This documentation assumes that ActiveMQ is started as a standalone application and listens on localhost:61616 to send and receive JMS messages.

## Setup WildFly module for ActiveMQ RA
ActiveMQ Resource Adaptater (RA) must be included and configured in WildFly application server
- Download ActiveMQ resource adapter from http://repo1.maven.org/maven2/org/apache/activemq/activemq-rar/5.11.1/activemq-rar-5.11.1.rar
- Create a JBoss module containing the RA code.
- Create the folder:`$JBOSS_HOME/modules/system/layers/base/org/apache/activemq/main/`
```
	mkdir  -p $JBOSS_HOME/modules/org/apache/activemq/main/  
```
- Copy all unpacked files of resource adapter to that folder, including the META-INF/ra.xml file.
```
	cd modules/org/apache/activemq/main/  
	wget http://repo1.maven.org/maven2/org/apache/activemq/activemq-rar/5.11.1/activemq-rar-5.11.1.rar  
	unzip activemq-rar-5.11.1.rar  
```
> The RAR archive contains many jars that are required to run ActiveMQ RA. It also contains the RA definition in META-INF/ra.xml
	
- Add module.xml file to the same folder with the following content:
```xml
 <module xmlns="urn:jboss:module:1.3" name="org.apache.activemq" slot="main" >
     <resources>
        <resource-root path="."/>
        <resource-root path="activemq-broker-5.11.1.jar"/>
        <resource-root path="activemq-client-5.11.1.jar"/>
        <resource-root path="activemq-jms-pool-5.11.1.jar"/>
        <resource-root path="activemq-kahadb-store-5.11.1.jar"/>
        <resource-root path="activemq-openwire-legacy-5.11.1.jar"/>
        <resource-root path="activemq-pool-5.11.1.jar"/>
        <resource-root path="activemq-protobuf-1.1.jar"/>
        <resource-root path="activemq-ra-5.11.1.jar"/>
        <resource-root path="activemq-spring-5.11.1.jar"/>
        <resource-root path="aopalliance-1.0.jar"/>
        <resource-root path="commons-pool-1.6.jar"/>
        <resource-root path="commons-logging-1.1.3.jar"/>
        <resource-root path="hawtbuf-1.11.jar"/>
        <resource-root path="spring-aop-3.2.11.RELEASE.jar"/>
        <resource-root path="spring-beans-3.2.11.RELEASE.jar"/>
        <resource-root path="spring-context-3.2.11.RELEASE.jar"/>
        <resource-root path="spring-core-3.2.11.RELEASE.jar"/>
        <resource-root path="spring-expression-3.2.11.RELEASE.jar"/>
        <resource-root path="xbean-spring-3.18.jar"/>
    </resources>
    <exports>
        <exclude path="org/springframework/**"/>
        <exclude path="org/apache/xbean/**"/>
        <exclude path="org/apache/commons/**"/>
        <exclude path="org/aopalliance/**"/>
        <exclude path="org/fusesource/**"/>
    </exports>
    <dependencies>
        <module name="javax.api"/>
        <module name="org.slf4j"/>
        <module name="javax.resource.api"/>
        <module name="javax.jms.api"/>
        <module name="javax.management.j2ee.api"/>
    </dependencies>
 </module>
 ```

## Setup WildFly standalone configuration
Once the org.apache.activemq module has been created, the WildFly standalone configuration must be updated to use the ActiveMQ RA as the default resource adapters for message-driven beans

- Open standalone/configuration/standalone-full.xml configuration file (we need full server profile)
- Add this snippet into the subsystem urn:jboss:domain:resource-adapters:2.0

```xml
<subsystem xmlns="urn:jboss:domain:resource-adapters:2.0">
	<resource-adapters>
		<resource-adapter id="activemq-rar-5.11.1.rar">
			<module slot="main" id="org.apache.activemq"/>
			<transaction-support>NoTransaction</transaction-support>
			<config-property name="ServerUrl">tcp://localhost:61616</config-property>
			<config-property name="Password">admin</config-property>
			<config-property name="UserName">admin</config-property>
			<connection-definitions>
			  <connection-definition class-name="org.apache.activemq.ra.ActiveMQManagedConnectionFactory" jndi-name="java:/AMQConnectionFactory" enabled="true" use-java-context="true" pool-name="AMQConnectionFactory">
			    <pool>
			       <min-pool-size>10</min-pool-size>
			       <max-pool-size>100</max-pool-size>
			    </pool>
			    </connection-definition>
			</connection-definitions>
			<admin-objects>
			    <admin-object class-name="org.apache.activemq.command.ActiveMQQueue" jndi-name="queue/JMSBridgeSourceQ" enabled="true" use-java-context="true" pool-name="source_queuet">  
			        <config-property name="PhysicalName">  
			            JMSBridgeSourceQ  
			        </config-property>  
			    </admin-object>  
			        <admin-object class-name="org.apache.activemq.command.ActiveMQQueue" jndi-name="queue/JMSBridgeTargetQ" enabled="true" use-java-context="true" pool-name="target_queue">  
			            <config-property name="PhysicalName">  
			                JMSBridgeTargetQ  
			            </config-property>  
			        </admin-object>   
			</admin-objects>
		</resource-adapter>
	</resource-adapters>
</subsystem>
```	

- Create Queues JMS
Command JBOSS CLI !!!! TODO

- This creates a bridge that will use the connection factory called ConnectionFactory to consume from the local queue with the JNDI name queue/JMSBridgeSourceQ. It will then use the connection factory called AMQConnectionFactory (which is created by our resource adapter) to send the messages to the queue with the JNDI name queue/JMSBridgeTargetQ. This is mapped by our resource adapter to the remote ActiveMQ queue. We also need to create a local queue called JMSBridgeSourceQ in the jms-destinations section of the configuration.

```xml
<hornetq-server>
	...
	<jms-destinations>
		<jms-queue name="JMSBridgeSourceQueue">
		    <entry name=" java:/queue/JMSBridgeSourceQ "/>
		    <entry name="java:jboss/exported/jms/queue/JMSBridgeSourceQ "/>
		    <durable>true</durable>
		</jms-queue>
		<jms-queue name="JMSBridgeTargetQueue">
		    <entry name=" java:/queue/JMSBridgeTargetQ "/>
		    <entry name="java:jboss/exported/jms/queue/JMSBridgeTargetQ "/>
		    <durable>true</durable>
		</jms-queue>
	</jms-destinations>
</hornetq-server>
```
This queue has two JNDI names, to allow it to be accessed both internally (by the bridge) and externally (by our client).

- The next step is to configure the bridge and the local queue. We edit the hornetq subsystem to add a JMS bridge after the definition of the hornetQ server.
```xml
<jms-bridge name="myBridge">
      <source>
        <connection-factory name=" AMQConnectionFactory "/>
        <destination name=" queue/JMSBridgeSourceQ "/>
    </source>
    <target>
        <connection-factory name=" ConnectionFactory "/>
        <destination name=" queue/JMSBridgeTargetQ "/>
    </target>
    <quality-of-service>AT_MOST_ONCE</quality-of-service>
    <failure-retry-interval>1000</failure-retry-interval>
    <max-retries>-1</max-retries>
    <max-batch-size>10</max-batch-size>
    <max-batch-time>100</max-batch-time>
</jms-bridge> 
```

- Start JBoss AS with command:
``` 	bin/standalone.sh -c standalone-full.xml ```

## ActiveMQ as an external messaging broker

- Download Apache ActiveMQ from 5.12 http://activemq.apache.org/activemq-5121-release.html
- Unpack the downloaded package
- Start the broker with: ```%AMQ_DIR%/bin/activemq.bat start ```
	
	
## Configure JCA endpoint for MDB !!!
TODO
