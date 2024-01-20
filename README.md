# Recoil and bounce!

<img src="img/logo.png" align="right"
     alt="Recoil logo" width="150" height="150">

Recoil enables you to play with everybody's favorite Postgres connection pooler and router [PgBouncer](https://www.pgbouncer.org/). Plus, you can do it from the comfort of your laptop! The enclosed Docker compose file will spin up two Postgres instances - a primary and a read replica. A third service is additionally spun up, which is unsurprisingly, PgBouncer. This service exposes one port and two database names, thus acting like a router. PgBouncer is also configured to have up to 200 client connections. 

## What is PgBouncer? 

PgBouncer is a lightweight connection pooler and router for PostgreSQL. It acts as middleware between your applications and PostgreSQL server(s), managing a pool of connections and reusing them whenever possible. This can significantly improve the performance of an application by reducing the overhead of opening and closing new connections to Postgres.

## Directions to see PgBouncer in action

1. Clone this repository and then change directories into `recoil`. 

2. Decompress the `employees_data.sql.bz2` file by running:

    ```
    $ bunzip2 employees_data.sql.bz2
    ```

    Note, you may need to install [bzip2](https://en.wikipedia.org/wiki/Bzip2). 

3. Next, run `docker compose up -d`, which will start three containers. One container is named `postgres-primary` and the other is named `postgres-replica`. A third service is started aptly named `pgbouncer`. Pay special attention here as this service exposes one port: `5432` but two database names: `recoil` and `recoil-ro` where the latter points at the read replica instance (`postgres-replica`).

4. You can use `psql` to shell to either Postgres instance via the standard `5432` port and changing the database name. For instance, to shell into the primary: 

    ```
    psql postgresql://postgres:recoil@localhost:5432/recoil
    ```

    And to shell into the read replica, simply append a `-ro` to the database name like so:

    ```
    psql postgresql://postgres:recoil@localhost:5432/recoil-ro
    ```
5. You can see this three-way setup in action by adding a record to the primary and then seeing the replica update too. Ensure you've shelled into the primary (again) like so:

    ```
    psql postgresql://postgres:recoil@localhost:5432/recoil
    ```

    Now, type the following SQL statement:

    ```
    select count(*) from employees.employee;
    ```

    You should see a result like so:

    ```
     count
    --------
    300024
    (1 row)
    ```

6. In the same `psql` session on the primary, add a new row to the `employee` table like so:

    ```
    insert into employees.employee values (99999999999, '1987-03-21', 'Joe', 'Smith', 'M', '2022-01-22');
    ```
    
    Rerun the SQL count query from above and you should see that the total count of employees is now 300,025. 

7. Next, either `\q` out of the primary `psql` session or open up a new terminal window and use `psql` to shell into the read replica instance like so:

    ```
    psql postgresql://postgres:recoil@localhost:5432/recoil-ro
    ```

    Issue the employee count query (`select count(*) from employees.employee;`) and you should see that the read replica also returns 300,025. 

## What's all this mean?

In essence, PgBouncer is acting like both a connection pool (with 200 connections) and a router. Conceptually, it's like a load balancer. Keen observers will note that the primary has a backdoor via port `49149`. It's with this port that I can demonstrate PgBouncer's other handy feature: connection pools. 

