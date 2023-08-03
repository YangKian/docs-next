# Quickstart with Docker-Compose

## Requirement

For optimal performance, we suggest utilizing a Linux kernel version of 4.14 or
higher when initializing an HStreamDB Cluster.

::: tip
In the case it is not possible for the user to use a Linux kernel version of
4.14 or above, we recommend adding the option `--enable-dscp-reflection=false`
to HStore while starting the HStreamDB Cluster.
:::

## Installation

### Install docker

::: tip
If you have already installed docker, you can skip this step.
:::

See [Install Docker Engine](https://docs.docker.com/engine/install/), and
install it for your operating system. Please carefully check that you have met
all prerequisites.

Confirm that the Docker daemon is running:

```sh
docker version
```

::: tip
On Linux, Docker needs root privileges. You can also run Docker as a
non-root user, see [Post-installation steps for Linux][non-root-docker].
:::

### Install docker compose

::: tip
If you have already installed docker compose, you can skip this step.
:::

See [Install Docker Compose](https://docs.docker.com/compose/install/), and
install it for your operating system. Please carefully check that you met all
prerequisites.

```sh
docker-compose version
```

## Start HStreamDB Services

::: warning
Do NOT use this configuration in your production environment!
:::

Create a docker-compose.yaml file for docker compose, you can
[download][quick-start.yaml] or paste the following contents:

<<< @/../assets/quick-start.yaml.template{yaml-vue}

then run:

```sh
docker-compose -f quick-start.yaml up
```

If you see some thing like this, then you have a running hstream:

```txt
hserver_1    | [INFO][2021-11-22T09:15:18+0000][app/server.hs:137:3][thread#67]************************
hserver_1    | [INFO][2021-11-22T09:15:18+0000][app/server.hs:145:3][thread#67]Server started on port 6570
hserver_1    | [INFO][2021-11-22T09:15:18+0000][app/server.hs:146:3][thread#67]*************************
```

::: tip
You can also run in background:
```sh
docker-compose -f quick-start.yaml up -d
```
:::

::: tip
If you want to show logs of server, run:
```sh
docker-compose -f quick-start.yaml logs -f hserver
```
:::

## Connect HStreamDB with HSTREAM CLI

HStreamDB can be directly managed using the `hstream` command-line interface (CLI), which is included in the `hstreamdb/hstream` image.

Start an instance of `hstreamdb/hstream` using Docker:

```sh
docker run -it --rm --name some-hstream-cli --network host hstreamdb/hstream:{{ $version() }} bash
```

### Create stream

To create a stream, you can use `hstream stream create` command. Now we will create a stream with 2 shard

```sh
hstream stream create s1 --shards 2
```

```sh
+-------------+---------+----------------+-------------+
| Stream Name | Replica | Retention Time | Shard Count |
+-------------+---------+----------------+-------------+
| s1          | 1       | 604800 seconds | 2           |
+-------------+---------+----------------+-------------+
```

### Write data to streams

The `hstream stream append` command can be used to write data to a stream.

```sh
hstream stream append s1 -p "{\"name\":\"Joe\",\"age\":12}" -p "{\"name\":\"Tom\",\"age\":13}" -j -k "key1"
hstream stream append s1 -p "{\"name\":\"Ann\",\"age\":15}" -j
```

- The `-p` option is used to specify the data to be written. Multiple data can be written simultaneously, these data will be dispatched to HStreamDB in a single batch.
- The `-j` option indicates that you are writing HRecords.
- The `-k` option is used to specify a key for the record.

For additional information, you can use `hstream stream append -h`.

### Read data from a stream

To read data from a particular stream, the `hstream stream read-stream` command is used.

```sh
hstream stream read-stream s1
```

```sh
timestamp: "1691055754068", id: 1899972067469254-8589934594-0, key: "key1", record: {"age":12.0,"name":"Joe"}
timestamp: "1691055754068", id: 1899972067469254-8589934594-1, key: "key1", record: {"age":13.0,"name":"Tom"}
timestamp: "1691057010982", id: 1899972067469254-8589934595-0, key: "", record: {"age":15.0,"name":"Ann"}
timestamp: "1691057030386", id: 1899972067469254-8589934596-0, key: "", record: {"age":25.0,"name":"Kite"}
```

You can also set a read offset, which can be one of the following types：

- `earliest`: This seeks to the first record of the stream.
- `latest`: This seeks to the end of the stream.
- `timestamp`: This seeks to a record with a specific creation timestamp.

For instance:

```sh
hstream stream read-stream s1 --from 1691057010982 --total 1
```

```sh
timestamp: "1691057010982", id: 1899972067469254-8589934595-0, key: "", record: {"age":15.0,"name":"Ann"}
```

## Start HStreamDB's interactive SQL CLI

```sh-vue
docker run -it --rm --name some-hstream-cli --network host hstreamdb/hstream:{{ $version() }} hstream --port 6570 sql
```

If everything works fine, you will enter an interactive CLI and see help
information like

```txt
      __  _________________  _________    __  ___
     / / / / ___/_  __/ __ \/ ____/   |  /  |/  /
    / /_/ /\__ \ / / / /_/ / __/ / /| | / /|_/ /
   / __  /___/ // / / _, _/ /___/ ___ |/ /  / /
  /_/ /_//____//_/ /_/ |_/_____/_/  |_/_/  /_/

Command
  :h                           To show these help info
  :q                           To exit command line interface
  :help [sql_operation]        To show full usage of sql statement

SQL STATEMENTS:
  To create a simplest stream:
    CREATE STREAM stream_name;

  To create a query select all fields from a stream:
    SELECT * FROM stream_name EMIT CHANGES;

  To insert values to a stream:
    INSERT INTO stream_name (field1, field2) VALUES (1, 2);

>
```

## Create a stream

What we are going to do first is create a stream by `CREATE STREAM` statement.

```sql
CREATE STREAM demo;
```

## Run a continuous query over the stream

Now we can run a continuous query over the stream we just created by `SELECT`
query.

The query will output all records from the `demo` stream whose humidity is above
70 percent.

```sql
SELECT * FROM demo WHERE humidity > 70 EMIT CHANGES;
```

It seems that nothing happened. But do not worry because there is no data in the
stream now. Next, we will fill the stream with some data so the query can
produce output we want.

## Start another CLI session

Start another CLI session, this CLI will be used for inserting data into the
stream.

```sh
docker exec -it some-hstream-cli hstream --port 6570 sql
```

## Insert data into the stream

Run each of the given `INSERT` statement in the new CLI session and keep an eye
on the CLI session created in (2).

```sql
INSERT INTO demo (temperature, humidity) VALUES (22, 80);
INSERT INTO demo (temperature, humidity) VALUES (15, 20);
INSERT INTO demo (temperature, humidity) VALUES (31, 76);
INSERT INTO demo (temperature, humidity) VALUES ( 5, 45);
INSERT INTO demo (temperature, humidity) VALUES (27, 82);
INSERT INTO demo (temperature, humidity) VALUES (28, 86);
```

If everything works fine, the continuous query will output matching records in
real time:

```json
{"humidity":{"$numberLong":"80"},"temperature":{"$numberLong":"22"}}
{"humidity":{"$numberLong":"76"},"temperature":{"$numberLong":"31"}}
{"humidity":{"$numberLong":"82"},"temperature":{"$numberLong":"27"}}
{"humidity":{"$numberLong":"86"},"temperature":{"$numberLong":"28"}}
```

[non-root-docker]: https://docs.docker.com/engine/install/linux-postinstall/#manage-docker-as-a-non-root-user
[quick-start.yaml]: https://raw.githubusercontent.com/hstreamdb/docs-next/main/assets/quick-start.yaml
