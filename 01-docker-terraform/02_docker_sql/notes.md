### Docker Lecture 1.2.1

General introduction to containerization. Below are some key points:
- `Dockerfile` is going to contain all the necessary information for container build
- Run the following in location with `Dockerfile` to install and establish: ` docker build -t test:pandas .`
- And then to actually run something like a docker pipeline you'd do the following:
```bash
 docker run -it test:pandas 2024-10-10
```
- the above indicates we want to run the `test:pandas` container we built in `interactive` mode. 
- it seems like `interactive` mode is going to allow us to then pass in command, such as `2024-10-10` which would just indicate the date of file to execute on
- The output in this example was as follows:
```bash
['pipeline.py', '2024-10-10']
job finished successfully for day = 2024-10-10 
```

### Docker Lecture 1.2.2

#### A) Setting up Database
Initial overview of setting up a postgres database via Docker. The command to run is:
```buildoutcfg
docker run -it \
  -e POSTGRES_USER="root" \
  -e POSTGRES_PASSWORD="root" \
  -e POSTGRES_DB="ny_taxi" \
  -v /Users/davidwanner/Repos/data_engineering/data-engineering-zoomcamp/ny_taxi_postgres_data:/var/lib/postgresql/data \
  -p 5432:5432 \
postgres:13
```

Important points:
- `postgres13` indicates the specific image of postgres docker will use
- I then had to create a local folder (`/Users/davidwanner/Repos/data_engineering/data-engineering-zoomcamp/ny_taxi_postgres_data`)
  - That local folder is mapped to a folder in docker, `/var/lib/postgresql/data `
  - My assumption is that this is my connection point for getting data over
- We then get a note showing that the system is ready:
```buildoutcfg
database system is ready to accept connections
```
#### B) Connecting to Database via Terminal

We can then connect to that Docker database:
```buildoutcfg
pgcli -h localhost -p 5432 -u root -d ny_taxi
```
Have to add our password and should see our connection to the database specified by `-d`:
```buildoutcfg
root@localhost:ny_taxi>
```
#### C) Connecting to Database via Python and Loading Data
With that Docker image running we can then connect within jupyter notebook:
```python
from sqlalchemy import create_engine

engine = create_engine('postgresql://root:root@localhost:5432/ny_taxi')
engine.connect() # <sqlalchemy.engine.base.Connection at 0x17d9c8c70>
```

Also, we needed to define a DDL schema (Data Definition Language):
```python
# get schema - this gives us the DDL schema for our connection
print(pd.io.sql.get_schema(df, 'yellow_taxi_data', con = engine))
```

And we get the following:
```python
CREATE TABLE yellow_taxi_data (
	"VendorID" FLOAT(53), 
	tpep_pickup_datetime TIMESTAMP WITHOUT TIME ZONE, 
	tpep_dropoff_datetime TIMESTAMP WITHOUT TIME ZONE, 
	passenger_count FLOAT(53), 
	trip_distance FLOAT(53), 
	"RatecodeID" FLOAT(53), 
	store_and_fwd_flag TEXT, 
	"PULocationID" BIGINT, 
	"DOLocationID" BIGINT, 
	payment_type FLOAT(53), 
	fare_amount FLOAT(53), 
	extra FLOAT(53), 
	mta_tax FLOAT(53), 
	tip_amount FLOAT(53), 
	tolls_amount FLOAT(53), 
	improvement_surcharge FLOAT(53), 
	total_amount FLOAT(53), 
	congestion_surcharge FLOAT(53)
)
```
Our process will load the data (since it is large) using an iterator, where pandas will only load based on `chunksize`.

```python
# We have a rather large set of data so first we will create our table via our connection
df.head(n = 0).to_sql(name = 'yellow_taxi_data', con = engine, if_exists = 'replace')
```

In the terminal (Step B above) we can run the following:
```buildoutcfg
\d yellow_taxi_data
```
This will yield the newly created table schema:
```buildoutcfg
root@localhost:ny_taxi> \d yellow_taxi_data
+-----------------------+-----------------------------+-----------+
| Column                | Type                        | Modifiers |
|-----------------------+-----------------------------+-----------|
| index                 | bigint                      |           |
| VendorID              | double precision            |           |
| tpep_pickup_datetime  | timestamp without time zone |           |
| tpep_dropoff_datetime | timestamp without time zone |           |
| passenger_count       | double precision            |           |
| trip_distance         | double precision            |           |
| RatecodeID            | double precision            |           |
| store_and_fwd_flag    | text                        |           |
| PULocationID          | bigint                      |           |
| DOLocationID          | bigint                      |           |
| payment_type          | double precision            |           |
| fare_amount           | double precision            |           |
| extra                 | double precision            |           |
| mta_tax               | double precision            |           |
| tip_amount            | double precision            |           |
| tolls_amount          | double precision            |           |
| improvement_surcharge | double precision            |           |
| total_amount          | double precision            |           |
| congestion_surcharge  | double precision            |           |
+-----------------------+-----------------------------+-----------+
```

