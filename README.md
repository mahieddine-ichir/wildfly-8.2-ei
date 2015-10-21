# wildfly-8.2-ei
A Wildfly 8.2 project for Entreprise integration


## Installer un broker externe ActiveMQ RA

- Télécharger le resource adapter d'ActiveMQ (version 5.11.1) et l'installer comme module dans WildFly
- Creation du dossier $JBOSS_HOME/modules/system/layers/base/org/apache/activemq/main/
- Copier le contenu de fichier rar dans le nouveau dossier
	unzip activemq-rar-5.11.1.rar
..* ce fichier rar contiens les differants jars pour le fonctionnement d'ActiveMQ RA

- Creation d'un fichier module.xml qui liste les differants fichiers contenu dans le dossier main.
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
  
* Ajouter de la configuration Standalone WildFly
	** déclarer la connexion dans le subsystem resource-adapters.
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

- creation de queues

- Pour qu'un MDB, il faut ajouter un Bridge
<hornetq-server>
...
- Ajouter le queue d'echange avec ActiveMQ
```xml
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
- Ajouter un Bridge
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

2/ lancement de Broker ActiveMQ
	- Telecharger le bin activemq 5.12
	- Extraire le contenu dans un dossier de travail
	- Lancement de broker  %AMQ_DIR%/bin/activemq.bat start
