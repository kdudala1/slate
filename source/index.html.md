---
title: API Reference

language_tabs: # must be one of https://github.com/rouge-ruby/rouge/wiki/List-of-supported-languages-and-lexers
  - shell


toc_footers:
  
includes:
  - errors

search: true

code_clipboard: true

meta:
  - name: description
    content: Documentation for Operational Data for VMware Tanzu
---

# Introduction

Welcome to the documentation for Operational Data for VMware Tanzu. This software enables easy collection of consumption information from your Tanzu products to enable you to meet your reporting obligations and ensure compliance with VMware's Terms of Service.

## Compatible Tanzu Products/Platforms

Name | Units of Measure
--------- | -----------
Tanzu Application Platform | Component and Workload vCPUs
Tanzu Mission Control | Total Cluster Capacity in vCPUs
Tanzu Kubernetes Grid | Total Cluster Capacity in vCPUs
Tanzu Application Service* | Application Instances, Physical Cores in Use

<aside class="notice">
*Please note that Tanzu Application Service uses a different collector, the documentation for which can be found here .
</aside>

## What data is collected?
The Operational Data Collector is designed to collect an extremely limited set of data that is indispensable for billing purposes. Currently, the tool collects the following data on an hourly basis:

Metric | Description
--------- | -----------
cluster_capacity_vcpus | The total cluster capacity in vcpus

<aside class="success">
Sensitive classes of data such as personally identifying information (PII) and IP addresses are not collected or stored. Only data required for billing is used.
</aside>


## Where is it stored and sent?
The data is stored locally in a SQLite database and data can be manually or automatically synced with VMware's Operational Data Lake. 

## Inspecting collected data
Data being collected and stored by the Operational Data for VMware Tanzu service can be queried by accessing the `/data` endpoint and generating a csv dump or by directly accessing the backing SQLite database.

# Architecture
The Operational Data for VMware Tanzu service consists of the following components:

* A SQLite database containing hourly consumption data for Tanzu products.
* An agent that syncs data with VMware's Operational Data Lake.
* A Tanzu CLI plugin to facilitate configuration, inspection and emission.

# Installation

A Tanzu CLI plugin that manages all operations pertaining to Operational Data is currently in development. In the meantime, we recommend installing via the provided install script.

```shell
$ .../scripts/tanzu-operational-usage-service-install-script.sh
```

## Prerequisites

* `kubectl`
* `kapp`
* Tanzu CLI with the Telemetry plugin
* Access to the `dev.registry.pivotal.io` registry
* Default `StorageClass` specified for the cluster

<aside class="warning">
A default `StorageClass` must be specified for the cluster and is needed to deploy the SQLite database.
</aside>

## Deploying

>Copy and paste this script and then run `chmod +x` to run.

