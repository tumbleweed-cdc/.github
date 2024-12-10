Tumbleweed is an open-source, user-friendly framework designed for fast and consistent data propagation between microservices using Change Data Capture (CDC) and the transactional outbox pattern.
It automatically deploys a self-hosted log-based CDC pipeline that abstracts away the complexities associated with setting up and using CDC tools. It is designed to monitor changes in one or more PostgreSQL databases and sync that data to consumer microservices in near real-time.

## Prerequisites

### Configuring PostgreSQL

To enable Tumbleweed's Change Data Capture capabilities, your source PostgreSQL database must be version 10 or later and configured to use the **logical** replication mode by setting the Write-Ahead Log (WAL) level to `logical`.

Note: The location of the `postgresql.conf` and `pg_hba.conf` files may vary depending on how PostgreSQL is installed. Additionally, these files are typically not directly accessible on managed cloud services, in which case you need to use the provider's configuration tools to modify these settings. (e.g, Parameter Groups on AWS RDS).
1. Modify the PostgreSQL Configuration File:
    * Locate the `postgresql.conf` file and update the following settings:
    ```
    wal_level = logical
    max_replication_slots = 5
    max_wal_senders = 5
    ```
   
    * `wal_level` must be set to `logical` to enable logical replication.
    * `max_replication_slots` number of maximum allowed replication slots.
    * `max_wal_senders` number of concurrent connections allowed for sending WAL data.
2. Edit the pg_hba.conf file:
    ```
    host replication all 0.0.0.0/0 md5
    ```
    * Update this file  by replacing `0.0.0.0/0` with a specific IP address or range to allow trusted connections for replication.
3. Apply the changes by restarting the PostgreSQL server using the command 
  `sudo systemctl restart postgresql`
4. Confirm the PostgreSQL server is correctly set up for logical replication by running the SQL query `SHOW wal_level`. The output should be `logical`.

### Database Permissions

Tumbleweed requires the ability to create an "outbox" table in your source database to propagate changes to consumers. Ensure the provided database user has the necessary privileges to connect to the database, create tables, and insert data into them.

### Updating Queries

Tumblweed also requires you to modify your existing database queries that involve create and update operations.  This involves inserting a corresponding record into the "outbox" table and then deleting it within the same transaction. 

1. Identify all SQL queries in your application that insert or update data in your source database.

2. Modify the queries to use transactions. This will vary based on the language being used. The corresponding record should include the following:
      * aggregatetype
      * aggregateid
      * type
      * payload

Here's an example of adding a transaction to a JavaScript application query that inserts data into an orders table:
```js
await query('BEGIN'); // Transaction Begin

const newOrder = await query(`INSERT INTO orders (product_name, cost)
      VALUES ($1, $2)
      RETURNING *`, 
      [product_name, cost]
    ); // Original query

await query(`INSERT INTO outbox (aggregatetype, aggregateid, type, payload)
      VALUES ($1, $2, $3, $4)`,
      ['order', newOrder.rows[0].id, 'order_created', JSON.stringify(newOrder.rows[0])] // Corresponding record being inserting into the outbox table

await query(`DELETE from outbox`); // Removes the record from the outbox table

await query('COMMIT'); // Transaction End
```

## Deployment

You will need the following:

* An Amazon Web Services (AWS) account
* IP address(es) that will have access to the Tumbleweed UI
* Install [AWS CLI](https://aws.amazon.com/cli/)
* Install [Terraform](https://developer.hashicorp.com/terraform/tutorials/aws-get-started/install-cli)

You are now ready to deploy Tumbleweed! A Command-Line Interface (CLI) tool is provided to simplify the setup and deployment of a Tumbleweed pipeline.

Now run the following command in your command line to get started:

`npx tumbleweed_cdc roll`

The pipeline can be destroyed with the following command:

`npx tumbleweed_cdc burn`
