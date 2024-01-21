# Recoil and bounce!

<img src="img/logo.png" align="right"
     alt="Recoil logo" width="150" height="150">

Recoil enables you to play with everybody's favorite Postgres connection pooler and router [PgBouncer](https://www.pgbouncer.org/). Plus, you can do it from the comfort of your laptop! The enclosed Docker compose file will spin up two Postgres instances - a primary and a read replica. A third service is additionally spun up, which is unsurprisingly, PgBouncer. This service exposes one port and two database names, thus acting like a router. PgBouncer is also configured to have up to 200 client connections. 

## What is PgBouncer? 

PgBouncer is a lightweight connection [pooler](#directions-to-see-pgbouncer-pooling-in-action) and [router](#directions-to-see-pgbouncer-routing-in-action) for PostgreSQL. It acts as middleware between your applications and PostgreSQL server(s), managing a pool of connections and reusing them whenever possible. Pooling can significantly improve the performance of an application by reducing the overhead of opening and closing new connections to Postgres. Routing too can improve the performance of a database by easily enabling routing to [horizontally scaled instances](https://www.thediscoblog.com/horizontally-scaling-postgres-with-read-replicas/). 

### Directions to see PgBouncer routing in action

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

In essence, PgBouncer is acting like a proxy or and a router. In fact, conceptually, it's like a load balancer. It does represent a single point of failure, but if you zoom out slightly, you'll remember that PgBouncer also is a connection pool.

### Directions to see PgBouncer pooling in action 

 Connections in Postgres are expensive because Postgres creates a new backend process for each connection, which involves memory allocation and initialization tasks. A connection pooler manages a set of connections on behalf of a service like a database thereby freeing up the service's resources to focus on other aspects, such as rapidly returning data. 
 
 To See how PgBouncer works as a connection pool, the following directions use [pgbench](https://www.postgresql.org/docs/current/pgbench.html), which is a simple command line utility to run benchmark tests on a Postgres instance. It's an easy way to exhaust database resources, which you'll do. Subsequently, you'll point pgbench at PgBouncer and see an improvement. 
 
 Before you start these directions, ensure no previous instances of Recoil's containers are running. Simply issue a `docker compose down` in the `recoil` directory if so. You'll be using the `pgbench` command line utility, so ensure that's in your path (i.e. make sure `pgbench --help` works). 

 1. First, fire up your favorite editor as you'll need to make a slight tweak of the `docker-compose.yml` file found in this repo. Look for the `ports` definition on the `postgres-primary` service and uncomment the lines: 

    ```
    ports:
     - "49149:5432"
    ```
    This'll enable access to this instance of Postgres via port `49149`. 

2. Next, issue the following command starting only the `postgres-primary` container: 

    ```
    docker compose up postgres-primary -d
    ```

3. At this point, you should only have a single Postgres instance running. Moreover, it's listening on port `49149`. In a terminal window, issue the following `pgbench` command, which creates four tables: `pgbench_accounts`, `pgbench_branches`, `pgbench_history`, and `pgbench_tellers` (in the `recoil` database) and generally prepares a Postgres instance for benchmarking:

    ```
    pgbench -i -s 500 -h localhost -p 49149 -U postgres recoil
    ```

    You'll be prompted for a password. You can type `export PGPASSWORD=recoil` in your terminal to avoid this hassle going forward. 
 
4. Now, rerun `pgbench`, but this time, the `-i` flag is omitted, which means it'll attempt to hammer your local Postgres instance. Issue the following command: 

    ```
    pgbench -c 101 -j 2 -t 100000 -h localhost -p 49149 -U postgres -S recoil
    ```

    The key flag in this step is `-c` which represents the number of clients or connections. 101 is beyond the default maximum for Postgres (which is 100). 

    You should see `pgbench` error out with a message along the lines of:

    ```
    FATAL:  sorry, too many clients already
    ```

5. You can assess what's going on by firing up `psql` and seeing how Postgres is configured. First, in a terminal window shell into the primary like so: 

    ```
    psql postgresql://postgres:recoil@localhost:49149/recoil
    ```

    Then, issue this query:

    ```
    SELECT name, setting FROM pg_settings WHERE name = 'max_connections';
    ``` 

    You should see that, by default, the value is 100: 

    ```
    name       | setting
    -----------+---------
    max_connections | 100
    ```
    
    So, in effect, the `pgbench` test you just ran exhausted the available connections for Postgres and it errored out. Of course, you could increase this setting, but doing so will come at a cost. It's a tradeoff. On the other hand, you could use PgBouncer, which will handle connections quite nicely. 

6. Go ahead and shut down the single Postgres container via a `docker compose down`. 

7. Start all three services up via `docker compose up -d`. 

8. Next, you're going to reseed the `recoil` database on the Primary instance using `pgbench` using the same exact command from step 3, but this time going through PgBouncer, which is listening on port `5432`. In a terminal window type: 

    ```
    pgbench -i -s 500 -h localhost -p 5432 -U postgres recoil
    ```

9. Rerun the benchmark using `pgbench` that uses 101 client connections like so: 

    ```
    pgbench -c 101 -j 2 -t 100000 -h localhost -p 5432 -U postgres -S recoil
    ```

    This'll take some time to complete. Incidentally, the initialize command (i.e. using `-i`) seeded the database with 50,000,000 rows (i.e. `-s 500`). The command you just ran issues 100,000 SQL statements (i.e. `-t`) for each client. Depending on your machine's CPU and memory, this could take 5 to 10 minutes to run. When `pgbench` finishes, it'll print some statistics such as tps, which is transactions per second. Ideally, your goal is to _economically_ increase that number.

    Take a peek at the `docker-compose.yml` file; specifically, the definition of the `pgbouncer` service. You'll note that there's an explicit environmental variable that sets the maximum connections to 200 (`PGBOUNCER_MAX_CLIENT_CONN=200`). 

As you can see, by configuring PgBouncer with 200 connections and putting it in front of Postgres, the `pgbench` benchmark test is able to run. 

## Benefits of PgBouncer

 With a connection pooler like PgBouncer, you get benefits like:
 
  * Improved performance: by reusing connections, PgBouncer can significantly reduce the time it takes for your application to connect to a database. This can lead to faster loading times and improved responsiveness.
  * Reduced load on the database server: by managing a pool of connections, PgBouncer can help to prevent your application from overloading the database server with too many connections. This can be especially important for applications with a high number of users.
  * Increased stability: PgBouncer can help to prevent your application from crashing if the database server becomes unavailable. This is because PgBouncer can retry failed connections and can also gracefully handle situations where the database server is overloaded.
  * Improved security: PgBouncer can help to improve the security of your application by providing features such as connection limiting and user authentication.


