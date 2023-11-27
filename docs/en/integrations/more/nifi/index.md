---
sidebar_label:  NiFi
sidebar_position: 12
keywords: [clickhouse, nifi, connect, integrate, etl, data integration]
slug: /en/integrations/nifi
description: Stream data into ClickHouse using NiFi data pipelines
---
import ConnectionDetails from '@site/docs/en/_snippets/_gather_your_details_http.mdx';

# Connect Apache NiFi to ClickHouse

<a href="https://nifi.apache.org/" target="_blank">Apache NiFi</a> is an open-source workflow management software designed to automate data flow between software systems. It allows the creation of ETL data pipelines and is shipped with more than 300 data processors. This step-by-step tutorial shows how to connect Apache NiFi to ClickHouse as both a source and destination, and to load a sample dataset.

## 1. Gather your connection details

To connect to ClickHouse with HTTP(S) you need this information:

The HOST and PORT: typically, the port is 8443 when using TLS or 8123 when not using TLS.

The DATABASE NAME: out of the box, there is a database named default, use the name of the database that you want to connect to.

The USERNAME and PASSWORD: out of the box, the username is default. Use the username appropriate for your use case.

The details for your ClickHouse Cloud service are available in the ClickHouse Cloud console. Select the service that you will connect to and click Connect:

![](./cloud-connect-button.png)

Choose HTTPS, and the details are available in an example curl command.

![](./connection-details-https.png)

If you are using self-managed ClickHouse, the connection details are set by your ClickHouse administrator.

## 2. Download and run Apache NiFi

1. For a new setup, download the binary from https://nifi.apache.org/download.html and start by running `./bin/nifi.sh start`


## 3. Download the ClickHouse JDBC driver

1. Visit the <a href="https://github.com/ClickHouse/clickhouse-java/releases" target="_blank">ClickHouse JDBC driver release page</a> on GitHub and look for  the latest JDBC release version
2. In the release version, click on "Show all xx assets" and look for the JAR file containing the keyword "shaded" or "all", for example, `clickhouse-jdbc-0.5.0-all.jar`
3. Place the JAR file in a folder accessible by Apache NiFi and take note of the absolute path

## 4. Add DBCPConnectionPool Controller Service and configure its properties

1. To configure a Controller Service in Apache NiFi, visit the NiFi Flow Configuration page by clicking on the "gear" button

    ![NiFi Flow Configuration.](./nifi_01.png)

2. Select the Controller Services tab and add a new Controller Service by clicking on the `+` button at the top right

    ![Add controller service.](./nifi_02.png)

3. Search for `DBCPConnectionPool` and click on the "Add" button

    ![Search for DBCPConnectionPool](./nifi_03.png)

4. The newly added DBCPConnectionPool will be in an Invalid state by default. Click on the "gear" button to start configuring

    ![NiFi flow configuration](./nifi_04.png)

5. Under the "Properties" section, input the following values

  | Property                    | Value                                                              | Remark                                                                        |
  | --------------------------- | ------------------------------------------------------------------ | ----------------------------------------------------------------------------- |
  | Database Connection URL     | jdbc:ch:https://HOSTNAME:8443/default?ssl=true                     | Replace HOSTNAME in the connection URL accordingly                            |
  | Database Driver Class Name  | com.clickhouse.jdbc.ClickHouseDriver                               ||
  | Database Driver Location(s) | /etc/nifi/nifi-X.XX.X/lib/clickhouse-jdbc-0.X.X-patchXX-shaded.jar | Absolute path to the ClickHouse JDBC driver JAR file                          |
  | Database User               | default                                                            | ClickHouse username                                                           |
  | Password                    | password                                                 | ClickHouse password                                                           |

6. In the Settings section, change the name of the Controller Service to "ClickHouse JDBC" for easy reference

    ![NiFi flow configuration.](./nifi_05.png)

7. Activate the DBCPConnectionPool Controller Service by clicking on the "lightning” button and then the "Enable" button

    ![NiFi flow configuration.](./nifi_06.png)

    <br/>

    ![NiFi flow configuration](./nifi_07.png)

8. Check the Controller Services tab and ensure that the Controller Service is enabled

    ![NiFi flow configuration](./nifi_08.png)

## 5. Read from a table using the ExecuteSQL processor

1. Add an ​​ExecuteSQL processor, along with the appropriate upstream and downstream processors

    ![ExecuteSQL processor](./nifi_09.png)

2. Under the "Properties” section of the ​​ExecuteSQL processor, input the following values

    | Property                            | Value                                | Remark                                                  |
    |-------------------------------------|--------------------------------------|---------------------------------------------------------|
    | Database Connection Pooling Service | ClickHouse JDBC                      | Select the Controller Service configured for ClickHouse |
    | SQL select query                    | SELECT * FROM system.metrics         | Input your query here                                   |

3. Start the ​​ExecuteSQL processor

    ![ExecuteSQL processor](./nifi_10.png)

4. To confirm that the query has been processed successfully, inspect one of the FlowFile in the output queue

    ![ExecuteSQL processor](./nifi_11.png)

5. Switch view to "formatted” to view the result of the output FlowFile

    ![ExecuteSQL processor](./nifi_12.png)

## 6. Write to a table using MergeRecord and PutDatabaseRecord processor

1. To write multiple rows in a single insert, we first need to merge multiple records into a single record. This can be done using the MergeRecord processor

2. Under the "Properties” section of the MergeRecord processor, input the following values

    | Property                  | Value             | Remark                                                                                                                          |
    |---------------------------|-------------------|---------------------------------------------------------------------------------------------------------------------------------|
    | Record Reader             | JSONTreeReader    | Select the appropriate record reader                                                                                            |
    | Record Writer             | JSONReadSetWriter | Select the appropriate record writer                                                                                            |
    | Minimum Number of Records | 1000              | Change this to a higher number so that the minimum number of rows are merged to form a single record. Default to 1 row |
    | Maximum Number of Records | 10000             | Change this to a higher number than “Minimum Number of Records”. Default to 1,000 rows                                         |

3. To confirm that multiple records are merged into one, examine the input and output of the MergeRecord processor. Note that the output is an array of multiple input records

    Input
    ![ExecuteSQL processor](./nifi_13.png)

    Ouput
    ![ExecuteSQL processor](./nifi_14.png)

4. Under the "Properties" section of the PutDatabaseRecord processor, input the following values

    | Property                            | Value           | Remark                                                                                                                                   |
    | ----------------------------------- | --------------- | ---------------------------------------------------------------------------------------------------------------------------------------- |
    | Record Reader                       | JSONTreeReader  | Select the appropriate record reader                                                                                                     |
    | Database Type                       | Generic         | Leave as default                                                                                                                         |
    | Statement Type                      | INSERT          |                                                                                                                                          |
    | Database Connection Pooling Service | ClickHouse JDBC | Select the ClickHouse controller service                                                                                                 |
    | Table Name                          | tbl             | Input your table name here                                                                                                               |
    | Translate Field Names               | false           | Set to "false" so that field names inserted must match the column name                                                                                      |
    | Maximum Batch Size                  | 1000            | Maximum number of rows per insert. This value should not be lower than the value of “Minimum Number of Records” in MergeRecord processor |

4. To confirm that each insert contains multiple rows, check that the row count in the table is incrementing by at least the value of "Minimum Number of Records” defined in MergeRecord.

    ![ExecuteSQL processor](./nifi_15.png)

5. Congratulations - you have successfully loaded your data into ClickHouse using Apache Nifi!