And we can load our data:
```python
from time import time

i = 0 
# establish iterator that will load based on chunk_size
df_iter = pd.read_csv("ny_taxi_postgres_data/yellow_tripdata_2021-01.csv", iterator = True, chunksize=100_000)

# iterate until we exhaust iterator
while True:
    t_start = time()
    
    df = next(df_iter) # take next from iter

    # convert to datetime (must do each step since reading from csv)
    df.tpep_pickup_datetime = pd.to_datetime(df.tpep_pickup_datetime)
    df.tpep_dropoff_datetime = pd.to_datetime(df.tpep_dropoff_datetime)
    
    # load the chunk via append
    df.to_sql(name = 'yellow_taxi_data', con = engine, if_exists = 'append')
    total_time = time() - t_start
    i += 1
    print(f'inserted chunk {i}, took {total_time:.3f} seconds')
```

Some stats:
- 14 chunks (last chunk looks like < `chunksize` as faster)
- Took about 4.5 seconds per chunk, not bad - speedy mac!

And we can use the `pgcli` tool to check some basic stuff:
```buildoutcfg
root@localhost:ny_taxi> select count(*) from yellow_taxi_data;
+---------+
| count   |
|---------|
| 1369765 |
+---------+
```
### 1.2.3 Connecting pdAdmin and postgres

This section moves away from being restricted to command line.

We can use the image for pgAdmin4. Having once taught a SQL course that relied on pgAdmin 4 this is AWESOME. the installation caused a ton of headaches back then.

```buildoutcfg
# moving into running container:
docker run -it \
  -e PGADMIN_DEFAULT_EMAIL="admin@admin.com" \
  -e PGADMIN_DEFAULT_PASSWORD="root" \
  -p 8080:80 \
  dpage/pgadmin4
```
Running that will then give us a port to access via web:
```buildoutcfg
[2024-12-23 23:01:15 +0000] [1] [INFO] Starting gunicorn 22.0.0
[2024-12-23 23:01:15 +0000] [1] [INFO] Listening at: http://[::]:80 (1)
[2024-12-23 23:01:15 +0000] [1] [INFO] Using worker: gthread
[2024-12-23 23:01:15 +0000] [123] [INFO] Booting worker with pid: 123
```
We can then direct via `http://localhost:8080/login?next=/` and are prompted to login using credentials above.

However, we can't actually set our server up without first using [docker network](https://docs.docker.com/engine/network/)
- My understanding of this is that we use the network command in order to allow our database (postgres) to connect to pgadmin

We establish this network with:
```buildoutcfg
docker network create pg-network
```
I now have a Docker network named `pg-network` which will allow multiple Docker containers to communicate.

We then update some of the prior commands.

#### Postgres Database
```buildoutcfg
docker run -it \
  -e POSTGRES_USER="root" \
  -e POSTGRES_PASSWORD="root" \
  -e POSTGRES_DB="ny_taxi" \
  -v /Users/davidwanner/Repos/data_engineering/data-engineering-zoomcamp/ny_taxi_postgres_data:/var/lib/postgresql/data \
  -p 5432:5432 \
  --network=pg-network \
  --name pg-database \
postgres:13
```
- `--network=pg-network` is going to specify that this container should be connected to the Docker network `pg-network`
- `--name pg-database` is going to be a name for referring to this Docker container rather than needing the Docker id.
- This means we could use something like `docker stop pg-database` to stop the container
- Also, since we already passed data in we don't need to reestablish. The whole point of using a mount here is for `persistence`, meaning data is preserved when the container is stopped.

#### PgAdmin connection
```buildoutcfg
docker run -it \
  -e PGADMIN_DEFAULT_EMAIL="admin@admin.com" \
  -e PGADMIN_DEFAULT_PASSWORD="root" \
  -p 8080:80 \
  --network=pg-network \
  --name pgadmin \
  dpage/pgadmin4
```
Similar to the above, we have now adding the open-source management tool `pgAdmin` to our `pg-network`. This will allow us to use pgAdmin to interact with our PostgreSQL database.

There is a lot of value in using `pgAdmin` via Docker vs installing:
- ease of setup: no manual install
- isolation: can isolate to a container from host system, avoiding conflict with other software
- consistency: we can always use the same version of `pgAdmin` - very helpful with reproducibility.

#### Adding a Server

We run the 3 commands above and need to log into our local: `http://localhost:8080/login?next=/`

![plot](images/pgadmin_server.png)

We then need to establish a connection, which can be done by using the `pg-database` host, which lets us connect to our container.

And we can see our database and do all kinds of SQL things:
![pgadmin_container](images/pgadmin_container.png)
