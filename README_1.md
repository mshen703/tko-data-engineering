# Snowpipe Streaming Ingest & Dynamic Tables
## Summit 2022 Keynote Demo

This repo contains the necessary code, configurations, and SQL files to conduct Part 1 of the 2022 Summit Keynote Demo around Snowpipe Streaming Ingest and Dynamic Tables. It consists of three main components:
1. a Python client app to stream syntheticall generated ad-click data (in the form of JSON messages) into an Apache Kafka topic
2. the Snowflake Kafka Connector (KC) configurations necessary to ingest these ad records into a Snowflake table using Snowpipe Streaming
3. the SQL files necessary to both configure the permissions/roles in the Snowflake account to conduct the demo, as well as build the Dynamic Table and query it for the sake of demonstrating auto-refresh capabilities in DTs (as well as joins to existing reference data)

**Please note: as of 15 December 2022, Dynamic Tables are in PrPr. Please follow the account enablement instructions below, and visit [PrPr Guidance for DTs](https://docs.google.com/document/d/1b02Fl8i52n5OYnDkDyWutsKW0m9SBB_aBJCsgwN8wN4/edit#heading=h.bhlqg220nh4d) to understand known limitations as part of the PrPr.**

**Additionally, Dynamic Table creation will cause the corresponding dedicated virtual warehouse to ***not*** suspend, even if configured for auto-suspension, due to the warehouse needing to remain up in order to process refreshes. This is true EVEN if you are no longer ingesting streaming data. To mitigate this and ensure warehouses do not incur unintended cost, you can either:**
- Drop the materialized tables after running the demo, and then re-create them next time you need them via `drop dynamic table parsed_streaming_ad_record` and `drop dynamic table campaign_spend`. This will allow the warehouse to auto-suspend
- Set the DT lag to a value significantly greater than auto-suspension time of the warehouse (e.g. 9999 days). If you do this, you should also enable the parameter `ENABLE_MATERIALIZED_TABLE_MANUAL_REFRESH=true` which will allow you to manually trigger a refresh during a demo should you need it, via an `alter dynamic table parsed_streaming_ad_record refresh` and `alter dynamic table campaign_spend refresh` command. This will cause auto-refresh to only occur very infrequently if data arrives, and thus the warehouse will remain auto-suspended unless new data is being ingested.

**Please also note: as of 10 January 2023, Schema Detection & Evolution with Kafka Connector is in PrPr. Please follow enablement instructions below and visit [the documentation](https://docs.snowflake.com/en/LIMITEDACCESS/snowpipe-streaming-kafka-schema-detection.html) to understand more about Schema Detection & Evolution.**

### Requirements

#### For Docker (Preferred Method)
- Docker desktop **(please ensure that you have logged in to Docker Desktop, otherwise subsequent Docker commands may fail)**
- A Snowflake account with key authentication configured for the relevant user role

#### For Local Runtime (Not Preferred)
- a Conda Python environment
- open-source Apache Kafka 2.13-3.1.0 installed locally
- Snowflake Kafka Connector 1.8.1 jar
- openJDK <= 15.0.2 (there is an `arrow` bug introduced for JDK 16+ that will cause errors in the Kafka Connector, so the machine must be equipped with openjDK <=15.0.2)

We will go through the steps of installing and running Apache Kafka, configuring and launching the Kafka Connector, configuring and launching the Python streaming client, configuring your Snowflake account, and running the demo.

### Snowflake - Account Set Up
#### Account Configuration
The following parameters need to be set in the Snowflake account:
```sql
--- Enable Snowpipe Streaming
alter account <LOCATOR> set EPS3_FDN_PACK_VERSION = 8 parameter_comment='Enabling persisting Start Offsets in EP Files for Streaming Ingest PrPr';
alter account <LOCATOR> set ENABLE_UNIFIED_TABLE_SCAN = true parameter_comment='Enabling unified scansets for Streaming Ingest PrPr';
alter account <LOCATOR> set ENABLE_PR_37692_MULTI_FORMAT_SCANSET = true parameter_comment='Enabling multi-format scansets for Streaming Ingest PrPr';
alter account <LOCATOR> set ENABLE_STREAMING_INGEST = true parameter_comment = 'Enabling the Streaming Ingest feature for PrPR';
alter account <LOCATOR> set VERSION_ACCOUNT_USAGE_SNOWPIPE_STREAMING_FILE_MIGRATION_HISTORY = 0 parameter_comment = 'Enable snowpipe streaming file migration history view in account usage';

--- Enable Dynamic Tables
alter account <LOCATOR> set 
                  ENABLE_MATERIALIZED_TABLES=true,
                  STREAM_REWRITE_IN_PLANNER_BY_KIND='DYNAMIC_TABLE',
                  CHANGES_ENABLE_GROUPBY=true,
                  ENABLE_FIX_404961= 'Enable'
                  UI_ENABLE_MATERIALIZED_TABLES='enabled'
    PARAMETER_COMMENT = 'Enabling for DT PrPr';

--- Enable Schema Evolution for KC w/ Snowpipe Streaming (PrPr)
alter account <LOCATOR> SET 
                    ENABLE_SCHEMA_EVOLUTION_PRIVILEGE = true 
                    ENABLE_SCHEMA_EVOLUTION_FEATURE = true 
                    ENABLE_SCHEMA_EVOLUTION_TABLE_OPTION = true 
                    ENABLE_INGEST_SCHEMA_EVOLUTION = true
                    PARAMETER_COMMENT = 'Schema Evolution PrPr';
```
Once the params have been enabled for Snowpipe Streaming, DTs, and Schema Evolution, open up the `./setup_kafka_connector.sql` file into a Snowsight worksheet and run the SQL commands to create databases, schemas, etc. and configure the `kafk_connector_role_1` role that KC is going to use to ingest data using Snowpipe Streaming

#### Configure RSA Keys for the KCs Account
- Create keys (for the demo we will use unencrypted keys) via the instructions located [here](https://docs.snowflake.com/en/user-guide/kafka-connector-install.html#using-key-pair-authentication-key-rotation). We recommend naming the keys `sf_kafka_rsa_key.pem` and `sf_kafka_rsa_pub_key.pub` and putting them in the `./include/` directory. These will be ignored by Git.
- Add public key to user in Snowflake:
```sql
alter user <USERNAME> set rsa_public_key='copy_public_key_here';
```

### Docker Setup - Preferred Method
We have provided a method for you to build and run all of the necessary components for this demo via a single `docker-compose.yml`. This will launch four services: Zookeeper, Apache Kafka, Kafka Connect, and the Python Streaming Client. To get started with this demo via Docker:

1. Clone this repository to your local machine that has Docker Desktop installed: `git clone git@github.com:snowflakecorp/snowpipe_streaming_summit_demo.git`
2. Open up the `SF_connect.properties` file in your text editor of choice and modify the following properties with your appropriate demo account information: `snowflake.url.name`, `snowflake.user.name`, and `snowflake.private.key`. If you are using an encrypted private key, you'll also neeed to uncomment and include the `snowflake.private.key.passphrase`, however if your key is not encrypted, leave this line commented out.
3. Once you have modified and saved the properties, you can build and launch the project:
```zsh
cd /path/to/repo/
docker-compose build
docker-compose up
```
This `docker-compose up` command will launch:
1. a Zookeeper instance
2. a local Kafka cluster
3. a Kafka connect instance with the Snowflake connector configured to ingest data into Snowflake
4. a Python client to stream fake data to Kafka

### Snowflake - Viewing & Querying Data in the Demo
#### Creating the Reference Table and Dynamic Table
Load the `./setup_tables.sql` and `./snowpipe_streaming_demo.sql` files into a Snowsight worksheet and walk through the SQL commands to query the data.

---

### Local Setup - Not Preferred
### Apache Kafka `2.13-3.3.1`
#### Installation
Follow step 1 in the [Apache Kafka Quickstart](https://kafka.apache.org/quickstart) to download and install Kafka 2.13-3.3.1. You can also launch Kafka and run steps 3-5 to ensure Kafka is working properly.

We recommend downloading and extracting Kafka into a directory such as `~/dev` or `~/apache_kafka`. We will refer to this download location (e.g. `~/dev/kafka_2.13-3.1.0/`) as `<kafka_dir>` throughout this README.
#### Running Kafka
Navigate to `<kafka_dir>` and launch zookeeper:
```zsh
cd <kafka_dir>
bin/zookeeper-server-start.sh config/zookeeper.properties
```
Open up another terminal window. Navigate to `<kafka_dir>` and launch the Kafka broker service:
```zsh
$ bin/kafka-server-start.sh config/server.properties
```
Now, Kafka should be up and running. You can also run steps 3-5 in the quickstart linked above to verify that the Kafka broker is working properly.

### Kafka Connector `1.8.1`
#### Installation
Refer to the [documentation](https://docs.snowflake.com/en/user-guide/kafka-connector-install.html#download-the-kafka-connector-jar-files) for full instructions, but the general instructions are as follows:
1. Download the [`Snowflake Kafka Connector 1.8.1 JAR`](https://mvnrepository.com/artifact/com.snowflake/snowflake-kafka-connector/1.8.1) from Maven
2. Copy the JAR into `<kafka_dir>/libs`

That's it!

#### Configure RSA Keys for the KCs Account
- Create keys (for the demo we will use unencrypted keys) via the instructions located [here](https://docs.snowflake.com/en/user-guide/kafka-connector-install.html#using-key-pair-authentication-key-rotation). We recommend naming the keys `sf_kafka_rsa_key.pem` and `sf_kafka_rsa_pub_key.pub` and putting them in the `./include/` directory. These will be ignored by Git.
- Add public key to user in Snowflake:
```sql
alter user <USERNAME> set rsa_public_key='copy_public_key_here';
```
- Add private key to [`./SF_connect.properties`](./SF_connect.properties) under `snowflake.private.key=`
- **If you are using an encrypted key** you'll also need to uncomment and set the `snowflake.private.key.passphrase=` propery. If your private key is not encrypted, you must leave this property commented out.

#### Configure the KC
Modify [`./SF_connect.properties`](./SF_connect.properties) file to reflect the desired demo account information. Likely changes include:
- `snowflake.url.name`
- `snowflake.user.name`
- `snowflake.private.key`
- `snowflake.private.key.passphrase` if you are using encrypted keys

```properties
name=snowpipe_streaming_ingest
connector.class=com.snowflake.kafka.connector.SnowflakeSinkConnector
tasks.max=8
topics=digital_ad_records
snowflake.topic2table.map=digital_ad_records:streaming_ad_record
buffer.count.records=10000
buffer.flush.time=10
buffer.size.bytes=20000000
snowflake.url.name=ACCOUNT_LOCATOR.snowflakecomputing.com:443
snowflake.user.name=USER_NAME_HERE
snowflake.private.key=YOUR_KEY_HERE
# snowflake.private.key.passphrase=YOUR_PASSPHRASE_IF_USING_ENCRYPTED_KEY_HERE
snowflake.database.name=snowpipe_streaming
snowflake.schema.name=dev
key.converter=org.apache.kafka.connect.storage.StringConverter
# value.converter=com.snowflake.kafka.connector.records.SnowflakeJsonConverter
value.converter=org.apache.kafka.connect.storage.StringConverter
snowflake.role.name=kafka_connector_role_1
snowflake.ingestion.method=SNOWPIPE_STREAMING
```

Ensure that `snowflake.ingestion.method` is set to `SNOWPIPE_STREAMING` and that `key.converter` and `value.converter` are both set to `org.apache.kafka.connect.storage.StringConverter`

#### Running KC
Once your [`./SF_connect.properties`](./SF_connect.properties) file has all of the correct info, copy and paste the file into `<kafk_dir>/config`. Then, assuming that your Snowflake account has been properly enabled, you'll be able to launch the Kafka Connector via:
```zsh
<kafka_dir>/bin/connect-standalone.sh <kafka_dir>/config/connect-standalone.properties <kafka_dir>/config/SF_connect.properties
```

### Python Streaming Client
#### Environment
We are assuming that there is an existing conda virtual environment being built for the other components of the demo. The Python streaming client can run in this same virtual environment. To install any additional dependencies:
```zsh
# activate the conda environment
conda activate <env_name>
pip install -r ./requirements.txt
```
#### Configuration
There are two relevant configuration files:
1. [`./kafka_producer_config.json`](./kafka_producer_config.json) which should not need to be modified because we are running a standalone local Kafka instance
2. [`./kafka_topic_config.json`](./kafka_topic_config.json) which should ONLY be modified if you want to either (1) change the streaming message frequency or (2) backfill historical data.

To change the streaming message frequency, change the `messageFrequency` param. This param control the amount of time (in seconds) between subsequent messages being sent from the Python client to a Kafka topic. E.g. `"messageFrequency": 5` implies a message will be sent every 5 seconds, while `"messageFrequency": 0.01` means 100 msgs/sec will be written to Kafka.

To backfill historical data, you can set the `messageDate` param to the unix timestamp that you want to correspond to the date/time of the *first* record produced. So, to backfill data beginning May 1, 2012 at 12:00:01am, set `"messageDate": 1335852060`. The client will then generate messages *as fast as it can* and publish them to Kafka, producing timestamps in the data records based on the message's relation to the `messageDate` configured value. There is an example of this in the [`./kafka_topic_config_for_historical_backfill.json`](./kafka_topic_config_for_historical_backfill.json) sample file (**THIS SAMPLE FILE IS NOT USED AT RUNTIME, ONLY THERE FOR DEMONSTRATION PURPOSES**).

So, as an example, if you have `"messageFrequency": 5, "messageDate": 1335852060`, then the second message will be produced with a datetime of `1335852065`, five seconds later. This is not reflected by the *wall clock* however- the messages are produced and published as fast as the client can possibly produce them. Once the program's clock reaches the current wall-clock, the program will fall back into a real-time operating mode and stream data normally, in sync with the real-world wall clock. This provides a handy way to initialize the demo environment database with 10+ years of historical data. It will take several minutes for that much data to stream and populate, but is simpler than otherwise generating CSVs, bulk loading, etc.

If you do not need to backfill historical data, leave the `messageDate` param set to `null`

#### Running the Client
Once you've set the `./kafka_topic_config.json` params appropriately, run the client in a terminal window using your conda environment:
```zsh
conda activate <env_name>
python ./snowpipe_streaming_kafka_producer.py
```
The client will log output to the terminal every X messages, and you should see data populating in your Snowflake account from the Kafka Connector!

---