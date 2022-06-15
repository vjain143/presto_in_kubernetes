

# Step 0: Install Kind cluster if you dont have any other

Create a Kind Cluster, with memory limitation

```
kind create cluster --config /home/alex/coding/preso_hive/kind-cluster-config.yaml
```

Install Minio for S3

Via helm

```
https://github.com/minio/minio/tree/master/helm/minio
helm install mino-test -f mino/values.yaml  minio/minio
```

Create a bucket `test` from the UI

```
kubectl port-forward svc/mino-test-minio-console 9001
```

# Step 1: Install Postgres in Kubernetes with Kubegres Operator

Postgres in Kubernetes https://www.kubegres.io/

https://www.kubegres.io/doc/getting-started.html


```
kubectl apply -f https://raw.githubusercontent.com/reactive-tech/kubegres/v1.15/kubegres.yaml
kubectl apply -f postgres/postgres-secret.yaml
kubectl apply -f postgres/kubegres-porstrescluster.yaml
```
```
kubectl get pods
NAME             READY   STATUS    RESTARTS   AGE
mypostgres-1-0   1/1     Running   0          22m
mypostgres-2-0   1/1     Running   0          22m
mypostgres-3-0   1/1     Running   0          22m
```

Manually Create a DB after installing Postgres

```
 kubectl  exec -it mypostgres-1-0 /bin/sh
 psql -U postgres
 <password from the secret>

postgres=# create database metadata;
CREATE DATABASE
```
Check if the DB is cerated

```
postgres=# \list
                                 List of databases
   Name    |  Owner   | Encoding |  Collate   |   Ctype    |   Access privileges   
-----------+----------+----------+------------+------------+-----------------------
 metadata  | postgres | UTF8     | en_US.utf8 | en_US.utf8 | 
 postgres  | postgres | UTF8     | en_US.utf8 | en_US.utf8 | 
 template0 | postgres | UTF8     | en_US.utf8 | en_US.utf8 | =c/postgres          +
           |          |          |            |            | postgres=CTc/postgres
 template1 | postgres | UTF8     | en_US.utf8 | en_US.utf8 | =c/postgres          +
           |          |          |            |            | postgres=CTc/postgres

```

Other commands

 \c metadata (conenct to metadata DB)

 \dt (list the tables - After Step 2.1)



Get the EP  of Postgres

```
kubectl get ep 

mypostgres               10.244.0.78:5432                    9h
mypostgres-replica       10.244.0.80:5432,10.244.0.82:5432   9h
```

Update it in connection string
in `hive/metastore-cfg.yaml` and `hive/hive-initschema.yaml`



# Step 2: Install Hive Metadata Standalone


## Step 2.1

Run the hive/hive-initschema.yaml Job to intialise the schema in the Postgre table

```
kubectl apply -f hive/hive-initschema.yaml
```

and verify if the tables are created properly

```

 kubectl  exec -it mypostgres-1-0 /bin/sh
 psql -U postgres

\c metadata
\dt
                     List of relations
 Schema |             Name              | Type  |  Owner   
--------+-------------------------------+-------+----------
 public | BUCKETING_COLS                | table | postgres
 public | CDS                           | table | postgres
 public | COLUMNS_V2                    | table | postgres
 public | CTLGS                         | table | postgres
...

````


Create the S3 secrets for Hive

```
kubectl create secret generic my-s3-keys --from-literal=access-key=’minio’ --from-literal=secret-key=’minio123’
```

Get the Mino/S3 EP

```
kubectl get ep  

alex@pop-os:~/coding/preso_hive$ kubectl get ep
NAME                      ENDPOINTS                                                        AGE
kubernetes                172.18.0.3:6443                                                  23h
metastore                 10.244.1.37:9083                                                 16m
mino-test-minio           10.244.1.33:9000,10.244.1.34:9000,10.244.1.35:9000 + 1 more...   18h
mino-test-minio-console   10.244.1.33:9001,10.244.1.34:9001,10.244.1.35:9001 + 1 more...   18h
mino-test-minio-svc       10.244.1.33:9000,10.244.1.34:9000,10.244.1.35:9000 + 1 more...   18h
mypostgres                10.244.1.4:5432                                                  23h
mypostgres-replica        10.244.1.6:5432,10.244.1.8:5432                                  23h
```

Update in `hive\metastroe-cfg.yaml` for S3

The following properties using Endpoints or Ingress for S3 and Postgres

```
<name>fs.s3a.endpoint</name>
<value>http://10.244.1.33:9000:9000</value>
```                

Install the hive metdata server 

```
kubectl apply -f   hive/metastore-cfg.yaml
kubectl apply -f  hive/hive-meta-store-standalone.yaml
```


# Step 4. Install Trino (PrestoSQL)

Configure the Postgres,S3, and metastrore EP first in `trino\trino_cfg.yaml`

```
kubectl apply -f trino/trino_cfg.yaml
kubectl apply -f trino.yaml
```

Port forward to see the UI

```
kubectl   port-forward svc/trino 8080  &
```


