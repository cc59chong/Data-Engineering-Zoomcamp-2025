# Module 1 Homework: Docker, SQL & Terraform

## Question 1. Understanding docker first run 

Run docker with the `python:3.12.8` image in an interactive mode, use the entrypoint `bash`.

What's the version of `pip` in the image?

- 24.3.1
- 24.2.1
- 23.3.1
- 23.2.1<br><br>
![image](https://github.com/user-attachments/assets/ca379982-f304-4278-823a-2c4e9875a814)<br><br>
Answer: 24.3.1

## Question 2. Understanding Docker networking and docker-compose

Given the following `docker-compose.yaml`, what is the `hostname` and `port` that **pgadmin** should use to connect to the postgres database?

```yaml
services:
  db:
    container_name: postgres
    image: postgres:17-alpine
    environment:
      POSTGRES_USER: 'postgres'
      POSTGRES_PASSWORD: 'postgres'
      POSTGRES_DB: 'ny_taxi'
    ports:
      - '5433:5432'
    volumes:
      - vol-pgdata:/var/lib/postgresql/data

  pgadmin:
    container_name: pgadmin
    image: dpage/pgadmin4:latest
    environment:
      PGADMIN_DEFAULT_EMAIL: "pgadmin@pgadmin.com"
      PGADMIN_DEFAULT_PASSWORD: "pgadmin"
    ports:
      - "8080:80"
    volumes:
      - vol-pgadmin_data:/var/lib/pgadmin  

volumes:
  vol-pgdata:
    name: vol-pgdata
  vol-pgadmin_data:
    name: vol-pgadmin_data
```

- postgres:5433
- localhost:5432
- db:5433
- postgres:5432
- db:5432

If there are more than one answers, select only one of them

Answer: db:5432<br><br>
This is because the db service runs the Postgres database and exposes port 5432 internally in the docker-compose.yaml file. Since the pgadmin service is in the same Docker network as the db service, it can communicate with the db service using the service name (db) and internal port number (5432).<br><br>
##  Prepare Postgres

Run Postgres and load data as shown in the videos
We'll use the green taxi trips from October 2019:

```bash
wget https://github.com/DataTalksClub/nyc-tlc-data/releases/download/green/green_tripdata_2019-10.csv.gz
```

You will also need the dataset with zones:

```bash
wget https://github.com/DataTalksClub/nyc-tlc-data/releases/download/misc/taxi_zone_lookup.csv
```

Download this data and put it into Postgres.

You can use the code from the course. It's up to you whether
you want to use Jupyter or a python script.

**1.Start the Docker Compose services:**
Use Docker Compose to start the PostgreSQL and pgAdmin services in detached mode. 
```docker-compose up -d```

**2.Build the data ingestion Docker image:**
Build the custom Docker image for the data ingestion task using the provided Dockerfile.
```docker build -t taxi_ingest:hw1 .```

**3.Run the data ingestion task:**
Run the data ingestion container to start importing data into the PostgreSQL database from the specified URLs.

**4.Check the status of the PostgreSQL service:**
Check the status of the PostgreSQL service to confirm that it is running correctly.
```docker-compose ps```
![image](https://github.com/user-attachments/assets/5b7a830e-a174-4d4c-83a2-ca5c5e7d77c1)
![image](https://github.com/user-attachments/assets/835bac20-feba-4f0a-b136-702413cf12be)
![image](https://github.com/user-attachments/assets/6b9df2e9-91f2-4b2d-bc14-907ae419abc1)

## Question 3. Trip Segmentation Count

During the period of October 1st 2019 (inclusive) and November 1st 2019 (exclusive), how many trips, **respectively**, happened:
1. Up to 1 mile
2. In between 1 (exclusive) and 3 miles (inclusive),
3. In between 3 (exclusive) and 7 miles (inclusive),
4. In between 7 (exclusive) and 10 miles (inclusive),
5. Over 10 miles 

- 104,802;  197,670;  110,612;  27,831;  35,281
- 104,802;  198,924;  109,603;  27,678;  35,189
- 104,793;  201,407;  110,612;  27,831;  35,281
- 104,793;  202,661;  109,603;  27,678;  35,189
- 104,838;  199,013;  109,645;  27,688;  35,202
 
```
SELECT 
    CASE 
        WHEN trip_distance <= 1 THEN 'Up to 1 mile'
        WHEN trip_distance > 1 AND trip_distance <= 3 THEN 'Between 1 and 3 miles'
        WHEN trip_distance > 3 AND trip_distance <= 7 THEN 'Between 3 and 7 miles'
        WHEN trip_distance > 7 AND trip_distance <= 10 THEN 'Between 7 and 10 miles'
        WHEN trip_distance > 10 THEN 'Over 10 miles'
    END AS distance_category,
    COUNT(*) AS trip_count
FROM 
    green_taxi_trips
WHERE 
    (CAST(lpep_pickup_datetime  AS DATE) >= '2019-10-01') AND (CAST(lpep_pickup_datetime  AS DATE) < '2019-11-01')
GROUP BY 
    distance_category
ORDER BY 
    distance_category DESC;
```
Answers: 104,838; 199,013; 109,645; 27,688; 35,202 X <br>
Correct Answer: 104,802; 198,924; 109,603; 27,678; 35,189

## Question 4. Longest trip for each day

Which was the pick up day with the longest trip distance?
Use the pick up time for your calculations.

Tip: For every day, we only care about one single trip with the longest distance. 

- 2019-10-11
- 2019-10-24
- 2019-10-26
- 2019-10-31

```
SELECT 
    CAST(lpep_pickup_datetime AS DATE) AS longest_trip_distance
FROM 
    green_taxi_trips
GROUP BY 
    CAST(lpep_pickup_datetime AS DATE)
ORDER BY 
    MAX(trip_distance) DESC
LIMIT 1;
```

 Answer: 2019-10-31


## Question 5. Three biggest pickup zones

Which were the top pickup locations with over 13,000 in
`total_amount` (across all trips) for 2019-10-18?

Consider only `lpep_pickup_datetime` when filtering by date.

- East Harlem North, East Harlem South, Morningside Heights
- East Harlem North, Morningside Heights
- Morningside Heights, Astoria Park, East Harlem South
- Bedford, East Harlem North, Astoria Park
  
```
SELECT 
    z.Zone AS pickup_zone,
    SUM(g.total_amount) AS total_amount_sum
FROM green_taxi_trips g
JOIN taxi_zones z
  ON g.PULocationID = z.LocationID
WHERE CAST(g.lpep_pickup_datetime AS DATE) = '2019-10-18'
GROUP BY z.Zone
HAVING SUM(g.total_amount) > 13000
ORDER BY total_amount_sum DESC;
LIMIT 3;
```
Answer: East Harlem North, East Harlem South, Morningside Heights


## Question 6. Largest tip

For the passengers picked up in October 2019 in the zone
named "East Harlem North" which was the drop off zone that had
the largest tip?

Note: it's `tip` , not `trip`

We need the name of the zone, not the ID.

- Yorkville West
- JFK Airport
- East Harlem North
- East Harlem South
  
```
SELECT
    drop_zone.Zone AS dropoff_zone,
    g.tip_amount
FROM green_taxi_trips g
JOIN taxi_zones pu_zone 
    ON g.PULocationID = pu_zone.LocationID
JOIN zones drop_zone
    ON g.DOLocationID = drop_zone.LocationID
WHERE pu_zone.Zone = 'East Harlem North'
  AND CAST(g.lpep_pickup_datetime AS DATE) >= '2019-10-01'
  AND CAST(g.lpep_pickup_datetime AS DATE) < '2019-11-01'
ORDER BY g.tip_amount DESC
LIMIT 1;
```

Answer: JFK Airport



## Terraform

In this section homework we'll prepare the environment by creating resources in GCP with Terraform.

In your VM on GCP/Laptop/GitHub Codespace install Terraform. 
Copy the files from the course repo
[here](../../../01-docker-terraform/1_terraform_gcp/terraform) to your VM/Laptop/GitHub Codespace.

Modify the files as necessary to create a GCP Bucket and Big Query Dataset.


## Question 7. Terraform Workflow

Which of the following sequences, **respectively**, describes the workflow for: 
1. Downloading the provider plugins and setting up backend,
2. Generating proposed changes and auto-executing the plan
3. Remove all resources managed by terraform`

- terraform import, terraform apply -y, terraform destroy
- teraform init, terraform plan -auto-apply, terraform rm
- terraform init, terraform run -auto-approve, terraform destroy
- terraform init, terraform apply -auto-approve, terraform destroy
- terraform import, terraform apply -y, terraform rm
  
Answers: terraform init, terraform apply -auto-approve, terraform destroy<br><br>
terraform init: Downloads provider plugins and sets up the backend. <br>
terraform apply -auto-approve: Generates the execution plan and automatically applies it (i.e., no manual approval required). <br>
terraform destroy: Removes all resources previously managed by Terraform.

## Submitting the solutions

* Form for submitting: https://courses.datatalks.club/de-zoomcamp-2025/homework/hw1
