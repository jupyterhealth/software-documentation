---
title: "Example: Deploying JupyterHealth Exchange on Kubernetes"
downloads:
  - file: ./examples/jhe-example.yml
    title: Example kubernetes resources
---

This page documents how to deploy the [jupyterhealth-exchange](https://github.com/the-commons-project/jupyterhealth-exchange) application onto a kubernetes cluster running on AWS. In this case, the associated JupyterHub happened to also be running on AWS, but in a different AWS account.

## Create the Kubernetes Cluster

Define the cluster in a configuration file, `cluster.yml`. Specify values
that are appropriate for your deployment.

```yaml
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig

metadata:
  name: jhe
  region: us-east-2
  version: '1.30'

vpc:
  clusterEndpoints:
    publicAccess: true
    privateAccess: true
  nat:
    gateway: Single

nodeGroups:
  - name: public-nodes
    instanceType: t2.micro
    desiredCapacity: 2
    privateNetworking: false

managedNodeGroups:
  - name: system-nodes
    instanceType: t2.small
    privateNetworking: true
    minSize: 1
    maxSize: 3
```

The configuration will be provided to `eksctl` which in this case had access to the following environment variables:

  - `AWS_SECRET_ACCESS_KEY`
  - `AWS_ACCESS_KEY_ID`
  - `AWS_DEFAULT_REGION`

Create the cluster:
```shell
eksctl create cluster -f cluster.yml
```

## Install Cluster Components

Install ingress-nginx.
```shell
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
helm install nginx-ingress ingress-nginx/ingress-nginx -f ingress-nginx.yml
```

Install certmanager.

```shell
helm repo add jetstack https://charts.jetstack.io
helm repo update
helm install \
  cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --version v1.15.4 \
  --set crds.install=true \
  --set installCRDs=true \
  --wait
```

## Create a Database

This is an example configuration for an Amazon RDS PostgreSQL instance. Use values appropriate for your deployment. The VPC-related values would come from identifiers created when the cluster was created.

### Example RDS

:::{table} AWS RDS Configuration
| **Parameter** | **Value** |
|---------------|-----------|
| Creation method | Standard create |
| Engine type | PostgreSQL |
| Engine version | 16.3-R3 |
| Templates | Dev/Test |
| Availability and durability, deployment | Multi-AZ DB Instance |
| DB instance identifier | jhe-db-staging-1 |
| Credentials management | Self managed, not auto generated |
| DB instance class | Burstable classes, db.t3.small |
| Storage type | General Purpose SSD (gp2) |
| Allocated storage | 100 GiB |
| Enable storage autoscaling | yes |
| Maximum storage threshold | 1000 GiB |
| Compute resource | Don't connect to an EC2 compute resource |
| VPC | eksctl-jhe-cluster/VPC |
| DB subnet group | create new db subnet group |
| Public access | no |
| VPC security group | choose existing |
| Existing VPC security groups | default, `eks-cluster-jhe-...` |
| Database authentication | password |
| Enable Performance insights | yes |
| Retention period | 7 days (free tier) |
| AWS KMS key | (default) aws/rds |
| Initial database name | `jhe` |
:::

Note the attributes of the database, e.g.

:::{table} Database Attributes
| Parameter | Value |
|-----------|-------|
| db identifier | `database-1` |
| endpoint | `database-1...rds.amazonaws.com` |
| port | `5432` |
| master username | `postgres` |
| secret value | (your secret) |
| rotation | `365d` |
:::

### Test the Database

Launch a shell in the cluster.
```shell
$ kubectl run postgres-test -it --rm --image=postgres:16.3 -- bash
If you don't see a command prompt, try pressing enter.
root@postgres-test:/# 
```

Use the database endpoint, username, and secret to connect to the database you created.
```shell
root@postgres-test:/# psql -h {endpoint} -U {master username} -d postgres
Password for user postgres:
psql (16.3 (Debian 16.3-1.pgdg120+1))
SSL connection (protocol: TLSv1.3, cipher: TLS_AES_256_GCM_SHA384, compression:
off)
Type "help" for help.

postgres=>
```

## Seed Data into the Database

### Migrate Database

Create a Job to migrate the database using our existing ConfigMap.

:::{note}
This job uses code from jupyterhealth-exchange software. While the project includes a `Dockerfile`, there is no official docker image for it. An example was built and pushed to `ryanlovett/jupyterhealth-exchange:a30ad58` representing the latest commit to the jupyterhealth-exchange repo at the time.
:::

```yaml
# job-manage-migrate.yml
apiVersion: batch/v1
kind: Job
metadata:
  name: jhe-manage-migrate
  namespace: jhe
spec:
  template:
    metadata:
      name: jhe-manage-migrate
    spec:
      restartPolicy: Never
      containers:
      - name: jhe-manage-migrate
        image: ryanlovett/jupyterhealth-exchange:a30ad58
        command: ["python", "manage.py", "migrate"]
        envFrom:
        - configMapRef:
            name: jhe-config
```

Run the job.
```shell
kubectl apply -f job-manage-migrate.yml
```

### Seed the Database

This requires the [seed.sql](https://github.com/the-commons-project/jupyterhealth-exchange/blob/main/db/seed.sql) file from the [jupyterhealth-exchange repository](https://github.com/the-commons-project/jupyterhealth-exchange), and a new python script, `jhe/scripts/seed.py` to import it. `seed.py` is currently available in a [pull request to jupyterhealth-exchange](https://github.com/the-commons-project/jupyterhealth-exchange/pull/37/files).

Injest them as ConfigMaps by running the following commands from within the working directory of [jupyterhealth-exchange](https://github.com/the-commons-project/jupyterhealth-exchange).


```shell
kubectl -n jhe create configmap db-seed-sql --from-file=db/seed.sql
kubectl -n jhe create configmap jhe-scripts-seed.py --from-file=jhe/scripts/seed
.py
```

Create a Job to seed the database.

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: import-seed
  namespace: jhe
spec:
  template:
    metadata:
      name: import-seed
    spec:
      containers:
      - name: import-seed
        image: ryanlovett/jupyterhealth-exchange:a30ad58
        command: ["python", "/app/seed.py"]
        envFrom:
        - configMapRef:
            name: jhe-config
        volumeMounts:
        - name: seed-sql
          mountPath: /app/seed.sql
          subPath: seed.sql
        - name: seed-py
          mountPath: /app/seed.py
          subPath: seed.py
      restartPolicy: Never
      volumes:
      - name: seed-sql
        configMap:
          name: db-seed-sql
      - name: seed-py
        configMap:
          name: jhe-scripts-seed.py
```

and run it
```shell
kubectl apply -f job-import-seed.yml
```

## Install the Application

Finally, install the application into the cluster. [jhe-example.yml](examples/jhe-example.yml) is provided as example kubernetes configuration, although you will need to substitute values appropriate for your deployment.

```shell
kubectl apply -f jhe-example.yml
```

## Administering JHE

1. Login to your JupyterHealth Exchange app, https://jhe.example.org/admin/
1. Under Django OAuth Toolkit, add application

   a. Save Client id

   b. Add space-separated redirect uris for hubs
      - http://localhost:8000/auth/callback
      - https://jupyterhub.example.org/hub/oauth_callback
      - https://jupyterhub.example.org/services/smart/oauth_callback
      - https://jupyterhub.example.org/user-redirect/smart/oauth_callback
      - https://jhe.example.org/auth/callback

   c. Client type: Public

   d. Authorization grant type: Authorization code

   e. Client secret: {client secret}

   f. Hash client secret: yes

   g. Skip authorization: yes

   h. Algorithm: RSA with SHA-2 256

