# Streaming Data infra Module

**Goal**

- The basic requirements and overall architecture of real-time data warehouse project
- Common implementation schemes
- Write a canal client to collect binlog messages
- Google protobuf serialization
- the principle of Apache Canal data acquisition

## Real-time computing scenarios and technology selection

### The use of real-time computing in the industry

The company has adopted technologies such as Mapreduce and spark for the offline calculation. Why we should use the real-time calculation?

- The problem of offline computing is that the speed of data flow is **too slow**
- There are scenes with high requirements for real-time data
	- For example: Uber's risk control, Taobao's double 11 marketing screen, e-commerce shopping recommendation, and audience statistics of the Spring Festival Gala
	

### The Selection of real-time computing technology

Spark streaming、 Struct streaming、Storm、Kafka Streaming、Flink?

- Technical foundation of the company's employees
- Popularity
- Technology reuse
- Scenes



If the project team have very high requirements for latency, we can use Flink, the streaming processing framework at present, and adopt the native stream processing system to ensure low latency. It is also relatively perfect in API and fault tolerance. Its use and deployment are relatively simple. In addition, with blink contributed by Alibaba in China, Flink's functions will be more perfect in the future. 

This project: use  **Flink** to build a real-time computing platform

## Project implementation environment

### Data

- The order data has already existed, which will be written to MySQL.
- Binary log data (access log)

### Hardware

- 1 physical servers
- Service configuration
	- CPU x 2: main frequency 2.8 - 3.6, 24 cores (12 cores per CPU)
	- Memory (768gb / 1t)
	- Hard disk (1t x 8 SAS disk)
	- Network card (4 ports 2000m)

### Team

- 2 persons
	- Big data engineer (1 person)
	- Project client/sponsor (1 person)

## Demand analysis

### Project requirements

- At present, there are front-end visualization projects. The company needs a large screen to display order data and user access data.

![Screen Shot 2021-11-29 at 3.41.36 PM](/Users/cathyqin/Library/Application Support/typora-user-images/Screen Shot 2021-11-29 at 3.41.36 PM.png)

### Data source

#### PV/UV 

- From the event tracking data of the page, which will be sent to the web server.
- The web server directly writes this part of the data to the  **click of Kafka_ Log** in topic

#### Sales and order quantity

- From MySQL
- From the binary log and will be written to the topic of **order** in Kafka in real time via Apahce Canal


#### Shopping cart data and comment data

- Generally, the shopping cart data is not directly operated on MySQL, but will be written to Kafka (message queue) through the client program
- The comment data is also written to Kafka (message queue) through the client side

## Common software engineering models

### Waterfall model

Characteristics

1. The results of the previous development activity are used as the input of the next procedure.
3. Conduct **review on the implementation results of this activity**
	- If it is passed, then the project team proceed to the next development activity
	- If it fails, then the team will be returned to the previous or even more previous activities

Scope of use

1. The clients' requirements are clear, and there are few changes in the development process.
2. Developers are familiar with the business.

3. The user's use environment is relatively stable;

4. The development work has low requirements for user participation.

Advantages

1. The responsibilities of personnel are clear and specific, which is conducive to the organization and management of personnel in the process of large-scale software development
2. The implementation steps are clear and orderly, which is conducive to the research of software development methods and tools and ensure the quality / efficiency of large-scale projects

shortcoming

1. Generally, the development process cannot be reversed.

2. The actual project development is difficult to strictly follow the model

3. It is difficult for clients to clearly give all their needs.

4. The actual situation of software can only be seen in the later stage of project development, which requires customers to have enough patience

### Agile development

#### Introduction

- Taking the evolution of users' needs as the core in the project, the software development is carried out by iterative and step-by-step method.
- Divide a large project into several small projects that are interrelated but can also run independently, and complete them separately.

#### Shortcomings

- Agile pays attention to personnel communication. If there is many changes in the flow of project personnel, it will bring many difficulties for the maintenance. 
- People with strong experience in the project are required, and it is not easy to encounter bottleneck problems in the project.

## Solution

### Real time pipeline project architecture

<img src="assets/architecture.jpg" align="left" style="border:1px solid #999">

## Canal Briefing

### Introduction

* Based on MySQL database, it provides the incremental data subscription and consumption.
* In the early days, Alibaba had the business requirement of cross  synchronization due to the deployment of dual machine rooms in Hangzhou and the United States. The implementation method was mainly to obtain incremental changes based on business triggers.
* Since 2010, the business has gradually tried to obtain the synchronization for the incremental changes and database log parsing, resulting in a large number of database incremental subscription and consumption businesses. The businesses based on log incremental subscription and consumption include:
  * Database mirroring
  * Real time database backup
  * Index construction and real-time maintenance (splitting heterogeneous indexes, inverted indexes, etc.)
  * Cache refresh
  * Incremental data processing with business logic
* Current canal supports source MySQL versions, including 5.1. X, 5.5. X, 5.6. X, 5.7. X, and 8.0. 
* GitHub address: https://github.com/alibaba/canal

## Environment Deployment

### MySQL

- MySQL needs to enable the binlog write function first, and configure binlog format to row mode. 

- The configuration in /etc/my.cnf is as follows

  ```properties
  [mysqld]
  log-bin=mysql-bin # start binlog
  binlog-format=ROW # choose ROW format
  server_id=1 # 
  ```

- Authorize the canal linked MySQL account to have the permission to act as a MySQL slave. 

  ```sql
  CREATE USER root IDENTIFIED BY '123456';  
  GRANT ALL PRIVILEGES ON *.* TO 'root'@'%' ;
  FLUSH PRIVILEGES;
  ```

### Canal Installation

**Note**: the version used in this project is canal1.0.24

Environmental requirements:：

- Install zookeeper

- Decompress

	```shell
	mkdir /export/servers/canal
	tar -zxvf canal.deployer-1.0.24.tar.gz  -C /export/servers/canal/
	```

- After decompression, enter /export/servers/canal/ directory:

	```shell
	drwxr-xr-x. 2 root root 4096 2月   1 14:07 bin
	drwxr-xr-x. 4 root root 4096 2月   1 14:07 conf
	drwxr-xr-x. 2 root root 4096 2月   1 14:07 lib
	drwxrwxrwx. 2 root root 4096 4月   1 2017 logs
	```