```shell
#!/bin/bash

echo -e "\n --- Install script for Tanzu Operational Data Service (TODS) --- \n\n"

TODS_VERSION="latest"

echo "TODS needs to create a Secret to pull the image from the 'dev.registry.pivotal.io' registry"
echo "Please enter credentials for the 'dev.registry.pivotal.io' registry"

echo "Username: "
read -r USERNAME

echo "Password: "
read -rs PASSWORD

AUTH=$(echo -n "${USERNAME}:${PASSWORD}" | base64)

DOCKER_CONFIG_JSON=$(cat <<EOF
{
  "auths": {
    "dev.registry.pivotal.io": {
      "username":"$USERNAME",
      "password":"$PASSWORD",
      "auth":"$AUTH"
    }
  }
}
EOF
)

DOCKER_CONFIG_JSON_BASE64=$(echo -n "$DOCKER_CONFIG_JSON" | base64 )

echo -e "\nRegistry credentials set\n\n"

echo "TODS needs an Entitlement Account Number (EAN) to work"
echo -e "This EAN will be on a ConfigMap in the vmware-system-telemetry namespace\n"
echo "Do you want to make the ConfigMap and add the EAN now? (y/n) "
read RESPONSE
if [ "$RESPONSE" = "yes" ] || [ "$RESPONSE" = "y" ]; then
    echo "Do you want to use the default EAN of '987654321'? (y/n): "
    read DEFAULT_EAN
        case $DEFAULT_EAN in
            y|yes)
                echo "Running: tanzu telemetry update --entitlement-account-number 987654321"
                tanzu telemetry update --entitlement-account-number 987654321
                ;;
            n|no)
                echo "Enter the non-default EAN: "
                read NON_DEFAULT_EAN
                echo "Running: tanzu telemetry update --entitlement-account-number $NON_DEFAULT_EAN"
                tanzu telemetry update --entitlement-account-number $NON_DEFAULT_EAN
                ;;
            *)
                echo "Invalid option. No action for EAN taken"
                ;;
        esac
fi

DEPLOYMENT_YAML=$(cat <<EOF
apiVersion: v1
kind: Namespace
metadata:
  name: tanzu-operational-data-service
---
apiVersion: v1
kind: Secret
metadata:
  name: tods-dev-reg-creds
  namespace: tanzu-operational-data-service
type: kubernetes.io/dockerconfigjson
data:
  .dockerconfigjson: $DOCKER_CONFIG_JSON_BASE64
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: tods-service-account
  namespace: tanzu-operational-data-service
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: resource-lister
rules:
  - apiGroups: ["", "metrics.k8s.io"]
    resources: ["nodes", "pods", "namespaces"]
    verbs: ["list", "get"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: tods-resource-lister-binding
subjects:
  - kind: ServiceAccount
    name: tods-service-account
    namespace: tanzu-operational-data-service
roleRef:
  kind: ClusterRole
  name: resource-lister
  apiGroup: rbac.authorization.k8s.io
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: configmap-viewer
  namespace: vmware-system-telemetry
rules:
  - apiGroups: [""]
    resources: ["configmaps"]
    verbs: ["get"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: configmap-viewer-binding
  namespace: vmware-system-telemetry
subjects:
  - kind: ServiceAccount
    name: tods-service-account
    namespace: tanzu-operational-data-service
roleRef:
  kind: Role
  name: configmap-viewer
  apiGroup: rbac.authorization.k8s.io
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: kube-system-namspace-viewer
  namespace: kube-system
rules:
  - apiGroups: [""]
    resources: ["namespaces"]
    verbs: ["get"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: kube-system-namspace-viewer-binder
  namespace: kube-system
subjects:
  - kind: ServiceAccount
    name: tods-service-account
    namespace: tanzu-operational-data-service
roleRef:
  kind: Role
  name: kube-system-namspace-viewer
  apiGroup: rbac.authorization.k8s.io
---
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: tanzu-operational-data-service-pv-claim
  namespace: tanzu-operational-data-service
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Mi
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: tanzu-operational-data-deployment
  namespace: tanzu-operational-data-service
spec:
  serviceName: "tanzu-operational-data-service-api"
  replicas: 1
  selector:
    matchLabels:
      app.kubernetes.io/name: tanzu-operational-data-service
  template:
    metadata:
      name: tanzu-operational-data
      namespace: tanzu-operational-data-service
      labels:
        app.kubernetes.io/name: tanzu-operational-data-service
    spec:
      serviceAccountName: tods-service-account
      containers:
        - name: tanzu-operational-data-service
          image: dev.registry.pivotal.io/tanzu-portfolio-insights/tanzu-usage-service/tanzu-usage-service-exploration:$TODS_VERSION
          imagePullPolicy: Always
          volumeMounts:
            - mountPath: /data
              name: tanzu-operational-data-service-pv-storage
          ports:
            - containerPort: 8080
          env:
            - name: SEND_TO_SUPERCOLLIDER
              value: "True"
            - name: SEND_TO_PROD
              value: "False"
      imagePullSecrets:
        - name: tods-dev-reg-creds
      volumes:
        - name: tanzu-operational-data-service-pv-storage
          persistentVolumeClaim:
            claimName: tanzu-operational-data-service-pv-claim
---
apiVersion: v1
kind: Service
metadata:
  name: tanzu-operational-data-service-api
  namespace: tanzu-operational-data-service
spec:
  selector:
    app.kubernetes.io/name: tanzu-operational-data-service
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8080
EOF
)

echo "$DEPLOYMENT_YAML" > tanzu-operational-data-service-deployment.yaml

echo -e "\nDeploying Tanzu Operational Data Service via kapp\n"

kapp deploy -a tanzu-operational-data-service -f tanzu-operational-data-service-deployment.yaml

rm tanzu-operational-data-service-deployment.yaml

echo -e "\n --- Tanzu Operational Data Service installed successfully --- \n\n"

```

### What the script does
1. Prompts for `Username` and `Password` to the `dev.registry.pivotal.io` registry
   - This is used to generate a `.dockerconfigjson` that is used to make a `Secret` for pulling down the TODS image
2. Prompts to create an Entitlement Account Number (EAN)
   - EAN is used to identify the operational data that is collected and sent back to telemetry
   - EAN is added to a `ConfigMap` in the `vmware-system-telemetry` namespace via `tanzu telemetry update --entitlement-account-number <EAN>`
   - The default EAN is `987654321` and this is the defaulted EAN in the script
3. Deploys TODS
   - Deploys TODS via `kapp deploy -a tanzu-operational-data-service -f tanzu-operational-data-service-deployment.yaml`

### To use the script
1. Copy the script from either
   - The code window here
   - In the repo under `scripts/tanzu-operational-usage-service-install-script.sh`
2. Paste in a location to deploy from
   - i.e. `pbpaste > /path/to/tanzu-operational-usage-service-install-script.sh`
3. Run: `chmod +x /path/to/tanzu-operational-usage-service-install-script.sh` to make the script runnable
4. Point to the Kubernetes cluster that TODS will be deployed on
   - i.e. Set the `KUBECONFIG` environment variable
5. Run the script and follow the prompts

# Usage
The Operational Data for VMware Tanzu service supports the following operations:

1. Viewing hourly grain usage data (`/data`)
2. Obtaining a csv of historical hourly grain usage data
3. Sending hourly grain usage data directly to VMware
4. Viewing and editing the current configuration of the Operational Data Collector
5. Viewing logs for recent operational data events (`/logs`)


