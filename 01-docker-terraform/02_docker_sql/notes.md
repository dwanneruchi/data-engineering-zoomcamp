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

We aren't 