# Overview

The following documentation outlines how to incorporate a JDBC Source as part of your project. This allows a user to 
construct applications that interact with a SQL Database, and supports almost all standard SQL Commands. These interactions 
include the ability to run queries against the aforementioned SQL Database, periodically poll the database, and use all of the 
returned data in the given project.

In order to incorporate this Extension Source, you will need to set up your local machine with a JDBC Driver that can connect 
to your SQL Database. Once you have done this, you will need to create the Source in the VANTIQ Modelo IDE. The documentation 
has been split into two parts, [Setting Up Your Machine](#machine) and [Setting Up Your VANTIQ Modelo IDE](#vantiq).

# Prerequisites <a name="pre" id="pre"></a>

**IMPORTANT:** Read the [Testing](#testing) section before building this project.

An understanding of the VANTIQ Extension Source SDK is assumed. Please read the [Extension Source README.md](../README.md) for more 
information.

The user must [define the JDBC Source implementation](../README.md#-defining-a-typeimplementation) in the VANTIQ Modelo IDE. For an example of the definition, 
please see the [*jdbcImpl.json*](src/test/resources/jdbcImpl.json) file located in the *src/test/resources* directory.

Additionally, an example project named *jdbcExample.zip* can be found in the *src/test/resources* directory.

*   It should be noted that this example connects to a MySQL database.
*   In order to activate the pollTime/pollQuery, simply remove the comment prepending the pollTime.

# Setting Up Your Machine <a name="machine" id="machine"></a>

## Repository Contents

*   **JDBCMain** -- The main function for the program. Connects to sources as specified in a
    configuration file.
*   **JDBCCore** -- Coordinates the connections to the database, and sends the resulting data back to VANTIQ Modelo if
    necessary.
*   **JDBCHandleConfiguration** -- Sets up the JDBC connection based on the source's configuration document, and
    initializes the queryHandler and publishHandler.
*   **JDBC** -- The class that directly interacts with the JDBC Driver, executing the query and publish requests as sent
    by the VANTIQ Modelo IDE and appropriately formatting the results.

## How to Run the Program

1.  In order to effectively use the JDBC Source, you will need to download the appropriate JDBC Driver with respect to the 
    SQL Database you are using. Once you have installed this, you will need to create an environment variable named
    `JDBC_DRIVER_LOC` that contains the location of the jar file (*i.e.* `/Users/yourName/somePath/mysql-connector-java-8.0.12.jar`)
2.  Clone this repository (vantiq-extension-sources) and navigate into `<repo location>/vantiq-extension-sources`.
3.  Run `./gradlew jdbcSource:assemble`.
4.  Navigate to `<repo location>/vantiq-extension-sources/jdbcSource/build/distributions`. The zip and tar files both contain 
    the same files, so choose whichever you prefer.
5.  Uncompress the file in the location that you would like to install the program.
6.  Run `<install location>/jdbcSource/bin/jdbcSource` with a local server.config file or specifying the [server config file](#serverConfig) as the first argument. Note that the `server.config` file can be placed in the `<install location>/jdbcSource/serverConfig/server.config` or `<install location>/jdbcSource/server.config` locations.

## Logging
To change the logging settings, edit the logging config file `<install location>/jdbcSource/src/main/resources/log4j2.xml`,
which is an [Apache Log4j configuration file.](https://logging.apache.org/log4j/2.x/manual/configuration.html). The logger 
name for each class is the class's fully qualified class name, *e.g.* "io.vantiq.extjsdk.ExtensionWebSocketClient".  

## Server Config File
(Please read the [SDK's server config documentation](../extjsdk/README.md#serverConfig) first.)

### Vantiq Options
*   **authToken**: Required. The authentication token to connect with. These can be obtained from the namespace admin.
*   **sources**: Required. A comma separated list of the sources to which you wish to connect. Any whitespace will be
    removed when read.
*   **targetServer**: Required. The Vantiq server hosting the sources.

# Setting Up Your VANTIQ Modelo IDE <a name="vantiq" id="vantiq"></a>

## Source Configuration

To set up the Source in the VANTIQ Modelo IDE, you will need to add a Source to your project. Please check the [Prerequisites]
(#pre) to make sure you have properly added a Source Definition to VANTIQ Modelo. Once this is complete, you can select JDBC
(or whatever you named your Source Definition) as the Source Type. You will then need to fill out the Source Configuration 
Document.

The Configuration document may look similar to the following example:

    {
       "vantiq": { 
          "packageRows": "true"
       },
       "jdbcConfig": {
          "general": {
             "username": "sqlUsername",
             "password": "sqlPassword",
             "dbURL": "jdbc:mysql://localhost/myDB?useSSL=false&serverTimezone=UTC",
             "asynchronousProcessing": true,
             "maxActiveTasks": 10,
             "maxQueuedTasks" 20,
             "pollTime": 3000,
             "pollQuery": "SELECT * FROM myTable"
          }
       }
    }

### Options Available for vantiq
*   **packageRows**: Required. This value **MUST** be set to "true". In future versions of the JDBC Source, setting this value 
    to "false" will allow the data to be processed as a stream of rows, instead of an array containing all the rows.

### Options Available for jdbcConfig
*   **username**: Required. This is the username that will be used to connect to the SQL Database.
*   **password**: Required. This is the password that will be used to connect to the SQL Database.
*   **dbURL**: Required. This is the URL corresponding to the SQL Database that you will connect to.
*   **asynchronousProcessing**: Optional. If set to `true`, query and publish requests will be handled asynchronously, as 
opposed to the default behavior which is synchronous.
*   **maxActiveTasks**: Optional. Only used if `asynchronousProcessing` is set to `true`. The maximum number of threads 
running at any given point for query or publish requests, respectively. Must be a positive integer. Default value is 5.
*   **maxQueuedTasks**: Optional. Only used if `asynchronousProcessing` is set to `true`. The maximum number of queued 
tasks at any given point for query or publish requests, respectively. Must be a positive integer. Default value is 10.
*   **pollTime**: Optional. If specified, you must specify the pollQuery as well. This option allows you to specify a polling 
    rate indicating the frequency (in milliseconds) at which the pollQuery will be executed. The value must be a positive
    number greater than 0, (*i.e.* 3000 --> executing every 3 seconds).
*   **pollQuery**: Optional. If specified, you must specify the pollTime as well. This option indicates the SQL Query that
    will be executed by the JDBC Source, (frequency assigned by the pollTime). The SQL Query must be a **SELECT** statement,
    and the returned data will be sent as a Notification to the source. The data can be captured by creating a Rule in the
    VANTIQ Modelo IDE, as in the following example:
    
    ```
    RULE checkSourceNotification
    WHEN EVENT OCCURS ON "/sources/JDBC1" as sourceEvent
    
    // Grab the actual value from the event
    var message = sourceEvent.value
    
    try {  
        // Creating a map of each row, used to INSERT into our type
        var resultData = {}
        resultData.id = message.id
        resultData.age = message.age
        resultData.first = message.first
        resultData.last = message.last

        // Inserting the map into our JDBCType
        INSERT JDBCType(resultData)
    } catch (error) {
        // Catching any errors and throwing the exception.
        exception(error.code, error.message)
    }
    ```

## Messages from the Source

Messages that are sent to the source as Notifications from the pollQuery are JSON objects in the following format:
```
{
   {columnName:columnValue, columnName:columnValue, etc..}
}
```
The data is formatted as a HashMap which represents a row of data. Each map is a series of key-value pairs with the keys 
being the column name and the values being the column value. If multiple rows of data are returned by the pollQuery, each row
will be sent as a unique Notification. 

*   (**NOTE:** Multiple rows of data sent as Noticfications will be returned in a random order.)

## Select Statements

In order to interact with the JDBC Source, one option is to use VAIL to select from the source. To do this, you will need 
to specify the SQL Query you wish to execute against your database as part of the WITH clause. *Typically*, the SQL Queries 
used here would be **SELECT STATEMENTS**. The data will be returned to VANTIQ as a set of messages, where each message 
contains some number of rows.

However, the SQL Queries here can also be any valid SQL Update Statement (i.e. `CREATE`, `INSERT`, `DELETE`, `DROP`, etc...). 
This feature mimics the behavior of the Publish Statements, which are defined [below](#publish). In this case, the VAIL Select 
Statements will return an empty map if the statement was executed successfully, or a query error otherwise.

A query parameter named `bundleFactor` determines how many rows are bundled into each message. Generally, the default value of 
500 will be fine. However, in cases where rows may be very large, a smaller value may be required.

The `bundleFactor` parameter is specified in the `WITH` clause, allowing each query to adjust as appropriate. If the parameter 
is not specified, the default value (500) is used -- that will be fine for most cases. If a value of `0` is provided, then all 
rows will be placed into a single message (the use of this is discouraged as it may cause problems). If the value is positive 
(and non-zero), then that many rows will be sent at a time.

From the perspective of consuming the rows, there is no visible difference here. The `bundleFactor` parameter is present to 
allow control when returning very large rows.

The following example uses a Vail Select Statement to **query** a database:
```
PROCEDURE queryJDBC()

try {        
    // Normal SELECT Statement in VAIL, but using WITH Clause is important.
    SELECT * FROM SOURCE JDBC1 AS results WITH
    // WITH Clause specifies the query, whose value is the SQL Query.
    // This parameter must be named 'query'.
    query: "SELECT id, first, last, age FROM Test",
    bundleFactor: 500

    {
        // Iterating over each row of data in 'results'
        FOR (row in results) {
            // Creating a map of each row, used to INSERT into our type
    	    var resultData = {}
    	    resultData.id = row.id
    	    resultData.age = row.age
    	    resultData.first = row.first
    	    resultData.last = row.last
            
	    // Inserting the map into our JDBCType
    	    INSERT JDBCType(resultData)
        }
    }
} catch (error) {
    // Catching any errors and throwing the exception.
    exception(error.code, error.message)
}
```

The following example uses a Vail Select Statement to **update** a database:
```
try {
    // The SQL Statement that the JDBC Source will execute
    var sqlQuery = "create table Test(id int not null, age int not null, first varchar (255), last varchar (255));"
    
    // Normal SELECT Statement in VAIL, passing the 'sqlQuery' as a parameter in the 
    // 'query' field. The field must be named 'query'.
    SELECT * FROM SOURCE JDBC1 AS results WITH 
    query:sqlQuery,
    
    {
    	log.info("The result is an empty map: " + results.toString())
    }
} catch (error) {
    // Catching any errors and throwing the exception.
    exception(error.code, error.message)
}
```

## Publish Statements <a name="publish" id="publish"></a>

Another method to interact with the JDBC Source is to use VAIL to publish to the source. To do this, you will need to
specify the SQL Query you wish to execute against your database as part of the Publish Parameters. The SQL Queries used here 
can be **CREATE**, **INSERT**, **DELETE**, **DROP**, or other commands supported by your SQL database. The SQL Query must 
either be represented as a String, or as a List of Strings. If the query is represented as a list, then all of the queries in 
the list will be executed together as a batch. The following are examples of Procedures created in VANTIQ Modelo publishing to 
a JDBC source.

**Creating a table:**

```
PROCEDURE createTable()

try {
    // The SQL Statement that the JDBC Source will execute
    var sqlQuery = "create table Test(id int not null, age int not null, first varchar (255), last varchar (255));"
    
    // Normal PUBLISH Statement in VAIL, passing the 'sqlQuery' as a parameter in the 
    // 'query' field. The field must be named 'query'.
    PUBLISH {"query":sqlQuery} to SOURCE JDBC1
} catch (error) {
    // Catching any errors and throwing the exception.
    exception(error.code, error.message)
}
```

**Inserting to table:**

```
PROCEDURE insertJDBC()

try {
    // The for-loop is used to insert multiple rows of data
    FOR i in range(0, 5) {
       	// Creating the values that will be inserted
        var id = i.toString()
	var age = (20+i).toString()
	var first = "Firstname" + i.toString()
	var last = "Lastname" + i.toString()

	// The SQL Statement that the JDBC Source will execute
	// Notice that when inserting Strings, the values must be surrounded by ''. This 
	// can be seen around the 'first' and 'last' values.
        var sqlQuery = "INSERT INTO Test VALUES (" + id + ", " + age + ", '" + first + "', '" + last + "');"

        // Normal PUBLISH Statement in VAIL, passing the 'sqlQuery' as a parameter in the 
        // 'query' field. The field must be named 'query'.
        PUBLISH {"query":sqlQuery} to SOURCE JDBC1
    }
} catch (error) {
    // Catching any errors and throwing the exception.
    exception(error.code, error.message)
}
```

**Inserting to table as a batch:**

```
PROCEDURE insertBatchJDBC()

try {
    var myList = []
    
    // The for-loop is used to insert multiple rows of data
    FOR i in range(0, 5) {
    	// Creating the values that will be inserted
        var id = i.toString()
	var age = (20+i).toString()
	var first = "Firstname" + i.toString()
	var last = "Lastname" + i.toString()

	// The SQL Statement that the JDBC Source will execute
	// Notice that when inserting Strings, the values must be surrounded by ''. This 
	// can be seen around the 'first' and 'last' values.
        var sqlQuery = "INSERT INTO Test VALUES (" + id + ", " + age + ", '" + first + "', '" + last + "');"
	
	// Adding the SQL Statement to the list.
	push(myList, sqlQuery)
    }
    
    // Using a VAIL PUBLISH statement with the list as our 'query' parameter
    PUBLISH {query:myList} to SOURCE JDBC1
} catch (error) {
    // Catching any errors and throwing the exception.
    exception(error.code, error.message)
}
```

## Error Messages

Query errors originating from the source will always have the code be the fully-qualified class name with a small descriptor 
attached, and the message will include the exception causing it and the request that spawned it.

The exception thrown by the JDBC Class will always be a VantiqSQLException. This is a wrapper around the traditional 
SQLException, and contains the Error Message, SQL State, and Error Code from the original SQLException.

## Testing <a name="testing" id="testing"></a>

In order to properly run the tests, you must create an environment variable named **JDBC\_DRIVER\_LOC** which points to the 
appropriate JDBC Driver .jar file.

Additionally, you must add properties to your _gradle.properties_ file in the _~/.gradle_ directory. These properties include 
the JDBC Database username, password, and URL. You must also add the Target VANTIQ Server URL, as well as an Authentication 
Token for that server. The Target VANTIQ Server and Auth Token will be used to create a temporary VANTIQ Source, VANTIQ Type, 
VANTIQ Topic, VANTIQ Procedure and VANTIQ Rule. They will be named _testSourceName_, _testTypeName_, _testTopicName_, 
_testProcedureName_ and _testRuleName_ respectively. These names can optionally be configured by adding 
`EntConTestSourceName`, `EntConTestTypeName`, `EntConTestTopicName`, `EntConTestProcedureName` and `EntConTestRuleName` to the 
gradle.properties file. The following shows what the gradle.properties file should look like:

```
    EntConJDBCUsername=<yourUsername>
    EntConJDBCPassword=<yourPassword>
    EntConJDBCURL=<yourURL>
    TestVantiqServer=<yourVantiqServer>
    TestAuthToken=<yourAuthToken>
    EntConTestSourceName=<yourDesiredSourceName>
    EntConTestTypeName=<yourDesiredTypeName>
    EntConTestTopicName=<yourDesiredTopicName>
    EntConTestProcedureName=<yourDesiredProcedureName>
    EntConTestRuleName=<yourDesiredRuleName>
```

* **NOTE:** We strongly encourage users to create a unique VANTIQ Namespace in order to ensure that tests do not accidentally 
override any existing Sources or Types.

## Licensing
The source code uses the [MIT License](https://opensource.org/licenses/MIT).  

HikariCP, okhttp3, log4j, and jackson-databind are licensed under
[Apache Version 2.0 License](http://www.apache.org/licenses/LICENSE-2.0).  

slf4j is licensed under the [MIT License](https://opensource.org/licenses/MIT).  