## /data
The `/data` endpoint returns a dump of all the metrics in the SQLite database. It returns the data in a CSV format. 

>To see the raw output:

```shell
kubectl get --raw /api/v1/namespaces/tanzu-operational-data-service/services/tanzu-operational-data-service-api/proxy/data
```

>To put the data in a CSV file:

```shell
kubectl get --raw /api/v1/namespaces/tanzu-operational-data-service/services/tanzu-operational-data-service-api/proxy/data > tanzu-operational-data.csv
```
## /info
The `/info` endpoint returns the state of various configuration options:
 - `Sending to Super Collider`: Boolean for TODS to try and send data to Supercollider.
 
```shell
kubectl get --raw /api/v1/namespaces/tanzu-operational-data-service/services/tanzu-operational-data-service-api/proxy/info
```

## /logs
>To see log output

```shell
kubectl logs tanzu-operational-data-deployment-0 -n tanzu-operational-data-service
```


# Authentication

> To authorize, use this code:

```ruby
require 'kittn'

api = Kittn::APIClient.authorize!('meowmeowmeow')
```

```python
import kittn

api = kittn.authorize('meowmeowmeow')
```

```shell
# With shell, you can just pass the correct header with each request
curl "api_endpoint_here" \
  -H "Authorization: meowmeowmeow"
```

```javascript
const kittn = require('kittn');

let api = kittn.authorize('meowmeowmeow');
```

> Make sure to replace `meowmeowmeow` with your API key.

Kittn uses API keys to allow access to the API. You can register a new Kittn API key at our [developer portal](http://example.com/developers).

Kittn expects for the API key to be included in all API requests to the server in a header that looks like the following:

`Authorization: meowmeowmeow`

<aside class="notice">
You must replace <code>meowmeowmeow</code> with your personal API key.
</aside>

# Kittens

## Get All Kittens

```ruby
require 'kittn'

api = Kittn::APIClient.authorize!('meowmeowmeow')
api.kittens.get
```

```python
import kittn

api = kittn.authorize('meowmeowmeow')
api.kittens.get()
```

```shell
curl "http://example.com/api/kittens" \
  -H "Authorization: meowmeowmeow"
```

```javascript
const kittn = require('kittn');

let api = kittn.authorize('meowmeowmeow');
let kittens = api.kittens.get();
```

> The above command returns JSON structured like this:

```json
[
  {
    "id": 1,
    "name": "Fluffums",
    "breed": "calico",
    "fluffiness": 6,
    "cuteness": 7
  },
  {
    "id": 2,
    "name": "Max",
    "breed": "unknown",
    "fluffiness": 5,
    "cuteness": 10
  }
]
```

This endpoint retrieves all kittens.

### HTTP Request

`GET http://example.com/api/kittens`

### Query Parameters

Parameter | Default | Description
--------- | ------- | -----------
include_cats | false | If set to true, the result will also include cats.
available | true | If set to false, the result will include kittens that have already been adopted.

<aside class="success">
Remember — a happy kitten is an authenticated kitten!
</aside>

## Get a Specific Kitten

```ruby
require 'kittn'

api = Kittn::APIClient.authorize!('meowmeowmeow')
api.kittens.get(2)
```

```python
import kittn

api = kittn.authorize('meowmeowmeow')
api.kittens.get(2)
```

```shell
curl "http://example.com/api/kittens/2" \
  -H "Authorization: meowmeowmeow"
```

```javascript
const kittn = require('kittn');

let api = kittn.authorize('meowmeowmeow');
let max = api.kittens.get(2);
```

> The above command returns JSON structured like this:

```json
{
  "id": 2,
  "name": "Max",
  "breed": "unknown",
  "fluffiness": 5,
  "cuteness": 10
}
```

This endpoint retrieves a specific kitten.

<aside class="warning">Inside HTML code blocks like this one, you can't use Markdown, so use <code>&lt;code&gt;</code> blocks to denote code.</aside>

### HTTP Request

`GET http://example.com/kittens/<ID>`

### URL Parameters

Parameter | Description
--------- | -----------
ID | The ID of the kitten to retrieve

## Delete a Specific Kitten

```ruby
require 'kittn'

api = Kittn::APIClient.authorize!('meowmeowmeow')
api.kittens.delete(2)
```

```python
import kittn

api = kittn.authorize('meowmeowmeow')
api.kittens.delete(2)
```

```shell
curl "http://example.com/api/kittens/2" \
  -X DELETE \
  -H "Authorization: meowmeowmeow"
```

```javascript
const kittn = require('kittn');

let api = kittn.authorize('meowmeowmeow');
let max = api.kittens.delete(2);
```

> The above command returns JSON structured like this:

```json
{
  "id": 2,
  "deleted" : ":("
}
```

This endpoint deletes a specific kitten.

### HTTP Request

`DELETE http://example.com/kittens/<ID>`

### URL Parameters

Parameter | Description
--------- | -----------
ID | The ID of the kitten to delete

