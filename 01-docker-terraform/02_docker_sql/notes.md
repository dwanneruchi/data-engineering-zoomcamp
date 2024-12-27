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

### Building Via the `ingest.py` file

This took me a bit, a few things to note:
- Need to restart the containers built above:
  - Start the pg-admin with : `docker start pg-admin`
  - Start the database with: `docker start pg-database`
- Also needed to clear out `yellow_taxi_data` if trying to load there. I just decided to try a new table name
- Also, I am on a Mac and decided to locally install `wget`. That said, since this is running in a container I don't know if I needed to do that

The command once the above is done:
```buildoutcfg
# URL to the gzipped files
URL="https://github.com/DataTalksClub/nyc-tlc-data/releases/download/yellow/yellow_tripdata_2021-01.csv.gz"

# command to run the actual ingest with the relevant params
python ingest.py \
  --user=root \
  --password=root \
  --host=localhost \
  --port=5432 \
  --db=ny_taxi \
  --table_name=yellow_taxi_script_build \
  --url=${URL}
```

And here is the output, we can see the script has pulled down data from Git, batched, and inserted:

![automated_load](images/automated_load.png)

Let's take a closer look at the script `ingest.py`:

#### Ingestion Process:
The below script is how we pass over all of our relevant info:
```python
if __name__ == '__main__':
    parser = argparse.ArgumentParser(description='Ingest CSV data to Postgres')

    parser.add_argument('--user', required=True, help='user name for postgres')
    parser.add_argument('--password', required=True, help='password for postgres')
    parser.add_argument('--host', required=True, help='host for postgres')
    parser.add_argument('--port', required=True, help='port for postgres')
    parser.add_argument('--db', required=True, help='database name for postgres')
    parser.add_argument('--table_name', required=True, help='name of the table where we will write the results to')
    parser.add_argument('--url', required=True, help='url of the csv file')

    args = parser.parse_args()

    main(args)
```
`argparse` libary will handle the various args needed for establishing our connection. 

Then the `main` function in the script gets this `args` object which can be unpacked:
```python
    user = params.user
    password = params.password
    host = params.host
    port = params.port
    db = params.db
    table_name = params.table_name
    url = params.url
```

And then the same steps as before can be run:
- create engine (our connect to the postgres database):
```python
engine = create_engine(f'postgresql://{user}:{password}@{host}:{port}/{db}')
```
- Build the schema for the table from our input data (using 0 records)
```python
df.head(n=0).to_sql(name=table_name, con=engine, if_exists='replace')
```
- iterate through the iterable created and append data in `batches`:
```python
df.to_sql(name=table_name, con=engine, if_exists='append')
```

So now instead of needing to manually gather our data, open up a jupyter notebook and make connections we have a single scripts that can be run.

And we see now that there are two tables available:

![two_table](images/two_table.png)


### 1.2.5 Docker-Compose

Great help: https://www.youtube.com/watch?v=hKI6PkPhpa0&list=PL3MmuxUbc_hJed7dXYoJw8DoCuVHhGEQb&index=9

The main point here is that we can run multiple, related services with just one config file.
So, instead of needing to run `postgres` container and `pgadmin` container we can run this single compose

```yaml
services:
  pgdatabase:
    image: postgres:13
    environment:
      - POSTGRES_USER=root
      - POSTGRES_PASSWORD=root
      - POSTGRES_DB=ny_taxi
    volumes:
      - "/Users/davidwanner/Repos/data_engineering/data-engineering-zoomcamp/ny_taxi_postgres_data:/var/lib/postgresql/data"
    ports:
      - "5432:5432"
  pgadmin:
    image: dpage/pgadmin4
    environment:
      - PGADMIN_DEFAULT_EMAIL=admin@admin.com
      - PGADMIN_DEFAULT_PASSWORD=root
    ports:
      - "8080:80"
```
In the above we are specifying:
- pg database 
  - Note: on my mac I needed to explicitly add full path in volumes 
- pgadmin connection

Starting it:
`docker compose up`

Or, detached mode: `docker compose up -d`

I found it suprising that I did not need to point to the `docker-compose.yaml` specifically. My assumption is docker looks for a `.yaml`?

Ah, yes:
```yaml
If you don't specify a file with the -f flag, docker compose up will look for a file named compose.yaml in the current directory. 
If it doesn't find that, it will look for docker-compose.yml.
```

And when we run `docker ps` we see that two containers are now running:
![docker_compose](images/docker compose.png)
 
We will need to reconfigure our pgdatabase now for a docker version, I just used the same steps as above and called it `docker local`.
- I will now have access to the database previously built when I run docker compose.

Finally, we can shut this down with `ctrl+c`, but the proper way would be `docker compose down`