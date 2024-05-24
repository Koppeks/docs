---
title: Deploying Layer5 Cloud
description: "Layer5 Cloud is a collection of services that can be deployed on-premises using Helm."
categories: [Self-Hosted]
weight: 1
---

## High-level List of Deployment Tasks

<ol>
    <li>Review the prequisites for installing Layer5 Cloud on Kubernetes. (<a href="#prerequisites">docs</a>)</li>
    </li>
    <li>Install Layer5 Cloud on Kubernetes using Helm. Deploy it's services in Kubernetes in-cluster. (<a href="#installation">docs</a>)</li>
    <li>Meshery deployments are separate from <a href="https://docs.meshery.io/extensibility/providers">Remote Provider</a> deployments (Layer5 Cloud). Deploy Meshery in Kubernetes in-cluster (or out-of-cluster). (<a href="https://docs.meshery.io/installation/quick-start">docs</a>)</li>
    <li>Configure Meshery Server point to your Remote Provider. Learn more about the Meshery Server registration process with Remote Providers. (<a href="https://docs.meshery.io/extensibility/providers#meshery-server-registration">docs</a>)</li>
</ol>

### Kubernetes-based Installation with Helm

Layer5 offers on-premises installation of its [Meshery Remote Provider](https://docs.meshery.io/extensibility/providers): Layer5 Cloud. Contained in the [Layer5 Helm repository](https://docs.layer5.io/charts) is one chart with two subcharts (see repo [index](https://docs.layer5.io/charts/index.yaml)).

#### Prerequisites

##### 1. Prepare a Persistent Volume

A persistent volume to store the Postgres database is necessary to prepare prior to deployment. If your target cluster does not have a persistent volume readily available (or not configured for automatic PV provisioning and binding of PVCs to PV), we suggest to apply the following configuration to your cluster.

```bash
kubectl apply -f install/kubernetes/persistent-volume.yaml
```

##### 2. Prepare a dedicated namespace and add the chart repo to your helm configuration

*You may choose to use an alternative namespace, but the following instructions assume the use of `layer5` namespace.*

```bash
kubectl create ns layer5
helm repo add layer5 https://docs.layer5.io/charts
```

##### 3. Ensure NGINX Ingress Controller is deployed

*You may chose to use an alternative ingress controller, but the following instructions assume the use of NGINX Ingress Controller.*

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.8.2/deploy/static/provider/cloud/deploy.yaml
```

#### Installation

The first service to install is the Postgres database. The following command installs the Postgres database and initializes it's dataset. The dataset is used by the Layer5 Cloud server and the Layer5 Cloud identity provider.

Layer5 Cloud `postgres` instance requires `pg_cron` extension to be enabled and configured in the `postgres` database to schedule PostgreSQL commands. 
The cloud instance is bundled with required migrations to schedule the jobs with the required commands/sql queries.

__NOTE: Configuring the extension in some other database will result in failure to apply SQL migrations.__

##### 1. Install Postgres database

```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
helm install postgresql bitnami/postgresql --version 14.0.1
```

##### 2. Install Remote Provider Server and Identity Provider

```bash
## TBD: Delete local filesystem reference
# helm install -f ./install/kubernetes/values.yaml cloud ./install/kubernetes -n <namespace>`

helm install -f ./install/helm-chart-values/layer5-cloud-values.yaml cloud ./install/kubernetes -n postgres \
--set-file 'kratos.kratos.emailTemplates.recovery.valid.subject'=<path to the email templates to override>/valid/email-recover-subject.body.gotmpl \
--set-file 'kratos.kratos.emailTemplates.recovery.valid.body'=<path to the email templates to override>/valid/email-recover.body.gotmpl \
--set-file 'kratos.kratos.emailTemplates.verification.valid.subject'=<path to the email templates to override>/valid/email-verify-subject.body.gotmpl \
--set-file 'kratos.kratos.emailTemplates.verification.valid.body'=<path to the email templates to override>/valid/email-verify.body.gotmpl
```

##### 3. Create an OAuth 2.0 client
1. Port forward the Hydra Admin service.
2. ```bash
    hydra clients create \
    --endpoint <port forwarded endpoint> \
    --id meshery-cloud \ <--- ensure the id specified matches with the env.oauthclientid in values.yaml
    --secret some-secret \ <--- ensure the secret specified matches with the env.oauthsecret in values.yaml
    --grant-types authorization_code,refresh_token,client_credentials,implicit \
    --response-types token,code,id_token \
    --scope openid,offline,offline_access \
    --callbacks <Layer5 Cloud host>/callback 
    ```

## Uninstalling the Chart

    
