---
title: 'Example: Deploying JupyterHealth Exchange on Kubernetes'
downloads:
  - file: ./examples/jhe-example.yaml
    title: Example kubernetes resources
---

This page documents how to deploy the [jupyterhealth-exchange](https://github.com/jupyterhealth/jupyterhealth-exchange) application onto a kubernetes cluster running on AWS. In this case, the associated JupyterHub happened to also be running on AWS, but in a different AWS account.

## Create the Kubernetes Cluster

Define the cluster in a configuration file, `cluster.yml`. Specify values
that are appropriate for your deployment.

```yaml
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig

metadata:
  name: jhe-example
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

### Install ingress-nginx

First, prepare parameters in `ingress-nginx.yaml`:

```yaml
controller:
  service:
    annotations:
      service.beta.kubernetes.io/aws-load-balancer-type: nlb
      service.beta.kubernetes.io/aws-load-balancer-cross-zone-load-balancing-enabled: "true"
  config:
    use-forwarded-headers: "true"
```

Then run the following:

```shell-session
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
helm install nginx-ingress ingress-nginx/ingress-nginx -f ingress-nginx.yml
```

### Install certmanager

```shell-session
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

(install-application)=

### Install the Application

Finally, install the application into the cluster. [jhe-example.yaml](./examples/jhe-example.yaml) is provided as example kubernetes configuration, although you will need to substitute values appropriate for your deployment.

```shell-session
kubectl apply -f jhe-example.yml
```

## Register Hostname

When the application is created, the cluster will create a service object for the nginx ingress controller.

```shell-session
kubectl get service nginx-ingress-ingress-nginx-controller -o jsonpath="{.status.loadBalancer.ingress[0].hostname}"
```

In this kind of deployment, it will have a hostname of the form `{long_string}.elb.{region}.amazonaws.com`. Create a CNAME in your DNS provider that maps the public address of your JupyterHealth Exchange application to the value of this address.

## Create a Database

This is an example configuration for an Amazon RDS PostgreSQL instance. Use values appropriate for your deployment. The VPC-related values would come from identifiers created when the cluster was created.

### Example RDS

```{table} AWS RDS Configuration

| **Parameter**                           | **Value**                                |
| --------------------------------------- | ---------------------------------------- |
| Creation method                         | Standard create                          |
| Engine type                             | PostgreSQL                               |
| Engine version                          | 17.4-R1                                  |
| Templates                               | Dev/Test                                 |
| Availability and durability, deployment | Multi-AZ DB Instance                     |
| DB instance identifier                  | database-2                               |
| Credentials management                  | Self managed, not auto generated         |
| DB instance class                       | Burstable classes, db.t3.small           |
| Storage type                            | General Purpose SSD (gp2)                |
| Allocated storage                       | 100 GiB                                  |
| Enable storage autoscaling              | yes                                      |
| Maximum storage threshold               | 1000 GiB                                 |
| Compute resource                        | Don't connect to an EC2 compute resource |
| VPC                                     | eksctl-jhe-example-cluster/VPC                   |
| DB subnet group                         | use default
| Public access                           | no                                       |
| VPC security group                      | choose existing                          |
| Existing VPC security groups            | default, `eks-cluster-sg-jhe-example-...`           |
| Database authentication                 | password                                 |
| Enable Performance insights             | yes                                      |
| Retention period                        | 7 days (free tier)                       |
| AWS KMS key                             | (default) aws/rds                        |
| Initial database name                   | `jhe`                                    |
```

Note the attributes of the database, some of which are available after it has been created.

```{table} Database Attributes

| Parameter       | Value                            |
| --------------- | -------------------------------- |
| db identifier   | `database-2`                     |
| endpoint        | `database-1...rds.amazonaws.com` |
| port            | `5432`                           |
| master username | `postgres`                       |
| secret value    | (your secret)                    |
| rotation        | `365d`                           |
```

### Test the Database

Launch a shell in the cluster.

```shell-session
$ kubectl run postgres-test -it --rm --image=postgres:17.4 -- bash
If you don't see a command prompt, try pressing enter.
root@postgres-test:/#
```

Use the database endpoint, username, and secret to connect to the database you created.

```shell-session
root@postgres-test:/# psql -h {endpoint} -U {master username} -d jhe
Password for user postgres:
psql (16.3 (Debian 16.3-1.pgdg120+1))
SSL connection (protocol: TLSv1.3, cipher: TLS_AES_256_GCM_SHA384, compression:
off)
Type "help" for help.

postgres=>
```

## Seed Data into the Database

(migrate-database)=

### Migrate Database

Create a Job to migrate the database using our existing ConfigMap.

```{note}
This job uses code from jupyterhealth-exchange software. While the project includes a `Dockerfile`, there is no official docker image for it. An example was built and pushed to `ryanlovett/jupyterhealth-exchange:b793913` representing the latest commit to the jupyterhealth-exchange repo at the time.
```

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
        image: ryanlovett/jupyterhealth-exchange:b793913
        command: ["python", "manage.py", "migrate"]
        envFrom:
        - configMapRef:
            name: jhe-config
```

Run the job.

```shell-session
kubectl apply -f job-manage-migrate.yml
```

### Seed the Database

**((OUTDATED))**

Seed the database (needs update):

```shell-session
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
        image: ryanlovett/jupyterhealth-exchange:b793913
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

```shell-session
kubectl apply -f job-import-seed.yml
```

## Administering JHE

1. Login to your JupyterHealth Exchange app, `https://jhe.example.org/admin/`

1. Under Django OAuth Toolkit, add application

   a. Save the value of `Client id` into the `jhe-config` ConfigMap in `jhe-example.yml` under `OIDC_CLIENT_ID`.

   b. Add space-separated redirect uris for hubs

   - `http://localhost:8000/auth/callback`
   - `https://jupyterhub.example.org/hub/oauth_callback`
   - `https://jupyterhub.example.org/services/smart/oauth_callback`
   - `https://jupyterhub.example.org/user-redirect/smart/oauth_callback`
   - `https://jhe.example.org/auth/callback`

   c. Client type: Public

   d. Authorization grant type: Authorization code

   e. Client secret: {client secret}

   f. Hash client secret: yes

   g. Skip authorization: yes

   h. Algorithm: RSA with SHA-2 256

   i. Create an [RS256 private key](https://django-oauth-toolkit.readthedocs.io/en/latest/oidc.html#creating-rsa-private-key): `openssl genrsa -out oidc.key 4096`

   j. Create a new static PKCE code verifier with the code below. Save it into the `jhe-config` ConfigMap in `jhe-example.yml` under `PATIENT_AUTHORIZATION_CODE_VERIFIER`.

   ```python
   import random
   import string


   def generate_pkce_verifier(length=44):
       characters = string.ascii_letters + string.digits
       return "".join(random.choices(characters, k=length))


   print(generate_pkce_verifier())
   ```

   h. Use the `PATIENT_AUTHORIZATION_CODE_VERIFIER` as the code verifier input at the [Online PKCE Generator Tool](https://tonyxu-io.github.io/pkce-generator/). Save the code challenge into the `jhe-config` ConfigMap in `jhe-example.yml` under `PATIENT_AUTHORIZATION_CODE_CHALLENGE`.

   l. Re-apply the application configuration and restart the application:

   ```shell-session
   kubectl apply -f jhe-example.yml
   kubectl -n jhe delete pod -l app=jhe
   ```

## Authenticating JupyterHub with JHE

In order for users of JupyterHub to have access to JHE,
the simplest way is to use JHE as the OAuth provider for logging into JupyterHub.
To do that, configure.
Below is the configuration to login to JupyterHub with JHE as OAuth provider:

```{literalinclude} ./examples/hub-jhe-auth.yaml
```

You have 3 choices for _authorizing_ JHE users to access the Hub:

1. allow any JHE user to use the Hub. In which case, set:

   ```yaml
   GenericOAuthenticator:
     allow_all: true
   ```

1. allow specific users by email address:

   ```yaml
   GenericOAuthenticator:
     allowed_users:
       - user@example.org
   ```

1. allow based on _organization membership_ in JHE, which requires a bit more configuration.

### Authorizing the Hub via JHE organization

To authorize access to the Hub based on JHE organization membership,
we need to connect JupyterHub groups with JHE organizations.
This lets you manage access to the Hub in the JHE UI
by adding/removing users to the authorized groups.

1. [In JHE] create the organization(s) that you want to grant access to the Hub. Note the integer "organization id" of each organization (they probably look like 2000X).

1. [In JHE] add users to these organizations

1. configure JupyterHub to populate group membership based on JHE organization membership:

   ```{literalinclude} ./examples/hub-jhe-access-groups.yaml
   ```

### Accessing JHE from the Hub

With the above configuration, when a user logs in to the Hub,
two environment variables are set when a user starts their server:

`JHE_URL`
: the URL of the Exchange

`JHE_TOKEN`
: the user's access token for the Exchange

You can use these to make API requests to the Exchange.
There is also the [jupyterhealth-client](xref:jupyterhealth_client) package,
which you can add to your user image:

```shell-session
pip install --pre jupyterhealth-client
```

And then you can use the [JupyterHealthClient](xref:jupyterhealth_client#jupyterhealth_client.JupyterHealthClient) class to fetch patient data.

## Update JupyterHealth Exchange

If you need to periodically update the application:

1. Create a new @migrate-database job. Configure it to use the docker image that contains the updated application. It is safe to perform this step with the existing service still running.
1. After this is complete, update the version of the docker image running in the cluster. For example you can update `jhe-example.yml` and then redeploy by following the @install-application step again.