- There are several configuration files under the conf of the canal server

  ~~~
  [root@node1 canal]# tree conf/ 
  conf/
  ├── canal.properties
  ├── example
  │   └── instance.properties
  ├── logback.xml
  └── spring
      ├── default-instance.xml
      ├── file-instance.xml
      ├── group-instance.xml
      ├── local-instance.xml
      └── memory-instance.xml
  ~~~

  - Let's first look at the first four configuration items of the **common** property of 'canal. Properties':

    ~~~properties
    canal.id= 1
    canal.ip=
    canal.port= 11111
    canal.zkServers=
    ~~~

    Canal.id is the number of canal. In the cluster environment, different canals have different IDs. We should note that it is different from MySQL server_ id.

    IP is not specified here. It defaults to the local machine. For example, my IP address is 192.168.1.120 and the port number is 11111. ZK is used for canal cluster.

  - Next, let's see the configuration related to **destinations** under 'canal. Properties':

    ~~~properties
    #################################################
    #########       destinations        ############# 
    #################################################
    canal.destinations = example
    canal.conf.dir = ../conf
    canal.auto.scan = true
    canal.auto.scan.interval = 5
    
    canal.instance.global.mode = spring 
    canal.instance.global.lazy = false
    canal.instance.global.spring.xml = classpath:spring/file-instance.xml
    ~~~

    Here, we can set multiple canal.destinations = example, such as example1, example2.
    
    But we need to create two corresponding folders, and there is an instance.properties file under each folder.
    
  - Spring is used for global canal instance management. The 'file instance. XML' here will eventually instantiate all destinations instances:
  
    ~~~xml
    <!-- properties -->
    <bean class="com.alibaba.otter.canal.instance.spring.support.PropertyPlaceholderConfigurer" lazy-init="false">
    	<property name="ignoreResourceNotFound" value="true" />
        <property name="systemPropertiesModeName" value="SYSTEM_PROPERTIES_MODE_OVERRIDE"/><!-- 允许system覆盖 -->
        <property name="locationNames">
        	<list>
            	<value>classpath:canal.properties</value>                     <value>classpath:${canal.instance.destination:}/instance.properties</value>
             </list>
        </property>
    </bean>
    
    <bean id="socketAddressEditor" class="com.alibaba.otter.canal.instance.spring.support.SocketAddressEditor" />
    <bean class="org.springframework.beans.factory.config.CustomEditorConfigurer"> 
       <property name="propertyEditorRegistrars">
    	   <list>
        		<ref bean="socketAddressEditor" />
           </list>
       </property>
    </bean>
    <bean id="instance" class="com.alibaba.otter.canal.instance.spring.CanalInstanceWithSpring">
    	<property name="destination" value="${canal.instance.destination}" />
        <property name="eventParser">
        	<ref local="eventParser" />
        </property>
        <property name="eventSink">
            <ref local="eventSink" />
        </property>
        <property name="eventStore">
            <ref local="eventStore" />
        </property>
        <property name="metaManager">
            <ref local="metaManager" />
        </property>
        <property name="alarmHandler">
            <ref local="alarmHandler" />
        </property>
    </bean>
    ~~~
  
- Modify instance configuration file

  vi conf/example/instance.properties
  ```properties
  ## mysql serverId，the slaveid here cannot be the same as the existing server_id in the myql cluster.
  canal.instance.mysql.slaveId = 1234
  
  #  Modity it to match our own database information
  #################################################
  ...
  canal.instance.master.address=192.168.1.120:3306
  # username/password
  ...
  canal.instance.dbUsername = root
  canal.instance.dbPassword = 123456
  #################################################
  ```

- Start

  ```
  sh bin/startup.sh
  ```

- Look through Canal log

  ```shell
  vi logs/canal/canal.log
  ```

  ```shell
  2021-10-05 22:45:27.967 [main] INFO  com.alibaba.otter.canal.deployer.CanalLauncher - ## start the canal server.
  2021-10-05 22:45:28.113 [main] INFO  com.alibaba.otter.canal.deployer.CanalController - ## start the canal server[10.1.29.120:11111]
  2021-10-05 22:45:28.210 [main] INFO  com.alibaba.otter.canal.deployer.CanalLauncher - ## the canal server is running now ......
  ```

- Look through instance log

  ```shell
  vi logs/example/example.log
  ```

  ```shell
  2021-10-05 22:50:45.636 [main] INFO  c.a.o.c.i.spring.support.PropertyPlaceholderConfigurer - Loading properties file from class path resource [canal.properties]
  2021-10-05 22:50:45.641 [main] INFO  c.a.o.c.i.spring.support.PropertyPlaceholderConfigurer - Loading properties file from class path resource [example/instance.properties]
  2021-10-05 22:50:45.803 [main] INFO  c.a.otter.canal.instance.spring.CanalInstanceWithSpring - start CannalInstance for 1-example 
  2021-10-05 22:50:45.810 [main] INFO  c.a.otter.canal.instance.spring.CanalInstanceWithSpring - start successful....
  ```

- Stop

  ```shell
  sh bin/stop.sh
  ```

## Canal client side programming

### creat client_demo

### Maven dependency

```xml
<dependencies>
	<dependency>
    	<groupId>com.alibaba.otter</groupId>
        <artifactId>canal.client</artifactId>
        <version>1.0.24</version>
    </dependency>
    <dependency>
        <groupId>com.alibaba</groupId>
        <artifactId>fastjson</artifactId>
        <version>1.2.58</version>
    </dependency>
</dependencies>
```

### Create the package in the canal_demo

| package name           | Description        |
| ---------------------- | ------------------ |
| com.itheima.canal_demo | Index for the code |

### Development steps

1. Create connector
2. Connect to the canal server and subscribe
3. Parse the canal message and print it

#### The format of Canal message

