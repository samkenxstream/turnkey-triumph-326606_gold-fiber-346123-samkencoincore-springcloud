## BigQuery

[Google Cloud BigQuery](https://cloud.google.com/bigquery) is a fully
managed, petabyte scale, low cost analytics data warehouse.

Spring Framework on Google Cloud provides:

  - A convenience starter which provides autoconfiguration for the
    [`BigQuery`](https://googleapis.dev/java/google-cloud-clients/latest/com/google/cloud/bigquery/BigQuery.html)
    client objects with credentials needed to interface with BigQuery.

  - A Spring Integration message handler for loading data into BigQuery
    tables in your Spring integration pipelines.

Maven coordinates,
using [Spring Framework on Google Cloud BOM](getting-started.xml#bill-of-materials):

``` xml
<dependency>
    <groupId>com.google.cloud</groupId>
    <artifactId>spring-cloud-gcp-starter-bigquery</artifactId>
</dependency>
```

Gradle coordinates:

    dependencies {
        implementation("com.google.cloud:spring-cloud-gcp-starter-bigquery")
    }

### Configuration

The following application properties may be configured with Spring
Framework on Google Cloud BigQuery libraries.

|                                                  |                                                                                                                                                                                                                  |          |                                                                                                                                                                                                                |
| ------------------------------------------------ |------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------| -------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Name                                             | Description                                                                                                                                                                                                      | Required | Default value                                                                                                                                                                                                  |
| `spring.cloud.gcp.bigquery.datasetName`          | The BigQuery dataset that the `BigQueryTemplate` and `BigQueryFileMessageHandler` is scoped to.                                                                                                                  | Yes      |                                                                                                                                                                                                                |
| `spring.cloud.gcp.bigquery.enabled`              | Enables or disables Spring Framework on Google Cloud BigQuery autoconfiguration.                                                                                                                                                 | No       | `true`                                                                                                                                                                                                         |
| `spring.cloud.gcp.bigquery.project-id`           | Google Cloud project ID of the project using BigQuery APIs, if different from the one in the [Spring Framework on Google Cloud Core Module](#spring-framework-on-google-cloud-core).                                                                      | No       | Project ID is typically inferred from [`gcloud`](https://cloud.google.com/sdk/gcloud/reference/config/set) configuration.                                                                                      |
| `spring.cloud.gcp.bigquery.credentials.location` | Credentials file location for authenticating with the Google Cloud BigQuery APIs, if different from the ones in the [Spring Framework on Google Cloud Core Module](#spring-framework-on-google-cloud-core)                                       | No       | Inferred from [Application Default Credentials](https://cloud.google.com/docs/authentication/production), typically set by [`gcloud`](https://cloud.google.com/sdk/gcloud/reference/auth/application-default). |
| `spring.cloud.gcp.bigquery.jsonWriterBatchSize` | Batch size which will be used by `BigQueryJsonDataWriter` while using [BigQuery Storage Write API](https://cloud.google.com/bigquery/docs/write-api). Note too large or too low values might impact performance. | No | 1000 |
| `spring.cloud.gcp.bigquery.threadPoolSize` | The size of thread pool of `ThreadPoolTaskScheduler` which is used by `BigQueryTemplate`                                                                                                                         | No | 4 |

#### BigQuery Client Object

The `GcpBigQueryAutoConfiguration` class configures an instance of
[`BigQuery`](https://googleapis.dev/java/google-cloud-clients/latest/com/google/cloud/bigquery/BigQuery.html)
for you by inferring your credentials and Project ID from the machine’s
environment.

Example usage:

``` java
// BigQuery client object provided by our autoconfiguration.
@Autowired
BigQuery bigquery;

public void runQuery() throws InterruptedException {
  String query = "SELECT column FROM table;";
  QueryJobConfiguration queryConfig =
      QueryJobConfiguration.newBuilder(query).build();

  // Run the query using the BigQuery object
  for (FieldValueList row : bigquery.query(queryConfig).iterateAll()) {
    for (FieldValue val : row) {
      System.out.println(val);
    }
  }
}
```

This object is used to interface with all BigQuery services. For more
information, see the [BigQuery Client Library usage
examples](https://cloud.google.com/bigquery/docs/reference/libraries#using_the_client_library).

#### BigQueryTemplate

The `BigQueryTemplate` class is a wrapper over the `BigQuery` client
object and makes it easier to load data into BigQuery tables. A
`BigQueryTemplate` is scoped to a single dataset. The autoconfigured
`BigQueryTemplate` instance will use the dataset provided through the
property `spring.cloud.gcp.bigquery.datasetName`.

Below is a code snippet of how to load a CSV data `InputStream` to a
BigQuery table.

``` java
// BigQuery client object provided by our autoconfiguration.
@Autowired
BigQueryTemplate bigQueryTemplate;

public void loadData(InputStream dataInputStream, String tableName) {
  CompletableFuture<Job> bigQueryJobFuture =
      bigQueryTemplate.writeDataToTable(
          tableName,
          dataFile.getInputStream(),
          FormatOptions.csv());

  // After the future is complete, the data is successfully loaded.
  Job job = bigQueryJobFuture.get();
}
```

Below is a code snippet of how to load a [newline-delimited JSON](https://cloud.google.com/bigquery/docs/loading-data-cloud-storage-json) data `InputStream` to a BigQuery table. This implementation uses the  [BigQuery Storage Write API](https://cloud.google.com/bigquery/docs/write-api).
[Here](https://github.com/GoogleCloudPlatform/spring-cloud-gcp/tree/main/spring-cloud-gcp-bigquery/src/test/resources/data.json) is a sample newline-delimited JSON file which can be used for testing this functionality.

``` java
// BigQuery client object provided by our autoconfiguration.
@Autowired
BigQueryTemplate bigQueryTemplate;

  /**
   * This method loads the InputStream of the newline-delimited JSON records to be written in the given table.
   * @param tableName name of the table where the data is expected to be written
   * @param jsonInputStream InputStream of the newline-delimited JSON records to be written in the given table
   */
  public void loadJsonStream(String tableName, InputStream jsonInputStream)
      throws ExecutionException, InterruptedException {
    CompletableFuture<WriteApiResponse> writeApFuture =
        bigQueryTemplate.writeJsonStream(tableName, jsonInputStream);
    WriteApiResponse apiRes = writeApFuture.get();//get the WriteApiResponse
    if (!apiRes.isSuccessful()){
      List<StorageError> errors = apiRes.getErrors();
      // TODO(developer): process the List of StorageError
    }
    // else the write process has been successful
  }
```
Below is a code snippet of how to create table and then load a [newline-delimited JSON](https://cloud.google.com/bigquery/docs/loading-data-cloud-storage-json) data `InputStream` to a BigQuery table. This implementation uses the [BigQuery Storage Write API](https://cloud.google.com/bigquery/docs/write-api).
[Here](https://github.com/GoogleCloudPlatform/spring-cloud-gcp/tree/main/spring-cloud-gcp-bigquery/src/test/resources/data.json) is a sample newline-delimited JSON file which can be used for testing this functionality.

``` java
// BigQuery client object provided by our autoconfiguration.
@Autowired
BigQueryTemplate bigQueryTemplate;

  /**
   * This method created a table with the given name and schema and then loads the InputStream of the newline-delimited JSON records in it.
   * @param tableName name of the table where the data is expected to be written
   * @param jsonInputStream InputStream of the newline-delimited JSON records to be written in the given table
   * @param tableSchema Schema of the table which is required to be created
   */
  public void createTableAndloadJsonStream(String tableName, InputStream jsonInputStream, Schema tableSchema)
      throws ExecutionException, InterruptedException {
    CompletableFuture<WriteApiResponse> writeApFuture =
        bigQueryTemplate.writeJsonStream(tableName, jsonInputStream, tableSchema);//using the overloaded method which created the table when tableSchema is passed
    WriteApiResponse apiRes = writeApFuture.get();//get the WriteApiResponse
    if (!apiRes.isSuccessful()){
      List<StorageError> errors = apiRes.getErrors();
      // TODO(developer): process the List of StorageError
    }
    // else the write process has been successful
  }
```

### Spring Integration

Spring Framework on Google Cloud BigQuery also provides a Spring Integration message
handler `BigQueryFileMessageHandler`. This is useful for incorporating
BigQuery data loading operations in a Spring Integration pipeline.

Below is an example configuring a `ServiceActivator` bean using the
`BigQueryFileMessageHandler`.

``` java
@Bean
public DirectChannel bigQueryWriteDataChannel() {
  return new DirectChannel();
}

@Bean
public DirectChannel bigQueryJobReplyChannel() {
  return new DirectChannel();
}

@Bean
@ServiceActivator(inputChannel = "bigQueryWriteDataChannel")
public MessageHandler messageSender(BigQueryTemplate bigQueryTemplate) {
  BigQueryFileMessageHandler messageHandler = new BigQueryFileMessageHandler(bigQueryTemplate);
  messageHandler.setFormatOptions(FormatOptions.csv());
  messageHandler.setOutputChannel(bigQueryJobReplyChannel());
  return messageHandler;
}
```

#### BigQuery Message Handling

The `BigQueryFileMessageHandler` accepts the following message payload
types for loading into BigQuery: `java.io.File`, `byte[]`,
`org.springframework.core.io.Resource`, and `java.io.InputStream`. The
message payload will be streamed and written to the BigQuery table you
specify.

By default, the `BigQueryFileMessageHandler` is configured to read the
headers of the messages it receives to determine how to load the data.
The headers are specified by the class `BigQuerySpringMessageHeaders`
and summarized below.

|                                               |                                                                        |
| --------------------------------------------- | ---------------------------------------------------------------------- |
| Header                                        | Description                                                            |
| `BigQuerySpringMessageHeaders.TABLE_NAME`     | Specifies the BigQuery table within your dataset to write to.          |
| `BigQuerySpringMessageHeaders.FORMAT_OPTIONS` | Describes the data format of your data to load (i.e. CSV, JSON, etc.). |

Alternatively, you may omit these headers and explicitly set the table
name or format options by calling `setTableName(…​)` and
`setFormatOptions(…​)`.

#### BigQuery Message Reply

After the `BigQueryFileMessageHandler` processes a message to load data
to your BigQuery table, it will respond with a `Job` on the reply
channel. The [Job
object](https://googleapis.dev/java/google-cloud-clients/latest/index.html?com/google/cloud/bigquery/package-summary.html)
provides metadata and information about the load file operation.

By default, the `BigQueryFileMessageHandler` is run in asynchronous
mode, with `setSync(false)`, and it will reply with a
`CompletableFuture<Job>` on the reply channel. The future is tied to the
status of the data loading job and will complete when the job completes.

If the handler is run in synchronous mode with `setSync(true)`, then the
handler will block on the completion of the loading job and block until
it is complete.

<div class="note">

If you decide to use Spring Integration Gateways and you wish to receive
`CompletableFuture<Job>` as a reply object in the Gateway, you will have
to call `.setAsyncExecutor(null)` on your `GatewayProxyFactoryBean`.
This is needed to indicate that you wish to reply on the built-in async
support rather than rely on async handling of the gateway.

</div>

### Sample

A BigQuery [sample
application](https://github.com/GoogleCloudPlatform/spring-cloud-gcp/tree/main/spring-cloud-gcp-samples/spring-cloud-gcp-bigquery-sample)
is available.
