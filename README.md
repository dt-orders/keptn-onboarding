# Overview

This is repo contains the helm charts and load testing scripts to deploy the keptn-orders
application. 

Assumptions:
* You provide a Kubernetes cluster
* You want keptn 0.8.0.
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
curl -sL https://get.keptn.sh | KEPTN_VERSION=0.8.0 bash
keptn version
```

### install Keptn

**NOTE: CHOOSE ONE OF THESE NOT BOTH**

```
# for Google GKE cluster
keptn install --keptn-version=release-0.8.0 --endpoint-service-type=ClusterIP --use-case=continuous-delivery

# for Amazon EKS cluster
keptn install --keptn-version=release-0.7.1 --use-case=continuous-delivery --endpoint-service-type=LoadBalancer

# for either cluster 
kubectl -n keptn get pods
```

### update keptn domain

***ONLY REQUIRED FOR AMAZON EKS***

You MUST first take the ELB value and update Route53 Hosted Zone subdomain
```
Get the public ip of your ELB created on your cluster ( EC2/LoadBalancer copy the name. Go to network interface and select the network interface of you ELB to get the public ip).
Update your Root53 DNS to create an alias pointing to your ELB.

Once configured :
KEPTN_ENDPOINT=http://<ENDPOINT_OF_API_GATEWAY>/api
```

#### install Istio
***ONLY REQUIRED FOR AMAZON Google Cloud***
Before installing Keptn you will need to : 
Download the Istio command line tool by following the []official instructions](https://istio.io/latest/docs/setup/install/) or by executing the following steps.
```
curl -L https://istio.io/downloadIstio | ISTIO_VERSION=1.8.2 sh -
```
Check the version of Istio that has been downloaded and execute the installer from the corresponding folder, e.g.,
```
./istio-1.8.2/bin/istioctl install
```
The installation of Istio should be finished within a couple of minutes.
```
This will install the Istio default profile with ["Istio core" "Istiod" "Ingress gateways"] components into the cluster. Proceed? (y/N) y
✔ Istio core installed
✔ Istiod installed
✔ Ingress gateways installed
✔ Installation complete
```
#### Configure Istio and Keptn
Once Keptn has been installed

We are using Istio for traffic routing and as an ingress to our cluster.
 To make the setup experience as smooth as possible we have provided some scripts for your convenience. If you want to run the Istio configuration yourself step by step, please [take a look at the Keptn documentation](https://keptn.sh/docs/0.8.x/operate/install/#option-3-expose-keptn-via-an-ingress).

The first step for our configuration automation for Istio is downloading the configuration bash script from Github:
```
curl -o configure-istio.sh https://raw.githubusercontent.com/keptn/examples/release-0.8.0/istio-configuration/configure-istio.sh
```
After that you need to make the file executable using the chmod command.
```
chmod +x configure-istio.sh
```
Finally, let's run the configuration script to automatically create your Ingress resources.
```
./configure-istio.sh
```
### Connect your Keptn CLI to the Keptn installation
#### FOR EKS
Set the environment variable KEPTN_API_TOKEN:
```
KEPTN_API_TOKEN=$(kubectl get secret keptn-api-token -n keptn -ojsonpath={.data.keptn-api-token} | base64 --decode)
```
To authenticate the CLI against the Keptn cluster, use the keptn auth command:
```
keptn auth --endpoint=$KEPTN_ENDPOINT --api-token=$KEPTN_API_TOKEN
```
#### FOR GKE
```
KEPTN_ENDPOINT=http://$(kubectl -n keptn get ingress api-keptn-ingress -ojsonpath='{.spec.rules[0].host}')/api
KEPTN_API_TOKEN=$(kubectl get secret keptn-api-token -n keptn -ojsonpath='{.data.keptn-api-token}' | base64 --decode)
```
Use this stored information and authenticate the CLI.
```
keptn auth --endpoint=$KEPTN_ENDPOINT --api-token=$KEPTN_API_TOKEN
```
### Authenticate Keptn Bridge
After installling and exposing Keptn, you can access the Keptn Bridge by using a browser and navigating to the Keptn endpoint without the api path at the end of the URL.

The Keptn Bridge has basic authentication enabled by default and the default user is keptn with an automatically generated password.
To get the user and password for authentication, execute:
```
keptn configure bridge --output
```
### Dynatrace install 
1. Deploy the dynatrace OneAgent Operator
To monitor a Kubernetes environment using Dynatrace, please setup dynatrace-operator as described below, or visit the official [Dynatrace documentation](https://www.dynatrace.com/support/help/technology-support/cloud-platforms/kubernetes/deploy-oneagent-k8/).
For setting up dynatrace-operator, perform the following steps:

    1. Log into your Dynatrace environment
    2. Open Dynatrace Hub (on the left hand side, scroll down to Manage and click on Deploy Dynatrace)
    3. Within Dynatrace Hub, search for Kubernetes
    4. Click on Kubernetes, and select Monitor Kubernetes at the bottom of the screen
    5. In the following screen, select the Platform, a PaaS and API Token, and the oenagent installation options (e.g., for GKE you need to enable volume storage).
    6. Copy the generated code and run it in a terminal/bash
    7. Optional: Verify if all pods in the Dynatrace namespace are running. It might take up to 1-2 minutes for all pods to be up and running.
    ```
    kubectl get pods -n dynatrace
    ```
    ```
    NAME                                          READY   STATUS    RESTARTS   AGE
    dynakube-kubemon-0                            1/1     Running   0          11h
    dynatrace-oneagent-operator-cc9856cfd-hrv4x   1/1     Running   0          2d11h
    dynatrace-oneagent-webhook-5d67c9bb76-pz2gh   2/2     Running   0          2d11h
    dynatrace-operator-fb56f7f59-pf5sg            1/1     Running   0          2d11h
    oneagent-gc2lc                                1/1     Running   0          35h
    oneagent-w7msm                                1/1     Running   0          35h
    ```
2. Install Dynatrace integration
    1. The Dynatrace integration into Keptn is handled by the dynatrace-service. To install the dynatrace-service, execute:
     ```
    kubectl apply -f https://raw.githubusercontent.com/keptn-contrib/dynatrace-service/release-0.11.0/deploy/service.yaml -n keptn
     ```
    2. When the service is deployed, use the following command to install Dynatrace on your cluster. If Dynatrace is already deployed, the current deployment of Dynatrace will not be modified.
     ```
    keptn configure monitoring dynatrace
     ```
    The output of the command will tell you what has been set up in your Dynatrace environment:
     ```
    ID of Keptn context: 79f19c36-b718-4bb6-88d5-cb79f163289b
    Dynatrace monitoring setup done.
    The following entities have been configured:
        
    ...
    ---Problem Notification:--- 
      - Successfully set up Keptn Alerting Profile and Problem Notifications
    ...