```java
Entry  
    Header  
        logfileName [binlog file name]  
        logfileOffset [binlog position]  
        executeTime [the timestamp of changes in the binlog]  
        schemaName   
        tableName  
        eventType [insert/update/delete]  
    entryType   [transaction BEGIN/transaction END/ROWDATA]  
    storeValue  [byte data, corresponding to RowChange]  
RowChange
    isDdl       [if there is a DDL, such as create table/drop table]
    sql         [detailed ddl sql]
rowDatas    [insert/update/delete data，]
    beforeColumns 
    afterColumns 
    Column
    index
    sqlType     [jdbc type]
    name        [column name]
    isKey       [if it is the key]
    updated     [if it is updated]
    isNull      [if the value null]
    value       [the context in the string format]
```

Code：

```json
public class CanalClientEntrance {
    public static void main(String[] args) {
        // 1. create connector
        CanalConnector connector = CanalConnectors.newSingleConnector(new InetSocketAddress("192.168.88.120",
                11111), "example", "canal", "canal");

        // batch size clarification
        int batchSize = 5 * 1024;
        boolean running = true;

        try {
            while(running) {
                // 2. connect
                connector.connect();
                // rollback last request to get the data
                connector.rollback();
                // subscribe the corresponding binlog
                connector.subscribe("itcast_shop.*");
                while(running) {
                    // get binlog in the batch size
                    Message message = connector.getWithoutAck(batchSize);
                    // get batchId
                    long batchId = message.getId();
                    // get the size of the batching binlog data
                    int size = message.getEntries().size();
                    if(batchId == -1 || size == 0) {

                    }
                    else {
                        printSummary(message);
                    }
                    // acknowlege that the batchId has successfully be consumed.
                    connector.ack(batchId);
                }
            }
        } finally {
            // disconnect
            connector.disconnect();
        }
    }

    private static void printSummary(Message message) {
        // for loop every binlog entity in the batching size
        for (CanalEntry.Entry entry : message.getEntries()) {
            // transaction begins
            if(entry.getEntryType() == CanalEntry.EntryType.TRANSACTIONBEGIN || entry.getEntryType() == CanalEntry.EntryType.TRANSACTIONEND) {
                continue;
            }

            // get log filename
            String logfileName = entry.getHeader().getLogfileName();
            // log offset
            long logfileOffset = entry.getHeader().getLogfileOffset();
            // get the execution timestamp for the SQL statement
            long executeTime = entry.getHeader().getExecuteTime();
            // get schemaname
            String schemaName = entry.getHeader().getSchemaName();
            // get tablename
            String tableName = entry.getHeader().getTableName();
            // get eventtype insert/update/delete
            String eventTypeName = entry.getHeader().getEventType().toString().toLowerCase();

            System.out.println("logfileName" + ":" + logfileName);
            System.out.println("logfileOffset" + ":" + logfileOffset);
            System.out.println("executeTime" + ":" + executeTime);
            System.out.println("schemaName" + ":" + schemaName);
            System.out.println("tableName" + ":" + tableName);
            System.out.println("eventTypeName" + ":" + eventTypeName);

            CanalEntry.RowChange rowChange = null;

            try {
                // get data，and parse the binary byte into rowchange entity
                rowChange = CanalEntry.RowChange.parseFrom(entry.getStoreValue());
            } catch (InvalidProtocolBufferException e) {
                e.printStackTrace();
            }

            // Iterate every changed record
            for (CanalEntry.RowData rowData : rowChange.getRowDatasList()) {
                // if it has been deleted
                if(entry.getHeader().getEventType() == CanalEntry.EventType.DELETE) {
                    System.out.println("---delete---");
                    printColumnList(rowData.getBeforeColumnsList());
                    System.out.println("---");
                }
                // if it has been updated
                else if(entry.getHeader().getEventType() == CanalEntry.EventType.UPDATE) {
                    System.out.println("---update---");
                    printColumnList(rowData.getBeforeColumnsList());
                    System.out.println("---");
                    printColumnList(rowData.getAfterColumnsList());
                }
                // if it has been inserted
                else if(entry.getHeader().getEventType() == CanalEntry.EventType.INSERT) {
                    System.out.println("---insert---");
                    printColumnList(rowData.getAfterColumnsList());
                    System.out.println("---");
                }
            }
        }
    }

    // print all the names and values of the column lists
    private static void printColumnList(List<CanalEntry.Column> columnList) {
        for (CanalEntry.Column column : columnList) {
            System.out.println(column.getName() + "\t" + column.getValue());
        }
    }
}
```

### Transform to JSON data

* Encapsulate the binlog log in a Map data structure, and convert it to JSON format using **fastjson**

Code：

```java
    // parse the binlog to json
    private static String binlogToJson(Message message) throws InvalidProtocolBufferException {
        // 1. create amp to store the parsed data.
        Map rowDataMap = new HashMap<String, Object>();

        // 2. iterate all the binlog in the message 
        for (CanalEntry.Entry entry : message.getEntries()) {
            // only deal with the transaction binlog
            if(entry.getEntryType() == CanalEntry.EntryType.TRANSACTIONBEGIN ||
            entry.getEntryType() == CanalEntry.EntryType.TRANSACTIONEND) {
                continue;
            }

         
            String logfileName = entry.getHeader().getLogfileName();
            long logfileOffset = entry.getHeader().getLogfileOffset();
            long executeTime = entry.getHeader().getExecuteTime();
            String schemaName = entry.getHeader().getSchemaName();
            String tableName = entry.getHeader().getTableName();
            // insert/update/delete
            String eventType = entry.getHeader().getEventType().toString().toLowerCase();

            rowDataMap.put("logfileName", logfileName);
            rowDataMap.put("logfileOffset", logfileOffset);
            rowDataMap.put("executeTime", executeTime);
            rowDataMap.put("schemaName", schemaName);
            rowDataMap.put("tableName", tableName);
            rowDataMap.put("eventType", eventType);

            // encapsulate the data in the column
            Map columnDataMap = new HashMap<String, Object>();
            // get all the changes in the row
            CanalEntry.RowChange rowChange = CanalEntry.RowChange.parseFrom(entry.getStoreValue());
            List<CanalEntry.RowData> columnDataList = rowChange.getRowDatasList();
            for (CanalEntry.RowData rowData : columnDataList) {
                if(eventType.equals("insert") || eventType.equals("update")) {
                    for (CanalEntry.Column column : rowData.getAfterColumnsList()) {
                        columnDataMap.put(column.getName(), column.getValue());
                    }
                }
                else if(eventType.equals("delete")) {
                    for (CanalEntry.Column column : rowData.getBeforeColumnsList()) {
                        columnDataMap.put(column.getName(), column.getValue());
                    }
                }
            }

            rowDataMap.put("columns", columnDataMap);
        }

        return JSON.toJSONString(rowDataMap);
    }
```

