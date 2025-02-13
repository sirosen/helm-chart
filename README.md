# helm-chart
Helm Chart for Deploying funcX stack

[![License](https://img.shields.io/badge/License-Apache%202.0-blue.svg)](https://opensource.org/licenses/Apache-2.0)
[![NSF-2004894](https://img.shields.io/badge/NSF-2004894-blue.svg)](https://nsf.gov/awardsearch/showAward?AWD_ID=2004894)
[![NSF-2004932](https://img.shields.io/badge/NSF-2004932-blue.svg)](https://nsf.gov/awardsearch/showAward?AWD_ID=2004932)

This application includes:
* FuncX Web-Service
* FuncX Websocket Service
* FuncX Forwarder
* Kuberentes endpoint
* Postgres database
* Redis Shared Data Structure
* RabbitMQ broker

## Preliminaries

There are two modes in which funcx-endpoints could be deployed:

1. funcx-endpoint deployed outside k8s, connecting to hosted services in k8s
2. funcx-endpoint deployed inside k8s

### Deploying funcx-endpoint outside of K8s

---
**NOTE**

This only works on Linux systems.

---

Here are the steps to install, preferably into your active conda environment:

```shell script
git clone https://github.com/funcx-faas/funcX.git
cd funcX
git checkout main
pip install funcx_sdk
pip install funcx_endpoint
```

Next create an endpoint configuration:

```shell script
funcx-endpoint
```

Update the endpoint's configuration file to point the endpoint to locally
deployed services, which we will setup in the next sections. If using default
values, the funcx_service_address should be set to http://localhost:5000/v2.

`~/.funcx/default/config.py`

```python
    config = Config(
    executors=[HighThroughputExecutor(
        provider=LocalProvider(
            init_blocks=1,
            min_blocks=0,
            max_blocks=1,
        ),
    )],
    funcx_service_address="http://127.0.0.1:5000/api/v1", # <--- UPDATE THIS LINE
)   
```


### Deploying funcx-endpoint into the K8s deployment

We can deploy the kubernetes endpoint as a pod as part of the chart. It
needs to have a valid copy of the funcx's `funcx_sdk_tokens.json` which can
be created by running on your local workstation and running
```shell script
 funcx-endpoint start
```

You will be prompted to follow the authorization link and paste the resulting
token into the console. Once you do that, funcx-endpoint will create a
`~/.funcx` directory and provide you with a token file.

The Kubernetes endpoint expects this file to be available as a Kubernetes
secret named `funcx-sdk-tokens`.

You can install this secret with:
```shell script
kubectl create secret generic funcx-sdk-tokens \
  --from-file ~/.funcx/credentials/funcx_sdk_tokens.json
```

## Installing FuncX
1. Make a clone of this repository
2. Download subcharts:
    ```shell script
     helm dependency update funcx
    ```
3. Create your own `values.yaml` inside the Git ignored directory `deployed_values/`
4. Obtain Globus Client ID and Secret. These secrets need to exist in the
   correct Globus Auth app. Ask for access to the credentials by contacting
   https://github.com/BenGalewsky or sending a message to the `dev` funcx Slack
   channel. Once you have your credentials, paste them into your `values.yaml`:
    ```yaml
    webService:
      globusClient: <<your app client>>
      globusKey: <<your app secret>>
    ```
5. Install the helm chart:
    ```shell script
    helm install -f deployed_values/values.yaml funcx funcx
    ```
6. You can access your web service through the ingress or via a port forward
to the web service pod. Instructions are provided in the displayed notes.

7. You should be able to see the endpoint registering with the web service
in their respective logs, along with the forwarder log. Check the endpoint's
logs for its ID. 

### Forwarder Debugging

> :warning: *Only for debugging*: You can set the forwarder curve server key manually by creating
  a public/secret curve pair, registering them with kubernetes as a secret and then specifying
  `forwarder.server_cert = true`. By default the forwarder auto generates keys, and distributes it
  to the endpoint during registration.

To manually setup the keys for debugging, here are the steps:

1. Create your curve server public/secret keypair with the create_certs.py script:

    ```shell script
    # CD into the funcx-forwarder repo
    python3 test/create_certs.py -d .curve
    ```

2. Similar to the `funcx-sdk-tokens` add the public/secret pair as kubernetes secret:`funcx-forwarder-secrets` for the forwarder service.
```shell script
# Make sure the server.key_secret file is in your $PWD/.curve dir
kubectl create secret generic funcx-forwarder-secrets --from-file=.curve/server.key --from-file=.curve/server.key_secret
```

3. Once the endpoint is registered to the newly deployed `funcx-forwarder`, make sure to check the `~/.funcx/<ENDPOINT_NAME>/certificates/server.key` file to confirm that the manually added key has been returned to the endpoint.


## Values

> :warning: **USE THE FOLLOWING deployed_values/values.yaml** Omit the
> funcx_endpoint section if using an externally deployed endpoint.

``` yaml
webService:
  pullPolicy: Always
  globusClient: <GLOBUS_CLIENT_ID_STRING>
  globusKey: <GLOBUS_CLIENT_KEY_STRING>
  tag: main

websocketService:
  pullPolicy: Always
  tag: main

# Note that we install numpy into the worker so that we can run tests against the local 
# deployment
# Note that the workerImage needs the same python version as is used in the funcx_endpoint 
# image. This requirement will be relaxed
funcx_endpoint:
  enabled: true
  funcXServiceAddress: http://funcx-funcx-web-service:8000
  image:
    pullPolicy: Always
    tag: main
  maxBlocks: 2
  initBlocks: 0
  minBlocks: 2
  workerInit: pip install funcx-endpoint==0.3.2 numpy
  workerImage: python:3.7-buster

forwarder:
  enabled: true
  tag: main
  pullPolicy: Always

redis:
  master:
    service:
      nodePort: 30379
      type: NodePort

postgresql:
  service:
    nodePort: 30432
    type: NodePort

rabbitmq:
  auth:
    erlangCookie: c00kie
  pullPolicy: Always
```

### Additional config

There are a few values that can be set to adjust the deployed system
configuration

| Value                          | Desciption                                                          | Default           |
| ------------------------------ | ------------------------------------------------------------------- | ----------------- |
| `secrets`                      | Name of a secret deployed into the cluster. Must follow example_secrets.yaml | -        |
| `webService.image`             | Docker image name for the web service                               | funcx/web-service |
| `webService.tag`               | Docker image tag for the web service                                | 213_helm_chart |
| `webService.pullPolicy`        | Kubernetes pull policy for the web service container                | IfNotPresent |
| `webService.loglevel`          | Setting for the App logging level                                   | DEBUG          |
| `webService.advertisedRedisPort` | Redis port that the forwarder (outside of cluster) can reach | 6379 |'
| `webService.advertisedRedisHost` | Redis host that the forwarder (outside of cluster) can reach | localhost |'
| `webService.globusClient`      | Client ID for globus app. Obtained from [http://developers.globus.org](http://developers.globus.org) | |
| `webService.globusKey`         | Secret for globus app. Obtained from [http://developers.globus.org](http://developers.globus.org) | |
| `webService.replicas`          | Number of replica web services to deploy                            | 1 |
| `endpoint.enabled`            | Deploy an internal kubernetes endpoint? | true |
| `endpoint.replicas`            | Number of replica endpoint pods to deploy | 1 |
| `endpoint.image`             | Docker image name for the endpoint                               | funcx/kube-endpoint |
| `endpoint.tag`               | Docker image tag for the endpoint                                | 213_helm_chart |
| `endpoint.pullPolicy`        | Kubernetes pull policy for the endpoint container                | IfNotPresent |
| `forwarder.enabled`            | Deploy an internal kubernetes forwarder? | true |
| `forwarder.minInterchangePort`    | The minimum port to assign interchanges. This will be the first port opened int he pod | 54000 |
| `forwarder.maxInterchangePort`    | The maximum port to assign interchanges. Only the first three ports are opened in the pod | 54002 |
| `forwarder.image`             | Docker image name for the forwarder                               | funcx/funcx/forwarder |
| `forwarder.tag`               | Docker image tag for the forwarder                                | dev |
| `forwarder.pullPolicy`        | Kubernetes pull policy for the forwarder container                | IfNotPresent |
| `ingress.enabled`              | Deploy an ingres to route traffic to web app?                       | false |
| `ingress.host`                 | Host name for the ingress. You will be able to reach your web service via a url that starts with the helm release name and ends with this host | uc.ssl-hep.org |
| `services.postgres.enabled`      | Deploy postgres along with service?                             | true |
| `services.postgres.externalURI`  | If postgres is deployed externally, what URI connects to it?    | sqlite:////sqlite/app.db |
| `services.redis.enabled`         | Deploy redis along with service?                             | true |
| `services.redis.externalHost`  | If redis is deployed externally, what is the host name?    |  |
| `services.redis.externalPort`  | If redis is deployed externally, what is the port?    |  6379 |


## Sealed Secrets
The chart can take advantage of Bitnami's sealed secrets controller to encrypt
sensitive config data so it can safely be checked into the GitHub repo.

Create a secrets.yaml file with these values (the values need to be base64 
encoded (use `echo "secret value" | base64`)). Use the file [example_secrets.yaml](example_secrets.yaml) to see
how the file should be formatted and the names of the secrets.

With kubectl configured to talk to the target cluster, encode the secrets file
with the command:
```console
cat local-dev-secrets.yaml | \
        kubeseal --controller-namespace kube-system \
        --controller-name sealed-secrets \
        --format yaml > local-dev-sealed-secrets.yaml
```

## Subcharts
This chart uses two subcharts to supply dependent services. You can update
settings for these by referenceing the subchart name and values from
their READMEs.

For example
``` yaml
postgresql.postgresqlUsername: funcx
```
| Subchart   | Link to Documentation |
| ---------- | --------------------- |
| postgresql | [https://github.com/bitnami/charts/tree/master/bitnami/postgresql](https://github.com/bitnami/charts/tree/master/bitnami/postgresql) |
| redis      | [https://github.com/bitnami/charts/tree/master/bitnami/redis](https://github.com/bitnami/charts/tree/master/bitnami/redis) |


## Helpful Scripts
It's common for the database to have limited external access. In this case it's
easier to create a busy-box inside the cluster and access plsql internally.

In the scripts directory there is `psql-busybox.yaml`. Create the pod with 
```console
$ kubectl create -f scripts/psql-busybox.yaml
```

You can then create a shell with `kubectl exec -it psql bash`

Inside that shell there is a fun pg sql client which can be invoked with the
same Postgres URL found in the web app's config file (`/opt/funcx/app.conf`)

```console
pgcli postgresql://funcx:XXXXXXXXXXXX@funcx-production-db.XXXXXX.rds.amazonaws.com:5432/funcx
```
