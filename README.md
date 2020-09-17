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
* You have [Neoload Web](https://neoload.saas.neotys.com) account. Generate an [API token](https://www.neotys.com/documents/doc/nlweb/latest/en/html/#24270.htm)
* Create NeoLoad Web [Managed Zone for the Load Testing INfrastructure used by Keptn](https://www.neotys.com/documents/doc/nlweb/latest/en/html/#27521.htm#o39043)
              

# Cluster setup

For requirements and example provisioning script, refer to [Keptn docs](https://keptn.sh/docs/0.6.0/installation/setup-keptn/#setup-kubernetes-cluster)

## Google GKE

**NOTE**
  * assumes GKE cluster - **tested with machine name: n1-standard-16 and cluster version: 1.13.11-gke.14**
  * assumes have installed kubectl, gcloud cli, jq
  * python 2.7 (required for Ubuntu 19.04)
* to use a bastion host on [GKE here is a link a documentation](https://github.com/keptn-orders/keptn-orders-setup/blob/master/GOOGLE.md)
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
curl -sL https://get.keptn.sh | sudo -E bash
keptn version
```

### install Keptn

**NOTE: CHOOSE ONE OF THESE NOT BOTH**

```
# for Google GKE cluster
keptn install --keptn-version=release-0.7.1 --use-case=continuous-delivery

# for Amazon EKS cluster
keptn install --keptn-version=release-0.7.1 --use-case=continuous-delivery

# for either cluster 
kubectl -n keptn get pods
```

### update keptn domain

***ONLY REQUIRED FOR AMAZON EKS***

You MUST first take the ELB value and update Route53 Hosted Zone subdomain
```
export DOMAIN=$(kubectl get cm keptn-domain -n keptn -ojsonpath={.data.app_domain})
keptn configure domain $DOMAIN --keptn-version=release-0.7.1
```


### Dynatrace install
1. Configure Dynatrace variables
```
export DT_TENANT=yourtenant.live.dynatrace.com
export DT_API_TOKEN=yourAPItoken
export DT_PAAS_TOKEN=yourPAAStoken
```
2. Create the dynatrace secret 
```
kubectl -n keptn create secret generic dynatrace --from-literal="DT_TENANT=$DT_TENANT" --from-literal="DT_API_TOKEN=$DT_API_TOKEN"  --from-literal="DT_PAAS_TOKEN=$DT_PAAS_TOKEN" --from-literal="KEPTN_API_URL=http://$(kubectl -n keptn get ingress api-keptn-ingress -ojsonpath='{.spec.rules[0].host}')/api" --from-literal="KEPTN_API_TOKEN=$(kubectl get secret keptn-api-token -n keptn -ojsonpath='{.data.keptn-api-token}' | base64 --decode)" --from-literal="KEPTN_BRIDGE_URL=http://$(kubectl -n keptn get ingress api-keptn-ingress -ojsonpath='{.spec.rules[0].host}')/bridge" 
```
3. Deploy OneAgent Operator
```
curl -o deploy-dynatrace-oneagent.sh https://raw.githubusercontent.com/keptn/examples/release-0.7.0/dynatrace-oneagent/deploy-dynatrace-oneagent.sh
chmod +x deploy-dynatrace-oneagent.sh
./deploy-dynatrace-oneagent.sh
```
4. Install Dynatrace integration
```
kubectl apply -f https://raw.githubusercontent.com/keptn-contrib/dynatrace-service/0.8.0/deploy/service.yaml -n keptn
keptn configure monitoring dynatrace
```
## NeoLoad Service install

```
cd ~
git clone --branch 0.7.0 https://github.com/keptn-contrib/neoload-service --single-branch

cd ~/neoload-service/installer/
./defineNeoLoadWebCredentials.sh
./deployNeoLoadWebWithDynatrace.sh keptn
```
### create keptn project

Make new repo with a README file

```
cd ~
git clone  --branch neoload https://github.com/keptn-orders/keptn-onboarding.git

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

### install Dynatrace SLI service

```
kubectl apply -f https://raw.githubusercontent.com/keptn-contrib/dynatrace-sli-service/0.6.0/deploy/service.yaml -n ketpn
keptn configure monitoring dynatrace --project=keptnorders

```

### install NeoLoad SLI service

```
cd ~
git clone --branch 0.7.0 https://github.com/keptn-contrib/neoload-sli-provider --single-branch

cd ~/neoload-sli-provider/installer/
./defineNeoLoadWebCredentials
./deployNeoLoadWeb.sh neoload
```
### Enable NeoLoad sli service withig Keptn Ligthhouse
```
cd ~/keptn-onboarding/
kubectl apply -f lighthouse-source-neoload.yaml
```

### add neoload SLI resources

```
cd ~/keptn-onboarding/frontend
keptn add-resource --project=keptnorders --service=frontend --stage=staging --resource=quality-gates/sli.yaml --resourceUri=neoload/sli.yaml
keptn add-resource --project=keptnorders --service=frontend --stage=production --resource=quality-gates/sli.yaml --resourceUri=neoload/sli.yaml

cd ~/keptn-onboarding/order
keptn add-resource --project=keptnorders --service=order --stage=staging --resource=quality-gates/sli.yaml --resourceUri=neoload/sli.yaml
keptn add-resource --project=keptnorders --service=order --stage=production --resource=quality-gates/sli.yaml --resourceUri=neoload/sli.yaml

cd ~/keptn-onboarding/customer
keptn add-resource --project=keptnorders --service=customer --stage=staging --resource=quality-gates/sli.yaml --resourceUri=neoload/sli.yaml
keptn add-resource --project=keptnorders --service=customer --stage=production --resource=quality-gates/sli.yaml --resourceUri=neoload/sli.yaml

cd ~/keptn-onboarding/catalog
keptn add-resource --project=keptnorders --service=catalog --stage=staging --resource=quality-gates/sli.yaml --resourceUri=neoload/sli.yaml
keptn add-resource --project=keptnorders --service=catalog --stage=production --resource=quality-gates/sli.yaml --resourceUri=neoload/sli.yaml
```



### Add NeoLoad testing resources files

```
cd ~/keptn-onboarding/frontend/neoload/staging
keptn add-resource --project=keptnorders --service=frontend --stage=staging --resource=workload.yaml --resourceUri=workload.yaml
cd ~/keptn-onboarding/frontend/neoload/production
keptn add-resource --project=keptnorders --service=frontend --stage=production --resource=workload.yaml --resourceUri=workload.yaml

cd ~/keptn-onboarding/order/neoload/staging
keptn add-resource --project=keptnorders --service=order --stage=staging --resource=workload.yaml --resourceUri=workloade.yaml
cd ~/keptn-onboarding/order/neoload/production
keptn add-resource --project=keptnorders --service=order --stage=production --resource=workload.yaml --resourceUri=workload.yaml

cd ~/keptn-onboarding/customer/neoload/staging
keptn add-resource --project=keptnorders --service=customer --stage=staging --resource=workload.yaml --resourceUri=workload.yaml
cd ~/keptn-onboarding/customer/neoload/production
keptn add-resource --project=keptnorders --service=customer --stage=production --resource=workload.yaml --resourceUri=workload.yaml

cd ~/keptn-onboarding/catalog/neoload/staging
keptn add-resource --project=keptnorders --service=catalog --stage=staging --resource=workload.yaml --resourceUri=workload.yaml
cd ~/keptn-onboarding/catalog/neoload/production
keptn add-resource --project=keptnorders --service=catalog --stage=production --resource=workload.yaml --resourceUri=workload.yaml
```

### remove the jmeter service
```   
kubectl delete deployment jmeter-service-deployment-distributor -n keptn
```   
### send deployment events

For the first time, ensure that the keptn CLI returns to the prompt without errors and that each 
service completes the deployment into staging before sending next event by monitoring the keptn bridge

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
echo "STAGING    @ https://frontend.keptnorders-staging.$(kubectl get cm keptn-domain -n keptn -ojsonpath={.data.app_domain})"
echo "PRODUCTION @ https://frontend.keptnorders-production.$(kubectl get cm keptn-domain -n keptn -ojsonpath={.data.app_domain})"i\

# services
echo "STAGING CATALOG  @ https://catalog.keptnorders-staging.$(kubectl get cm keptn-domain -n keptn -ojsonpath={.data.app_domain})"
echo "STAGING CUSTOMER @ https://customer.keptnorders-staging.$(kubectl get cm keptn-domain -n keptn -ojsonpath={.data.app_domain})"
echo "STAGING ORDER    @ https://order.keptnorders-staging.$(kubectl get cm keptn-domain -n keptn -ojsonpath={.data.app_domain})"
```






