# Redshift output plugins for Embulk

Redshift output plugins for Embulk loads records to Redshift.

## Overview

* **Plugin type**: output
* **Load all or nothing**: depnds on the mode. see below.
* **Resume supported**: depnds on the mode. see below.

## Configuration

- **host**: database host name (string, required)
- **port**: database port number (integer, default: 5439)
- **user**: database login user name (string, required)
- **password**: database login password (string, default: "")
- **database**: destination database name (string, required)
- **schema**: destination schema name (string, default: "public")
- **table**: destination table name (string, required)
- **access_key_id**: access key id for AWS
- **secret_access_key**: secret access key for AWS
- **iam_user_name**: IAM user name for uploading temporary files to S3. The user should have permissions of `s3:GetObject`, `s3:PutObject`, `s3:DeleteObject`, , `s3:ListBucket` and `sts:GetFederationToken`. (string, default: "", but we strongly recommend that you use IAM user for security reasons. see below.)
- **s3_bucket**: S3 bucket name for temporary files
- **s3_key_prefix**: S3 key prefix for temporary files (string, default:"")
- **options**: extra connection properties (hash, default: {})
- **mode**: "insert", "insert_direct", "truncate_insert", or "replace". See below. (string, required)
- **batch_size**: size of a single batch insert (integer, default: 16777216)
- **default_timezone**: If input column type (embulk type) is timestamp, this plugin needs to format the timestamp into a SQL string. This default_timezone option is used to control the timezone. You can overwrite timezone for each columns using column_options option. (string, default: `UTC`)
- **column_options**: advanced: a key-value pairs where key is a column name and value is options for the column.
  - **type**: type of a column when this plugin creates new tables (e.g. `VARCHAR(255)`, `INTEGER NOT NULL UNIQUE`). This used when this plugin creates intermediate tables (insert, truncate_insert and merge modes), when it creates the target table (insert_direct and replace modes), and when it creates nonexistent target table automatically. (string, default: depends on input column type. `BIGINT` if input column type is long, `BOOLEAN` if boolean, `DOUBLE PRECISION` if double, `CLOB` if string, `TIMESTAMP` if timestamp)
  - **value_type**: This plugin converts input column type (embulk type) into a database type to build a INSERT statement. This value_type option controls the type of the value in a INSERT statement. (string, default: depends on the sql type of the column. Available values options are: `byte`, `short`, `int`, `long`, `double`, `float`, `boolean`, `string`, `nstring`, `date`, `time`, `timestamp`, `decimal`, `json`, `null`, `pass`)
  - **timestamp_format**: If input column type (embulk type) is timestamp and value_type is `string` or `nstring`, this plugin needs to format the timestamp value into a string. This timestamp_format option is used to control the format of the timestamp. (string, default: `%Y-%m-%d %H:%M:%S.%6N`)
  - **timezone**: If input column type (embulk type) is timestamp, this plugin needs to format the timestamp value into a SQL string. In this cases, this timezone option is used to control the timezone. (string, value of default_timezone option is used by default)


### Modes

* **insert**:
  * Behavior: This mode writes rows to some intermediate tables first. If all those tasks run correctly, runs `INSERT INTO <target_table> SELECT * FROM <intermediate_table_1> UNION ALL SELECT * FROM <intermediate_table_2> UNION ALL ...` query. If the target table doesn't exist, it is created automatically.
  * Transactional: Yes. This mode successfully writes all rows, or fails with writing zero rows.
  * Resumable: Yes.
* **insert_direct**:
  * Behavior: This mode inserts rows to the target table directly. If the target table doesn't exist, it is created automatically.
  * Transactional: No. If fails, the target table could have some rows inserted.
  * Resumable: No.
* **truncate_insert**:
  * Behavior: Same with `insert` mode excepting that it truncates the target table right before the last `INSERT ...` query.
  * Transactional: Yes.
  * Resumable: Yes.
* **replace**:
  * Behavior: This mode writes rows to an intermediate table first. If all those tasks run correctly, drops the target table and alters the name of the intermediate table into the target table name.
  * Transactional: Yes.
  * Resumable: No.

### Supported types

|database type|default value_type|note|
|:--|:--|:--|
|bool|boolean||
|smallint|short||
|int|int||
|bigint|long||
|real|float||
|double precision|double||
|numeric|decimal||
|char|string||
|varchar|string||
|date|date||
|timestamp|timestamp||

You can use other types by specifying `value_type` in `column_options`.

### Example

```yaml
out:
  type: redshift
  host: myinstance.us-west-2.redshift.amazonaws.com
  user: pg
  password: ""
  database: my_database
  table: my_table
  access_key_id: ABCXYZ123ABCXYZ123
  secret_access_key: AbCxYz123aBcXyZ123
  iam_user_name: my-s3-read-only
  s3_bucket: my-redshift-transfer-bucket
  s3_key_prefix: temp/redshift
  mode: insert
```

Advanced configuration:

```yaml
out:
  type: redshift
  host: myinstance.us-west-2.redshift.amazonaws.com
  user: pg
  password: ""
  database: my_database
  table: my_table
  access_key_id: ABCXYZ123ABCXYZ123
  secret_access_key: AbCxYz123aBcXyZ123
  iam_user_name: my-s3-read-only
  s3_bucket: my-redshift-transfer-bucket
  s3_key_prefix: temp/redshift
  options: {loglevel: 2}
  mode: insert_direct
  column_options:
    my_col_1: {type: 'VARCHAR(255)'}
    my_col_3: {type: 'INT NOT NULL'}
    my_col_4: {value_type: string, timestamp_format: `%Y-%m-%d %H:%M:%S %z`, timezone: '-0700'}
    my_col_5: {type: 'DECIMAL(18,9)', value_type: pass}
```

### Build

```
$ ./gradlew gem
```

### Security
This plugin requires AWS access credentials so that it may write temporary files to S3. There are two security options, Standard and Federated.
To use Standard security, give **aws_key_id** and **secret_access_key**. To use Federated mode, also give the **iam_user_name** field.
Federated mode really means temporary credentials, so that a man-in-the-middle attack will see AWS credentials that are only valid for 1 calendar day after the transaction.