## Protocol Buffers

### Protocol Buffers introduction

* Protocal buffers (protobuf) is a technology of Google, used for structured data serialization and deserialization. It is commonly used in RPC system and continuous data storage system.
* It is similar to XML generation and parsing, but protobuf is more efficient than XML. However, protobuf generates **bytecode **, which is less readable than XML, similarly as JSON and Java serializable.
* It is very suitable for data storage or RPC data exchange format. Also, it can be used for language independent, platform independent and extensible serialization structure data format in communication protocol, and data storage.
* reference：https://zhuanlan.zhihu.com/p/53339153

### Idea protobuf plug-in installation

 install protobuf Support, and then restart

![assets-20191204152032939](assets/image-20191204152032939.png)

### Use ProtoBuf to serialize data

#### Maven dependency and plug-ins

```xml
<dependencies>
        <dependency>
            <groupId>com.google.protobuf</groupId>
            <artifactId>protobuf-java</artifactId>
            <version>3.4.0</version>
        </dependency>
</dependencies>

    <build>
        <extensions>
            <extension>
                <groupId>kr.motd.maven</groupId>
                <artifactId>os-maven-plugin</artifactId>
                <version>1.6.2</version>
            </extension>
        </extensions>
        <plugins>
            <!-- Protobuf plug-in -->
            <plugin>
                <groupId>org.xolstice.maven.plugins</groupId>
                <artifactId>protobuf-maven-plugin</artifactId>
                <version>0.5.0</version>
                <configuration>
                    <protoSourceRoot>${project.basedir}/src/main/proto</protoSourceRoot>
                    <protocArtifact>
                        com.google.protobuf:protoc:3.1.0:exe:${os.detected.classifier}
                    </protocArtifact>
                </configuration>
                <executions>
                    <execution>
                        <goals>
                            <goal>compile</goal>
                        </goals>
                    </execution>
                </executions>
            </plugin>
        </plugins>
    </build>
```

#### Write proto demo

```protobuf
syntax = "proto3";
option java_package = "com.itheima.protobuf";
option java_outer_classname = "DemoModel";

message User {
    int32 id = 1;
    string name = 2;
    string sex = 3;
}
```

Comparison between protobuf and Java types

| .proto Type | Java Type  |
| ----------- | ---------- |
| double      | double     |
| float       | float      |
| int32       | int        |
| int64       | long       |
| uint32      | int        |
| uint64      | long       |
| sint32      | int        |
| sint64      | long       |
| fixed32     | int        |
| fixed64     | long       |
| sfixed32    | int        |
| sfixed64    | long       |
| bool        | boolean    |
| string      | String     |
| bytes       | ByteString |



#### Execute the protobuf:compile command

* compile the proto file into java code

<img src="assets/image-20191203111833702.png" align="left"/>

<img src="assets/image-20191204152444786.png" align="left" />

####  Serialize and deserialize using protobuf

```java
public class ProtoBufDemo {
    public static void main(String[] args) throws InvalidProtocolBufferException {
        DemoModel.User.Builder builder = DemoModel.User.newBuilder();
        builder.setId(1);
        builder.setName("Cathy");
        builder.setSex("F");

        byte[] bytes = builder.build().toByteArray();
        System.out.println("--protobuf---");
        for (byte b : bytes) {
            System.out.print(b);
        }
        System.out.println();
        System.out.println("---");

        DemoModel.User user = DemoModel.User.parseFrom(bytes);

        System.out.println(user.getName());
    }
}
```

### Transform BINLOG into ProtoBuf message

#### write proto description file：CanalModel.proto

```protobuf
syntax = "proto3";
option java_package = "com.itheima.canal_demo";
option java_outer_classname = "CanalModel";

/* row data */
message RowData {
    string logfilename = 15;
    uint64 logfileoffset = 14;
    uint64 executeTime = 1;
    string schemaName = 2;
    string tableName = 3;
    string eventType = 4;

    /* column data */
    map<string, string> columns = 5;
}
```
#### Add binglogToProtoBuf as Protobuf

```java
    // parse the binlog intoProtoBuf
    private static byte[] binlogToProtoBuf(Message message) throws InvalidProtocolBufferException {
        // 1. create CanalModel.RowData
        CanalModel.RowData.Builder rowDataBuilder = CanalModel.RowData.newBuilder();

        // 1. iterate the binlog in the message
        for (CanalEntry.Entry entry : message.getEntries()) {
       
            if(entry.getEntryType() == CanalEntry.EntryType.TRANSACTIONBEGIN ||
                    entry.getEntryType() == CanalEntry.EntryType.TRANSACTIONEND) {
                continue;
            }

            String logfileName = entry.getHeader().getLogfileName();
      
            long logfileOffset = entry.getHeader().getLogfileOffset();
       
            long executeTime = entry.getHeader().getExecuteTime();
     
            String schemaName = entry.getHeader().getSchemaName();
          
            String tableName = entry.getHeader().getTableName();
            // insert/update/delete
            String eventType = entry.getHeader().getEventType().toString().toLowerCase();

            rowDataBuilder.setLogfilename(logfileName);
            rowDataBuilder.setLogfileoffset(logfileOffset);
            rowDataBuilder.setExecuteTime(executeTime);
            rowDataBuilder.setSchemaName(schemaName);
            rowDataBuilder.setTableName(tableName);
            rowDataBuilder.setEventType(eventType);

            // get the changes in the row
            CanalEntry.RowChange rowChange = CanalEntry.RowChange.parseFrom(entry.getStoreValue());
            List<CanalEntry.RowData> columnDataList = rowChange.getRowDatasList();
            for (CanalEntry.RowData rowData : columnDataList) {
                if(eventType.equals("insert") || eventType.equals("update")) {
                    for (CanalEntry.Column column : rowData.getAfterColumnsList()) {
                        rowDataBuilder.putColumns(column.getName(), column.getValue().toString());
                    }
                }
                else if(eventType.equals("delete")) {
                    for (CanalEntry.Column column : rowData.getBeforeColumnsList()) {
                        rowDataBuilder.putColumns(column.getName(), column.getValue().toString());
                    }
                }
            }
        }

        return rowDataBuilder.build().toByteArray();
    } 
```

