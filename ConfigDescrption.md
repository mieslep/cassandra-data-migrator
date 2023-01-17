# Worked Example Configuration
The intent of this document is to provide a more verbose description of how to configure a migration `.properties` file. It is based on `sparkConf.properties` [found here](./src/resources/sparkConf.properties).

**Table of Contents**
1. [Connecting](#Connecting)
2. [Table to Migrate - Basic Configuration](#table-to-migrate---basic-configuration)
3. [Throughput - Basic Configuration](#throughput---basic-configuration)
4. [Errors and Retries](#errors-and-retries)

## Connecting
Note that in both Origin and Target connection specifications, one can specify either `.host` or `.scb`:

* `.host` is a standard Cassandra cluster, and is in the format of `<hostname>:<port>`; if `:<port>` is not specified, it defaults to `9042`
* `.scb` is a [Datastax Astra](https://astra.datastax.com) Secure Connect Bundle.

The example configuration assumes a Cassandra Source, and an Astra Target.

As the Migrator is Java-based, if you are using TLS for your Source or Target you will need to specify Java Keystore and Truststore information as appropriate; note you may also need to explicitly specify a TLS port for connections, depending on your configuration.

### Connection to Origin
The Origin cluster contains the data to be migrated. Relevant configuration to establish the connection should be somewhat self-evident:
```
spark.origin.host                                 localhost
spark.origin.username                             some-username
spark.origin.password                             some-secret-password

########################## ONLY USE if SSL clientAuth is enabled on origin Cassandra/DSE ####################
#spark.origin.trustStore.path
#spark.origin.trustStore.password
#spark.origin.trustStore.type                     JKS
#spark.origin.keyStore.path
#spark.origin.keyStore.password
#spark.origin.enabledAlgorithms                   TLS_RSA_WITH_AES_128_CBC_SHA,TLS_RSA_WITH_AES_256_CBC_SHA
```

### Connection to Target
The Target cluster is that which should receive the migration. The configuration is similar to the Source configuration, with the addition of 
```
spark.target.scb                                  file:///aaa/bbb/secure-connect-enterprise.zip
spark.target.username                             client-id
spark.target.password                             client-secret

####################### ONLY USE if SSL clientAuth is enabled on target Cassandra/DSE #######################
#spark.target.trustStore.path
#spark.target.trustStore.password
#spark.target.trustStore.type                     JKS
#spark.target.keyStore.path
#spark.target.keyStore.password
#spark.target.enabledAlgorithms                   TLS_RSA_WITH_AES_128_CBC_SHA,TLS_RSA_WITH_AES_256_CBC_SHA
```

## Table to Migrate - Basic Configuration
The table to migrate is specified in a few places within the configuration:
```
spark.origin.keyspaceTable                        test.a2
spark.target.keyspaceTable                        test.a2
spark.query.origin                                partition-key,clustering-key,order-date,amount
spark.query.origin.partitionKey                   partition-key
spark.query.target.id                             partition-key,clustering-key
spark.query.types                                 9,1,4,3
spark.query.ttl.cols                              2,3
spark.query.writetime.cols                        2,3
```

1. The keyspace and table should be self-explanatory; they are both typically set to the same value.
   1. A valid use-case is to have origin and target be the same database, and create a copy (perhaps filtered) of the original table.
2. `spark.query.origin` is the list of column names to be selected
3. `spark.query.origin.partitionKey` is the list of column name(s) which comprise the Partition Key. For example, if the table is defined `PRIMRY KEY (partition-key,clustering-key)`, the value should be `partition-key`, but if it is defined `PRIMARY KEY ((partition-key,partition-subkey), clustering-key)`, the value should be `partition-key,partition-subkey`.
4. `spark.query.target.id` defines the complete primary key columns in the target database; it is the same as the complete list of columns in the `PRIMARY KEY` defintion. In our second example on `partitionKey`, the value here should be `partition-key,partition-subkey,clustering-key`.
5. `spark.query.types` corresponds with the list of types found at the bottom of the `.properties` example document. You need to have as many elements in the list here as there are on the `spark.query.origin` list.
6. `spark.query.ttl.cols` and `spark.query.writetime.cols` are the zero-based indexes of the `spark.query.origin` from which the new Target row-level TTL and Writetime values are derived.
   1. The column-level values cannot be preserved; if multiple columns are specified, the larger/later value will be used.
   2. If these are left blank, the write timestamp of the record will be the current time, and the TTL will be the target table's default TTL.
   3. The tool does not preserve TTL and write times at the field level.

## Throughput - Basic Configuration
The table to migrate is specified in a few places within the configuration:
```
spark.readRateLimit                               20000
spark.writeRateLimit                              20000
spark.splitSize                                   10000
spark.batchSize                                   1
```

1. Read and write rate limits are in records per second; they will typically be the same, unless data is being filtered, in which case the read rate can be configured higher than the write rate to achieve maximum thoroughput.
2. `spark.splitSize` dictates how many "chunks" of work the partition range will be divided into. The bigger this number, the smaller the overall chunk of work to complete...but the more chunks to complete. As a rough rule of thumb, you could aim to have the overall chunk size be in the region of 10k-100k records.
3. `spark.batchSize` should be set to `1` unless you have a clustering column that results in more than 10 records per partition. In this case, you may wish to increase this size to something like 20-50% of the number of records per partition, but probably no larger than a value of `50`. This is creating an unlogged batch operation - reducing the overall number of write operations sent to Target, but increasing the amount of coordination work that must be done to unpack that cluster. Setting this value too large can result in dramatically lower thorughput and a lot of messages like `Unlogged batch covering 14 partitions detected against table ... You should used a logged batch for atomicity, or asynchronous writes for performance.`


## Errors and Retries
The table to migrate is specified in a few places within the configuration:
```
spark.target.autocorrect.missing                  false
spark.target.autocorrect.mismatch                 false
spark.maxRetries                                  1
```

1. Autocorrect is used during [Validation](https://github.com/datastax/cassandra-data-migrator/#steps-for-data-validation) mode. It should generally be left `false`, particularly when writes are being done to both Origin and Target using a tool such as [ZDM](https://docs.datastax.com/en/astra-serverless/docs/migrate/introduction.html). 
   1. Valdation rarely finds errors, and in such dual-write circumstances they can be false errors (for example, a mutation has made it to one side but not the other).
   2. Suggested approach is to manually investigate the cause of validation failures, and if they are deemed as true errors, enable Autocorrect. 
2. `spark.maxRetries` should always be set to `1`. This will mean that any error will fail the entire chunk, rather than retry...but it also means the statistics reported at the end of the job will be accurate. Setting a value higher than 1 will initiate retries (which may fix the previous failures), however the stats reported at the end of the job wont be accurate (This is a known problem that needs to be resolved).
3. Log output will show, at the end, the ranges that have failed to run. These failed ranges can be put into a file `partitions.csv` and run by [migrating specific ranges](https://github.com/datastax/cassandra-data-migrator/#migrating-specific-partition-ranges).
   1. There will not be duplicate data created: Cassandra will resolve any 'duplicates' on read, and ultimately resolve them during compaction.
   2. The exception to the previous statement: non-frozen `list` columns: multiple inserts will add duplicates to these column types.