## NeoLoad Service install

```
cd ~
git clone --branch 0.8.0 https://github.com/keptn-contrib/neoload-service --single-branch

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
kubectl delete deployment jmeter-service -n keptn
```   
### send deployment events

For the first time, ensure that the keptn CLI returns to the prompt without errors and that each 
service completes the deployment into staging before sending next event by monitoring the keptn bridge

```
keptn send event new-artifact --project=keptnorders --service=frontend --image=dtdemos/keptn-orders-front-end --tag=1
keptn send event new-artifact --project=keptnorders --service=customer --image=dtdemos/keptn-orders-customer-service --tag=1
keptn send event new-artifact --project=keptnorders --service=catalog --image=dtdemos/keptn-orders-catalog-service --tag=1
keptn send event new-artifact --project=keptnorders --service=order --image=dtdemos/keptn-orders-order-service --tag=1
```

### Monitor and view application


```
# view progess on keptn bridge
echo "View keptn url $(keptn status})"
echo "View bridge credentials   $(keptn configure bridge --output})"
# view front-end
echo "STAGING    @ https://frontend.keptnorders-staging.$(kubectl get cm keptn-domain -n keptn -ojsonpath={.data.app_domain})"
echo "PRODUCTION @ https://frontend.keptnorders-production.$(kubectl get cm keptn-domain -n keptn -ojsonpath={.data.app_domain})"i\

# services
echo "STAGING CATALOG  @ https://catalog.keptnorders-staging.$(kubectl get cm keptn-domain -n keptn -ojsonpath={.data.app_domain})"
echo "STAGING CUSTOMER @ https://customer.keptnorders-staging.$(kubectl get cm keptn-domain -n keptn -ojsonpath={.data.app_domain})"
echo "STAGING ORDER    @ https://order.keptnorders-staging.$(kubectl get cm keptn-domain -n keptn -ojsonpath={.data.app_domain})"
```