## Canal mechanism

### MySQL master slave duplication

![mysql replication](assets/image.png)

- MySQL master node writes the data changes to the binary log.
- MySQL slave node copies the binary log events of the master to its relay log.
- MySQL slave replays the events in the relay log and reflects the data changes to its own data, so as to achieve data consistency.

>**binlog in mysql**
>
>It records all DDL and DML statements in the form of events, and the time consumed by the execution of the statements. It is mainly used for backup and data synchronization.
>
>Binlog has three types： STATEMENT、ROW、MIXED 
>
>*  STATEMENT records the executed SQL statements
>* ROW are real row data records.
>*  MIXED records are 1 + 2, and the mode of 1 is preferred

> **Explanation of terms **:
>
> *What is a relay log*
>
> The slave I / O thread reads the binary log of the master server and records it in the local file of the slave server. Then, the slave SQL thread reads the content of the relay log and applies it to the slave server, so as to keep the data of the slave server and the master server consistent

### Canal working mechanism

 ![img](assets/68747470733a2f2f696d672d626c6f672e6373646e696d672e636e2f32303139313130343130313733353934372e706e67.png)

- Canal simulates the interaction protocol of MySQL slave, disguises itself as MySQL slave, and sends dump protocol to MySQL master
- MySQL master node receives a dump request and starts pushing binary logs to slave (i.e. canal)
- Canal parses binary log object (originally byte stream)

### Structure 

- Server represents a canal running instance, corresponding to a JVM
- Instance corresponds to a data queue (one canal server corresponds to 1.. n instances)
- Sub modules under instance:
  - eventParser: access the data source, simulate the slave protocol, interact with the master, and analyze the protocol
  - eventSink: the link between the parser and store for data filtering, processing and distribution
  - eventStore: data storage
  - metaManager: incremental subscription & consumption manager

Before sending the dump command to MySQL, the eventparser will first obtain the location of the last successful parsing from the log position (if it is started for the first time, then **obtain the initial specified location**  or the binlog location of the current data segment). 

After MySQL receives the dump command, the eventparser will parse the binlog data from MySQL and pass it to eventsink, and update the log position after the transmission succeeds. The flow chart is as follows:

![image-20200214104557452](assets\image-20200214104557452.png)

- EventSink plays a function like a channel, which can *filter, distribute / route (1: n), merge (n: 1) and process data*. EventSink is the bridge between eventParser and eventStore.
- The implementation mode of eventStore is the memory mode, whose structure is a ring buffer. Three pointers (put, get and ACK) identify the location of data storage and reading.
- Metamanager is the incremental subscription & consumption information manager. The protocols between incremental subscription and consumption include get / ACK / rollback, respectively:
  -  Message getWithoutAck(int batchSize)，which allows to specify the batchsize. The object returned each time is message, including batch ID [unique ID] and entries [specific data object]
  -  Void rollback (long batchid), is to roll back the last get request and retrieves the data. Submit the batchid   obtained by get to avoid misoperation.
  -  Void ack (long batchid), confirms that the consumption has been successful, and notifies the server to delete the data. 

### server/client protocol 

#### canal client & server

The communication between canal client and canal server is in C / S mode. The client adopts NIO and the server adopts netty.

After the canal server is started, if there is no canal client, the canal server will not pull binlog from mysql.

That is, when the canal client actively initiates a pull request, the server will simulate a MySQL slave node to pull binlog from the master node.

Usually, the canal client is an endless loop, so the client always calls the get method, and the server will always pull the binlog

~~~
BIO、NIO、AIO
The ways of IO are divided into three categories，BIO(Blocking synchronous I/O)、NIO(non-blocking synchronous I/O)、AIO(non-blocking asynchronous I/O).

Blocking synchronous (BIO)：After initiating an IO operation,the user process must wait for the completion of the IO operation.

non-blocking synchronous (NIO): In this way, after initiating an IO operation, the user process can return to do other things but it needs to ask whether the IO operation is ready from time to time, so as to introduce unnecessary waste of CPU resources.

non-blocking asynchronous (AIO):The read and write method return types are Future objects. The Future model is asynchronous, and its core idea is to remove the waiting time of the main function.

reference：https://www.cnblogs.com/straybirds/p/9479158.html
~~~



```java
public class AbstractCanalClientTest {
    protected void process() {
        int batchSize = 5 * 1024; 
        try {
            connector.connect(); // connect the server
            connector.subscribe(); // subscribe
            // keep send request to canal server, thus canal server can fetch binlog from mysql
            while (running) { 
                Message message = connector.getWithoutAck(batchSize); 
                long batchId = message.getId();
                int size = message.getEntries().size();
                printSummary(message, batchId, size);
                printEntry(message.getEntries());
                connector.ack(batchId); 
                //connector.rollback(batchId); // rollback if the process fails
            }
        } finally {
            connector.disconnect();
        }
    }
}
```

The subscription / consumption between the canal client and the canal server is incremental. The flow chart is as follows: (the C-end is the canal client and the S-end is the canal server)

![img](assets/687474703a2f2f646c322e69746579652e636f6d2f75706c6f61642f6174746163686d656e742f303039302f363437302f37646537383537632d653362352d333061352d386337662d3936356536356639616137382e6a7067.jpg)

When a canal client calls the ` connect() `method, the type of packet (packettype) sent is:

1. [**handshake**]

2. [**ClientAuthentication**]

The canal client calls the 'subscribe()' method with the type of [* * subscription * *].

