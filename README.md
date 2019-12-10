# Overview

This is repo contains the helm charts and load testing scripts to deploy the keptn-orders
application. 

Assumptions:
* You provide a Kubernetes cluster
* You want keptn 0.6.0.beta2
* You have [Dynatrace tenant](https://www.dynatrace.com/trial/) with API and PaaS token
* will clone scripts from from these repos to your home directory
  * helper scripts -- https://github.com/grabnerandi/keptn-qualitygate-examples.git
  * keptn dynatrace service -- https://github.com/keptn-contrib/dynatrace-service
  * this repo for project files -- https://github.com/keptn-orders/keptn-onboarding.git

# Cluster setup

For requirements and example provisioning script, refer to [Keptn docs](https://keptn.sh/docs/0.6.0/installation/setup-keptn/#setup-kubernetes-cluster)

## Google GKE

**NOTE**
  * assumes GKE cluster - **tested with machine name: n1-standard-16 and cluster version: 1.13.11-gke.14**
  * assumes have installed kubectl, gcloud cli, jq
  * python 2.7 (required for Ubuntu 19.04)

## Amazon EKS cluster

**NOTE**
  * assumes EKS cluster - **tested with node-type=m5.2xlarge and cluster version 1.14** 
  * sub domain in route53
  * assumes have installed eksctl, kubectl, aws cli, jq

# Scripts to setup Keptn and the keptn-orders application

**NOTE**
Assumes using linux or mac and commands below assume they will be run from the home directory ```~```

### download and install Keptn CLI

```
cd ~
wget https://storage.googleapis.com/keptn-cli/0.6.0.beta2-20191205.1326/keptn-linux.zip
unzip keptn-linux.zip
sudo mv keptn /usr/local/bin/keptn
keptn version
```

### install Keptn

```
keptn install --keptn-version=release-0.6.0.beta2 --platform=eks
kubectl -n keptn get pods
```

### update keptn domain

***ONLY REQUIRED FOR AMAZON EKS***

You MUST first take the ELB value and update Route53 Hosted Zone subdomain
```
export DOMAIN=$(kubectl get cm keptn-domain -n keptn -ojsonpath={.data.app_domain})
keptn configure domain $DOMAIN --keptn-version=release-0.6.0.beta2
```

### expose bridge

```
cd ~
git clone https://github.com/grabnerandi/keptn-qualitygate-examples.git
cd ~/keptn-qualitygate-examples/simpleservice/keptn/
./exposeBridge.sh
echo "View bridge @ https://bridge.keptn.$(kubectl get cm keptn-domain -n keptn -ojsonpath={.data.app_domain})/#/"
```

### Dynatrace install

```
cd ~
git clone --branch 0.5.0 https://github.com/keptn-contrib/dynatrace-service --single-branch

cd ~/dynatrace-service/deploy/scripts
./defineDynatraceCredentials.sh
./deployDynatraceOnEKS.sh
kubectl -n dynatrace get pods -w
```

### create keptn project

Make new repo with a README file

```
cd ~
git clone https://github.com/keptn-orders/keptn-onboarding.git

cd ~/keptn-onboarding
export GIT_USER= <your id>
export GIT_TOKEN= <your personal access token>
export GIT_URL= <eg. https://github.com/robertjahn/keptn060-beta2-keptn-orders>
keptn create project keptnorders --shipyard=./shipyard.yaml --git-user=$GIT_USER --git-token=$GIT_TOKEN --git-remote-url=$GIT_URL
```

### add keptn services to project

```
cd ~/keptn-onboarding
keptn onboard service frontend --project=keptnorders --chart=./frontend/charts
keptn onboard service order --project=keptnorders --chart=./order/charts
keptn onboard service catalog --project=keptnorders --chart=./catalog/charts
keptn onboard service customer --project=keptnorders --chart=./customer/charts
```

### add SLO resources

```
cd ~/keptn-onboarding/frontend
keptn add-resource --project=keptnorders --service=frontend --stage=staging --resource=quality-gates/simple_slo.yaml --resourceUri=slo.yaml
keptn add-resource --project=keptnorders --service=frontend --stage=production --resource=quality-gates/simple_slo.yaml --resourceUri=slo.yaml

cd ~/keptn-onboarding/order
keptn add-resource --project=keptnorders --service=order --stage=staging --resource=quality-gates/simple_slo.yaml --resourceUri=slo.yaml
keptn add-resource --project=keptnorders --service=order --stage=production --resource=quality-gates/simple_slo.yaml --resourceUri=slo.yaml

cd ~/keptn-onboarding/customer
keptn add-resource --project=keptnorders --service=customer --stage=staging --resource=quality-gates/simple_slo.yaml --resourceUri=slo.yaml
keptn add-resource --project=keptnorders --service=customer --stage=production --resource=quality-gates/simple_slo.yaml --resourceUri=slo.yaml

cd ~/keptn-onboarding/catalog
keptn add-resource --project=keptnorders --service=catalog --stage=staging --resource=quality-gates/simple_slo.yaml --resourceUri=slo.yaml
keptn add-resource --project=keptnorders --service=catalog --stage=production --resource=quality-gates/simple_slo.yaml --resourceUri=slo.yaml
```

### add SLO config map

```
cd ~/keptn-onboarding
kubectl apply -f dynatrace-sli-config-keptnorders.yaml
```

### install SLI service

```
cd ~/keptn-qualitygate-examples/simpleservice/keptn
./setupDynatraceSLIService.sh
kubectl -n keptn get pods | grep dynatrace-sli-service
```

### enable SLI service within Keptn Lighthouse

```
cd ~/keptn-qualitygate-examples/simpleservice/keptn
./enableDynatraceSLIForProject.sh keptnorders
```

### Add testing resources files


<details><summary>
Neoload
</summary>

```
# To be filled in
```

</details>


<details><summary>
Jmeter
</summary>

```
cd ~/keptn-onboarding/frontend
keptn add-resource --project=keptnorders --service=frontend --stage=staging --resource=jmeter/basiccheck.jmx --resourceUri=jmeter/basiccheck.jmx
keptn add-resource --project=keptnorders --service=frontend --stage=staging --resource=jmeter/load.jmx --resourceUri=jmeter/load.jmx
keptn add-resource --project=keptnorders --service=frontend --stage=production --resource=jmeter/basiccheck.jmx --resourceUri=jmeter/basiccheck.jmx
keptn add-resource --project=keptnorders --service=frontend --stage=production --resource=jmeter/load.jmx --resourceUri=jmeter/load.jmx

cd ~/keptn-onboarding/customer
keptn add-resource --project=keptnorders --service=customer --stage=staging --resource=jmeter/basiccheck.jmx --resourceUri=jmeter/basiccheck.jmx
keptn add-resource --project=keptnorders --service=customer --stage=staging --resource=jmeter/load.jmx --resourceUri=jmeter/load.jmx
keptn add-resource --project=keptnorders --service=customer --stage=production --resource=jmeter/basiccheck.jmx --resourceUri=jmeter/basiccheck.jmx
keptn add-resource --project=keptnorders --service=customer --stage=production --resource=jmeter/load.jmx --resourceUri=jmeter/load.jmx

cd ~/keptn-onboarding/catalog
keptn add-resource --project=keptnorders --service=catalog --stage=staging --resource=jmeter/basiccheck.jmx --resourceUri=jmeter/basiccheck.jmx
keptn add-resource --project=keptnorders --service=catalog --stage=staging --resource=jmeter/load.jmx --resourceUri=jmeter/load.jmx
keptn add-resource --project=keptnorders --service=catalog --stage=production --resource=jmeter/basiccheck.jmx --resourceUri=jmeter/basiccheck.jmx
keptn add-resource --project=keptnorders --service=catalog --stage=production --resource=jmeter/load.jmx --resourceUri=jmeter/load.jmx

cd ~/keptn-onboarding/order 
keptn add-resource --project=keptnorders --service=order --stage=staging --resource=jmeter/basiccheck.jmx --resourceUri=jmeter/basiccheck.jmx
keptn add-resource --project=keptnorders --service=order --stage=staging --resource=jmeter/load.jmx --resourceUri=jmeter/load.jmx
keptn add-resource --project=keptnorders --service=order --stage=production --resource=jmeter/basiccheck.jmx --resourceUri=jmeter/basiccheck.jmx
keptn add-resource --project=keptnorders --service=order --stage=production --resource=jmeter/load.jmx --resourceUri=jmeter/load.jmx
```
</details>

### send deployment events

```
keptn send event new-artifact --project=keptnorders --service=frontend --image=robjahn/keptn-orders-front-end --tag=1
keptn send event new-artifact --project=keptnorders --service=customer --image=robjahn/keptn-orders-customer-service --tag=1
keptn send event new-artifact --project=keptnorders --service=catalog --image=robjahn/keptn-orders-catalog-service --tag=1
keptn send event new-artifact --project=keptnorders --service=order --image=robjahn/keptn-orders-order-service --tag=1
```

### Monitor and view application


```
# view progess on keptn bridge
echo "View bridge @ https://bridge.keptn.$(kubectl get cm keptn-domain -n keptn -ojsonpath={.data.app_domain})/#/"

# view front-end
echo "STAGING @ https://frontend.keptnorders-staging.$(kubectl get cm keptn-domain -n keptn -ojsonpath={.data.app_domain})"
echo "PRODUCTION @ https://frontend.keptnorders-production.$(kubectl get cm keptn-domain -n keptn -ojsonpath={.data.app_domain})"







