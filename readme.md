## Docker Compose Documentation

This Docker Compose file defines several services for a local development environment. The setup includes the following services:

### 1. **Dremio**
- **Image**: `dremio/dremio-oss:latest`
- **Container Name**: `dremio`
- **Ports**:
  - 9047: Web UI
  - 31010: ODBC/Arrow Flight
  - 32010: JDBC
  - 45678: Internal services
- **Environment Variables**:
  - `DREMIO_JAVA_SERVER_EXTRA_OPTS`: Path to Dremio distribution files.
- **Network**: `dremio-notebook`

### 2. **Spark**
- **Image**: `alexmerced/spark35nb:latest`
- **Ports**:
  - 8080: Spark Master Web UI
  - 7077: Spark Master job submission port
  - 8081: Worker Web UI
  - 4040-4045: Spark job UIs for concurrent jobs
  - 18080: Spark History Server
  - 8888: Jupyter Notebook
- **Environment Variables**:
  - `AWS_REGION`: AWS region for access.
  - `AWS_ACCESS_KEY_ID`: Minio username.
  - `AWS_SECRET_ACCESS_KEY`: Minio password.
  - `SPARK_MASTER_HOST`: Spark master hostname.
  - Other Spark-related configurations.
- **Volumes**: 
  - Maps local seed data to the `/workspace/seed-data` directory.
- **Entry Point**: Runs Spark services, History Server, and Jupyter Notebook on container startup.
- **Network**: `dremio-notebook`

### 3. **Minio**
- **Image**: `minio/minio`
- **Container Name**: `minio`
- **Ports**:
  - 9000: Minio server
  - 9001: Minio console
- **Environment Variables**:
  - `MINIO_ROOT_USER`: Admin username.
  - `MINIO_ROOT_PASSWORD`: Admin password.
  - `MINIO_REGION`: Set region for Minio.
- **Healthcheck**: Ensures Minio is healthy with a live check on port 9000.
- **Volumes**:
  - Maps local seed data to the `/minio-data` directory.
- **Entry Point**: Starts Minio server, configures Minio client (mc), creates necessary buckets, and copies seed data.
- **Network**: `dremio-notebook`

### 4. **Nessie**
- **Image**: `projectnessie/nessie:latest`
- **Container Name**: `nessie`
- **Ports**:
  - 19120: Nessie API
- **Environment Variables**:
  - Configures Nessie for production profile, logging, and RocksDB storage backend.
- **Network**: `dremio-notebook`

### 5. **Postgres**
- **Image**: `postgres:13`
- **Container Name**: `postgres`
- **Ports**:
  - 5435: PostgreSQL database
- **Environment Variables**:
  - `POSTGRES_USER`: Username for the Postgres database.
  - `POSTGRES_PASSWORD`: Password for the Postgres database.
  - `POSTGRES_DB`: Default database name.
- **Volumes**:
  - Seeds the database using SQL files from the local directory.
- **Network**: `dremio-notebook`

### 6. **Mongo**
- **Image**: `mongo:4.4`
- **Container Name**: `mongo`
- **Ports**:
  - 27017: MongoDB
- **Environment Variables**:
  - `MONGO_INITDB_ROOT_USERNAME`: Root username for MongoDB.
  - `MONGO_INITDB_ROOT_PASSWORD`: Root password for MongoDB.
- **Volumes**:
  - Seeds MongoDB using initialization scripts from the local directory.
- **Network**: `dremio-notebook`

### Networks
- A custom network `dremio-notebook` is created to ensure inter-service communication.


## Running the Services

To start the services, run the following command:

```bash
docker-compose up -d
```

Running this command will start all the containers in detached mode. Once all the services are up and running, you can access the following:

- **Dremio UI**: Accessible at [http://localhost:9047](http://localhost:9047)
- **MinIO Console**: Accessible at [http://localhost:9001](http://localhost:9001)
- **Nessie API**: Accessible at [http://localhost:19120](http://localhost:19120)
- **Postgres**: Available on port `5435`
- **MongoDB**: Available on port `27017`
- **Spark Master Web UI**: Accessible at [http://localhost:8080](http://localhost:8080)
- **Spark Worker Web UI**: Accessible at [http://localhost:8081](http://localhost:8081)
- **Spark History Server**: Accessible at [http://localhost:18080](http://localhost:18080)
- **Jupyter Notebook**: Accessible at [http://localhost:8888](http://localhost:8888)


## Stopping the Services

To stop the services, run:

```bash
docker-compose down
```

If you want to remove all data volumes (i.e., reset the environment):

```bash
docker-compose down -v
```

# Connecting Nessie, MinIO, Postgres, and MongoDB to Dremio from the Dremio Web UI

Once all services are up and running via Docker Compose, you can connect these sources to Dremio from the Dremio Web UI.

## Prerequisites

Ensure the services are running and accessible:

- Dremio UI: `http://localhost:9047`
- MinIO: `http://localhost:9001`
- Nessie API: `http://localhost:19120`
- Postgres: Accessible via `localhost:5435`
- MongoDB: Accessible via `localhost:27017`

## 1. Connecting to Nessie (Versioned Data Lake)

Nessie is a version control system for your data lake and can be connected to Dremio using the built-in Nessie support.

### Steps:

1. **Log in to Dremio**: Open your browser and go to `http://localhost:9047`. Log in with your Dremio credentials.
   
2. **Navigate to Sources**:
   - On the Dremio Web UI, click on the **Sources** tab in the left-hand sidebar.
   
3. **Add Nessie as a Source**:
   - Click **+ Source** in the upper-right corner.
   - Select **Nessie** from the list of available source types.

4. **Configure Nessie Source**:
   - **Name**: Give your Nessie source a name (e.g., `nessie`).
   - **Nessie REST API URL**: Enter `http://nessie:19120` (the API URL exposed by the Nessie container, based on the `container_name` defined in the `docker-compose.yml` file).
   - **Authentication**: Choose `None`

5. **Configure Nessie Storage**:
    - set the warehouse address to the name of the bucket in MinIO (datalakehouse)
    - access key and secret key are the same as the ones used to access MinIO defined  in the docker-compose.yml file. (admin/password)
    - set the following custom parameters:
        - `fs.s3a.path.style.access` to true
        - `fs.s3a.endpoint` to the endpoint of MinIO (minio:9000)
        - `dremio.s3.compat` to true
    
6. **Click Save**: Once the configuration is set, click **Save**. Dremio will now be connected to Nessie, and you will be able to read and write versioned data using Iceberg tables managed by Nessie.

---

## 2. Connecting to MinIO (Object Storage)

MinIO provides S3-compatible object storage, and you can add it as an external source in Dremio.

### Steps:

1. **Log in to Dremio**: If you're not already logged in, go to `http://localhost:9047` and log in.

2. **Navigate to Sources**:
   - Click on the **Sources** tab in the left-hand sidebar.

3. **Add MinIO as a Source**:
   - Click **+ Source** in the upper-right corner.
   - Select **Amazon S3** from the list of available source types. MinIO is S3-compatible, so use this option.

4. **Configure MinIO Source**:
   - **Name**: Give your MinIO source a name (e.g., `minio`).
   - **Access Key**: Enter `admin` (as per your Docker Compose environment variables).
   - **Secret Key**: Enter `password` (as per your Docker Compose environment variables).
   - **External Bucket Name**: Specify the bucket you want to access. For example, `datalake`.
   - **Root Path**: Leave blank if accessing the whole bucket, or provide a specific path.
   - **Encryption**: Set this to `None` for unencrypted data. (since your operating in this demo environment)
   - **Enable Compatibility with AWS S3**: Check this option since MinIO is S3-compatible.
   - **Connection Properties**: set the following connection properties:
        - `fs.s3a.path.style.access` to true
        - `fs.s3a.endpoint` to the endpoint of MinIO (minio:9000)

5. **Click Save**: Once the configuration is set, click **Save**. You will now be able to browse and query data stored in MinIO directly from Dremio.

---

## 3. Connecting to Postgres

Dremio natively supports PostgreSQL, making it easy to connect to a Postgres instance and query data from Dremio.

### Steps:

1. **Log in to Dremio**: If you're not already logged in, go to `http://localhost:9047` and log in.

2. **Navigate to Sources**:
   - Click on the **Sources** tab in the left-hand sidebar.

3. **Add Postgres as a Source**:
   - Click **+ Source** in the upper-right corner.
   - Select **PostgreSQL** from the list of available source types.

4. **Configure Postgres Source**:
   - **Name**: Give your Postgres source a name (e.g., `postgres_nessie`).
   - **Hostname**: Enter `postgres` (or `postgres` if referencing the Docker container name).
   - **Port**: Enter `5432`.
   - **Database**: Enter `mydatabase` (the database name set in your Docker Compose environment variables).
   - **Username**: Enter `admin` (the Postgres user defined in your Docker Compose environment).
   - **Password**: Enter `password` (the Postgres password defined in your Docker Compose environment).

5. **Click Save**: After filling in the details, click **Save**. You can now query data stored in your Postgres database from Dremio.

---

## 4. Connecting to MongoDB

MongoDB is a NoSQL database, and Dremio has native support for MongoDB connections.

### Steps:

1. **Log in to Dremio**: If you're not already logged in, go to `http://localhost:9047` and log in.

2. **Navigate to Sources**:
   - Click on the **Sources** tab in the left-hand sidebar.

3. **Add MongoDB as a Source**:
   - Click **+ Source** in the upper-right corner.
   - Select **MongoDB** from the list of available source types.

4. **Configure MongoDB Source**:
   - **Name**: Give your MongoDB source a name (e.g., `mongo`).
   - **Host**: Enter `mongo` (or `mongo` if referencing the Docker container name).
   - **Port**: Enter `27017` (the default MongoDB port).
   - **Authentication**: If MongoDB is set to require authentication, enable authentication and fill in the following:
     - **Username**: `admin` (as per your Docker Compose configuration).
     - **Password**: `password` (as per your Docker Compose configuration).
     - **Authentication Database**: `admin` (the default MongoDB authentication database).
   - **Database**: Specify the default database to connect to (or leave it blank to list all databases).

5. **Click Save**: Once you’ve filled in the necessary details, click **Save**. Dremio will now be connected to MongoDB, allowing you to query collections directly from Dremio.

---

## Summary

Once all the services are connected, you can explore data from:

- **Nessie**: Version-controlled datasets and tables.
- **MinIO**: S3-compatible object storage.
- **Postgres**: Relational database queries.
- **MongoDB**: NoSQL document collections.

These sources can be queried, transformed, and visualized in Dremio using SQL queries. You can also join data from multiple sources (e.g., join MongoDB collections with Postgres tables or MinIO object storage data).

## Troubleshooting

- **Connection Issues**: If you experience issues connecting to any source, ensure that the Docker services are running and accessible. Use `docker ps` to verify the status of containers.
- **Logs**: Check the logs of each container if you face issues. For example:
  - `docker logs postgres`
  - `docker logs mongo`
  - `docker logs minio`
- **Firewalls/Networking**: Ensure that your Docker networking allows access to all services from the Dremio container.

# Query to Join Sample Data

```sql
SELECT 
    c.customer_name, 
    c.email, 
    o.order_date, 
    o.amount, 
    p.preference, 
    p.loyalty_status
FROM 
    postgres.public.customers AS c
JOIN 
    postgres.public.orders AS o ON c.customer_id = o.customer_id
JOIN 
    mongo.mydatabase.customer_preferences AS p ON c.email = p.customer_email;
```