The corresponding server uses netty to process RPC requests ([` canalserverwithnetty ']

~~~java
public class CanalServerWithNetty extends AbstractCanalLifeCycle implements CanalServer {
    public void start() {
        bootstrap.setPipelineFactory(new ChannelPipelineFactory() {
            public ChannelPipeline getPipeline() throws Exception {
                ChannelPipeline pipelines = Channels.pipeline();
                pipelines.addLast(FixedHeaderFrameDecoder.class.getName(), new FixedHeaderFrameDecoder());
                // deal with HANDSHAKE request in the client side
                pipelines.addLast(HandshakeInitializationHandler.class.getName(),
                    new HandshakeInitializationHandler(childGroups));
                // deal with CLIENTAUTHENTICATION request in the client side
                pipelines.addLast(ClientAuthenticationHandler.class.getName(),
                    new ClientAuthenticationHandler(embeddedServer));

                // deal with the session request, including SUBSCRIPTION，GET
                SessionHandler sessionHandler = new SessionHandler(embeddedServer);
                pipelines.addLast(SessionHandler.class.getName(), sessionHandler);
                return pipelines;
            }
        });
    }
}
~~~

ClientAuthenticationHandler handles the authentication，it will remove the HandshakeInitializationHandler and [ClientAuthenticationHandler](https://github.com/alibaba/canal/blob/master/server/src/main/java/com/alibaba/otter/canal/server/netty/handler/ClientAuthenticationHandler.java#L81)。
The most important is the [**SessionHandler**](https://github.com/alibaba/canal/blob/master/server/src/main/java/com/alibaba/otter/canal/server/netty/handler/SessionHandler.java)。

For example, when the client side sends GET requestand server side gets binlog from mysql，then sends **MESSAGES** to the client, the rpc is like below：

SimpleCanalConnector sends [**GET**](https://github.com/alibaba/canal/blob/master/client/src/main/java/com/alibaba/otter/canal/client/impl/SimpleCanalConnector.java#L272) request，and read the corresponding result:

~~~java
public Message getWithoutAck(int batchSize, Long timeout, TimeUnit unit) throws CanalClientException {
    waitClientRunning();
    int size = (batchSize <= 0) ? 1000 : batchSize;
    long time = (timeout == null || timeout < 0) ? -1 : timeout; // -1 represents no timeout control 
    if (unit == null) unit = TimeUnit.MILLISECONDS;  // ms by default

    // client sends GET request
    writeWithHeader(Packet.newBuilder()
        .setType(PacketType.GET)
        .setBody(Get.newBuilder()
            .setAutoAck(false)
            .setDestination(clientIdentity.getDestination())
            .setClientId(String.valueOf(clientIdentity.getClientId()))
            .setFetchSize(size)
            .setTimeout(time)
            .setUnit(unit.ordinal())
            .build()
            .toByteString())
        .build()
        .toByteArray());
    // client get the result    
    return receiveMessages();
}

private Message receiveMessages() throws IOException {
    // read the data package from the server
    Packet p = Packet.parseFrom(readNextPacket());
    switch (p.getType()) {
        case MESSAGES: {
            Messages messages = Messages.parseFrom(p.getBody());
            Message result = new Message(messages.getBatchId());
            for (ByteString byteString : messages.getMessagesList()) {
                result.addEntry(Entry.parseFrom(byteString));
            }
            return result;
        }
    }
}
~~~

the server SessionHandler process the [**GET**](https://github.com/alibaba/canal/blob/master/server/src/main/java/com/alibaba/otter/canal/server/netty/handler/SessionHandler.java#L105) request sent by the client:

~~~java
case GET:
    // get the data package sent by the client and encapsulate as Get object
    Get get = CanalPacket.Get.parseFrom(packet.getBody());
    // destination represents canal instance
    if (StringUtils.isNotEmpty(get.getDestination()) && StringUtils.isNotEmpty(get.getClientId())) {
        clientIdentity = new ClientIdentity(get.getDestination(), Short.valueOf(get.getClientId()));
        Message message = null;
        if (get.getTimeout() == -1) {// if it is the default value
            message = embeddedServer.getWithoutAck(clientIdentity, get.getFetchSize());
        } else {
            TimeUnit unit = convertTimeUnit(get.getUnit());
            message = embeddedServer.getWithoutAck(clientIdentity, get.getFetchSize(), get.getTimeout(), unit);
        }
        // set the type of data package sent to the client is MESSAGES   
        Packet.Builder packetBuilder = CanalPacket.Packet.newBuilder();
        packetBuilder.setType(PacketType.MESSAGES);
        // create Message
        Messages.Builder messageBuilder = CanalPacket.Messages.newBuilder();
        messageBuilder.setBatchId(message.getId());
        if (message.getId() != -1 && !CollectionUtils.isEmpty(message.getEntries())) {
            for (Entry entry : message.getEntries()) {
                messageBuilder.addMessages(entry.toByteString());
            }
        }
        packetBuilder.setBody(messageBuilder.build().toByteString());
        // write the data and return it to the client side
        NettyUtils.write(ctx.getChannel(), packetBuilder.build().toByteArray(), null);
    }
~~~

Reference：[CanalProtocol.proto](https://github.com/alibaba/canal/blob/master/protocol/src/main/java/com/alibaba/otter/canal/protocol/CanalProtocol.proto)

get/ack/rollback briefing：

- Message getWithoutAck(int batchSize)
	- it is allowed to specify batchsize. The object returned each time is message, including:
		* batch id 
		* Entries specific data object, corresponding data object format：[EntryProtocol.proto](https://github.com/alibaba/canal/blob/master/protocol/src/main/java/com/alibaba/otter/canal/protocol/EntryProtocol.proto)
- getWithoutAck(int batchSize, Long timeout, TimeUnit unit)
	- Compared with getwithoutack (int batchsize), it is allowed to set the timeout time to obtain the data
		* Get enough batchsize records or exceed timeout time
		* Timeout = 0, wait until there is enough batchsizevoid rollback(long batchId)
	
	- Roll back the last get request and retrieve the data. Submit based on the batchid obtained by get to avoid misoperation.
- void ack(long batchId)
	- Confirm that the consumption is successful and notify the server to delete the data. Submit based on the batchid obtained by get to avoid misoperation

The canal message structure corresponding to entryprotocol.protocol is as follows:

~~~xml
Entry  
    Header  
        logfileName [binlog file name]  
        logfileOffset [binlog position]  
        executeTime 
        schemaName   
        tableName  
        eventType [insert/update/delete类型]  
    entryType  
    storeValue  [byte data, RowChange]  
      
RowChange  
    isDdl       [create table/drop table]  
    sql         [ddl sql]  
    rowDatas    [insert/update/delete]  
        beforeColumns 
        afterColumns 
          
Column   
    index         
    sqlType     [jdbc type]  
    name        [column name]  
    isKey         
    updated      
    isNull     
    value    
~~~

In sessionhandler, when the server handles other types of requests from the client, it will call [CanalServerWithEmbedded](https://github.com/alibaba/canal/blob/master/server/src/main/java/com/alibaba/otter/canal/server/embedded/CanalServerWithEmbedded.java):

~~~java
case SUBSCRIPTION:
        Sub sub = Sub.parseFrom(packet.getBody());
        embeddedServer.subscribe(clientIdentity);
case GET:
        Get get = CanalPacket.Get.parseFrom(packet.getBody());
        message = embeddedServer.getWithoutAck(clientIdentity, get.getFetchSize());
case CLIENTACK:
        ClientAck ack = CanalPacket.ClientAck.parseFrom(packet.getBody());
        embeddedServer.ack(clientIdentity, ack.getBatchId());
case CLIENTROLLBACK:
        ClientRollback rollback = CanalPacket.ClientRollback.parseFrom(packet.getBody());
        embeddedServer.rollback(clientIdentity);// 回滚所有批次

~~~

So the real processing logic is in canal server with embedded. Here's our focus:

#### **CanalServerWithEmbedded**

Canalserver contains multiple instances, and its member variable 'canalinstances' records the instance name and instnace mapping relationship.
Because it is a map, the same instance name is not allowed for the same server (in this case, the instance name is example), For example, you cannot have two examples on the same server at the same time. However, example1 and example2 are allowed on a server.

~~~java
public class CanalServerWithEmbedded extends AbstractCanalLifeCycle implements CanalServer, CanalService {
    private Map<String, CanalInstance> canalInstances;
    private CanalInstanceGenerator     canalInstanceGenerator;
}
~~~

The following figure shows that a server is configured with two canal instances, and each client is connected to one instance.

Each canal instance is simulated as a MySQL slave, so the slaveid of each instance must be different.

For example, the IDs of the two instances in the figure are 1234 and 1235 respectively. They will pull the binlog of the MySQL master node.

![instances](assets/image-202002011704001.png)

Here, each canal client corresponds to an instance. When each client is started,

It will specify a destination, which represents the name of the instance.

Therefore, the parameters of canalserverwithembedded when processing various requests have clientity,

Get the destination from clientidentity to get the corresponding canalinstance.



The corresponding relationship of each component:

- The canal client finds the corresponding canal instance in the canal server through destination/
- Multiple canal instances can be configured for one canal server.

Take the subscription method of canalserverwithembedded as an example:

1. Obtain canalinstance according to the client ID

2. Subscribe to the current client from the metadata manager of canalinstance

3. Get the cursor of the client from metadata management

4. Notify canalinstance that the subscription relationship has changed

>Note: the subscription method is used to add a new table to MySQL. The client did not synchronize this table before. Now it needs to be synchronized, so it needs to be re subscribed.

~~~java
public void subscribe(ClientIdentity clientIdentity) throws CanalServerException {
  
    CanalInstance canalInstance = canalInstances.get(clientIdentity.getDestination());
    if (!canalInstance.getMetaManager().isStart()) {
        canalInstance.getMetaManager().start(); 
    }
    canalInstance.getMetaManager().subscribe(clientIdentity); 
    Position position = canalInstance.getMetaManager().getCursor(clientIdentity);
    if (position == null) {
        position = canalInstance.getEventStore().getFirstPosition();
        if (position != null) {
            canalInstance.getMetaManager().updateCursor(clientIdentity, position); cursor
        }
    }
    =
    canalInstance.subscribeChange(clientIdentity);
}
~~~

Each canalinstance includes four components: **eventparser, eventsink, eventstore and metamanager**.

The main processing methods of the server include get / ACK / rollback. These three methods all use several internal components above the instance, mainly eventstore and metamanager:

- Before that, you should understand the meaning of eventstore. Eventstore is a ringbuffer with three pointers: **put, get and ACK**.

  

- Put: after the canal server pulls data from MySQL and puts it into memory, put increases

- Get: the consumer (canal client) consumes data from memory, and get increases

- ACK: when consumer consumption is completed, ACK increases. And the data in put that has been acked will be deleted

The relationship between these three operations and the instance component is as follows:

![instances](assets/image-202002011704002.png)

There are several ways for the client to obtain MySQL binlog through the canal server (get method and getwithoutack):

- If the timeout is null, the tryget method is used to obtain it immediately
- If timeout is not null
  1. If timeout is 0, get blocking method is adopted to obtain data without setting timeout until there is enough batchsize data
  2. If timeout is not 0, get + timeout is used to obtain data. If there is not enough data in batchsize after timeout, how much is returned

~~~java
private Events<Event> getEvents(CanalEventStore eventStore, Position start, int batchSize, Long timeout,
                                TimeUnit unit) {
    if (timeout == null) {
        return eventStore.tryGet(start, batchSize); 
    } else if (timeout <= 0){
        return eventStore.get(start, batchSize); 
    } else {
        return eventStore.get(start, batchSize, timeout, unit); 
    }
}
~~~

> Note: the implementation of eventstore adopts ringbuffer ring buffer similar to disruptor. The implementation class of ringbuffer is memoryeventstorewithbuffer

The difference between the get method and the getwithoutack method is:

- The get method immediately calls ack
- The getwithoutack method does not call ack

#### **EventStore**

Take 10 pieces of data as an example. Initially, current = - 1, the first element starts with next = 0, end = 9, and loops ` [0,9] 'all elements.

List elements are (a, B, C, D, e, F, G, h, I, J)

| next | entries[next] | next-current-1 | list element |
| :--: | :-----------: | :------------: | :----------: |
|  0   |  entries[0]   |   0-(-1)-1=0   |      A       |
|  1   |  entries[1]   |   1-(-1)-1=1   |      B       |
|  2   |  entries[2]   |   2-(-1)-1=2   |      C       |
|  3   |  entries[3]   |   3-(-1)-1=3   |      D       |
|  .   |     ……….      |      ……….      |      .       |
|  9   |  entries[9]   |   9-(-1)-1=9   |      J       |

After the first batch of 10 elements are put, putsequence is set to end = 9. Suppose the second batch put five more elements: (K,L,M,N,O)

Current = 9, start next = 9 + 1 = 10, end = 9 + 5 = 14. After put is completed, putsequence is set to end = 14.

| next | entries[next] | next-current-1 | list element |
| :--: | :-----------: | :------------: | :----------: |
|  10  |  entries[10]  |   10-(9)-1=0   |      K       |
|  11  |  entries[11]  |   11-(9)-1=1   |      L       |
|  12  |  entries[12]  |   12-(9)-1=2   |      M       |
|  13  |  entries[13]  |   13-(9)-1=3   |      N       |
|  14  |  entries[14]  |   14-(9)-1=3   |      O       |

Assuming that the maximum size of the ring buffer is 15 (16MB in the source code), the above two batches produce a total of 15 elements, just filling the ring buffer.

If another put event comes in and there is no available slot because the ring buffer is full, the put operation will be blocked until it is consumed.

The following is the code for put to fill the ring buffer. Check that the available slot (checkfreeslotat method) is in several put methods.

~~~java
public class MemoryEventStoreWithBuffer extends AbstractCanalStoreScavenge implements CanalEventStore<Event>, CanalStoreScavenge {
    private static final long INIT_SQEUENCE = -1;
    private int               bufferSize    = 16 * 1024;
    private int               bufferMemUnit = 1024;                         // memsize，1kb by default
    private int               indexMask;
    private Event[]           entries;

    // record put/get/ack
    private AtomicLong        putSequence   = new AtomicLong(INIT_SQEUENCE); 
    private AtomicLong        getSequence   = new AtomicLong(INIT_SQEUENCE); 
    private AtomicLong        ackSequence   = new AtomicLong(INIT_SQEUENCE); 
  	//When starting the eventstore, create a buffer of the specified size. The size of the event array is 16 * 1024
		//In other words, if the number is counted, the array can hold 16000 events. If you count the memory, the size is 16MB
    public void start() throws CanalStoreException {
        super.start();
        indexMask = bufferSize - 1;
        entries = new Event[bufferSize];
    }

    private void doPut(List<Event> data) {
        long current = putSequence.get(); 
        long end = current + data.size(); 
       // Write the data first, and then update the corresponding cursor. In case of high concurrency, the putsequence will be visible by the get request, taking out the old entry value in the ringbuffer
        for (long next = current + 1; next <= end; next++) {
            entries[getIndex(next)] = data.get((int) (next - current - 1));
        }
        putSequence.set(end);
    } 
}
~~~

Put is production data and get is consumption data. Get must not exceed put. For example, put has 10 pieces of data, and get can only get 10 pieces of data at most. However, sometimes put and get are not equal in order to ensure the speed of get processing.

Put can be regarded as a producer and get as a consumer. Producers can be fast, while consumers can consume slowly. For example, put has 1000 pieces of data, while get only needs to process 10 pieces of data at a time.

The previous example is still used to illustrate the process of get. Initially, current = - 1. It is assumed that put has two batches of data, a total of 15 pieces, maxablesequence = 14, and the batchsize of get is assumed to be 10.

Initially, next = current = - 1, end = - 1. Through startposition, next = 0 will be set. Finally, end is assigned 9, that is, there are 10 elements in the circular buffer [0,9].

~~~java
private Events<Event> doGet(Position start, int batchSize) throws CanalStoreException {
    LogPosition startPosition = (LogPosition) start;

    long current = getSequence.get();
    long maxAbleSequence = putSequence.get();
    long next = current;
    long end = current;
    // If startposition is null, it indicates that it is the first time. The default is + 1
    if (startPosition == null || !startPosition.getPostion().isIncluded()) { //After the first subscription, you need to include the start location to prevent the loss of the first record
        next = next + 1;
    }

    end = (next + batchSize - 1) < maxAbleSequence ? (next + batchSize - 1) : maxAbleSequence;
    for (; next <= end; next++) {
        Event event = entries[getIndex(next)];
        if (ddlIsolation && isDdl(event.getEntry().getHeader().getEventType())) {
            
            if (entrys.size() == 0) {
                entrys.add(event);// If there is no DML event, add the current DDL event
                end = next; // update the end point
            } else {
                // If there is a DML event before, it is returned directly. Because the current next record is not included, it needs to go back to a position.
                end = next - 1; 
            }
            break;
        } else {
            entrys.add(event);
        }
    }

    getSequence.compareAndSet(current, end)
}
~~~

The upper limit of ACK operation is get. It is assumed that 15 pieces of data are put and 10 pieces of data are get, and only 10 pieces of data can be acked at most. The purpose of ACK is to clear the data that has been got in the buffer.

~~~java
public void ack(Position position) throws CanalStoreException {
    cleanUntil(position);
}

public void cleanUntil(Position position) throws CanalStoreException {
    long sequence = ackSequence.get();
    long maxSequence = getSequence.get();

    boolean hasMatch = false;
    long memsize = 0;
    for (long next = sequence + 1; next <= maxSequence; next++) {
        Event event = entries[getIndex(next)];
        memsize += calculateSize(event);
        boolean match = CanalEventUtils.checkPosition(event, (LogPosition) position);
        if (match) {
            hasMatch = true;

            if (batchMode.isMemSize()) {
                ackMemSize.addAndGet(memsize);
                
                for (long index = sequence + 1; index < next; index++) {
                    entries[getIndex(index)] = null;
                }
            }

            ackSequence.compareAndSet(sequence, next)
        }
    }
}
~~~

rollback:

~~~java
public void rollback() throws CanalStoreException {
    getSequence.set(ackSequence.get());
    getMemSize.set(ackMemSize.get());
}
~~~







