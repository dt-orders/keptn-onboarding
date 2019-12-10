# keptn-onboarding

* you want keptn 0.6.0.beta2
* running linux or mac in the home directory ~
* assumes installs EKS cluster and domain in route53
* assumes have installed eksctl, kubectl, aws cli, jq
* Dynatrace account with API and PaaS token
* will clone scripts from from these repos:
  * helper scripts -- https://github.com/grabnerandi/keptn-qualitygate-examples.git
  * keptn dynatrace service -- https://github.com/keptn-contrib/dynatrace-service
  * this repo for project files -- https://github.com/keptn-orders/keptn-onboarding.git

### make EKS cluster

```
aws configure

export CLUSTER_NAME=jahn-keptn-orders-cluster
export CLUSTER_REGION=us-west-2
eksctl create cluster --name=$CLUSTER_NAME --node-type=m5.4xlarge --nodes=1 --region=$CLUSTER_REGION --version=1.14
eksctl utils update-coredns --name=$CLUSTER_NAME --region=$CLUSTER_REGION --approve
eksctl utils write-kubeconfig --name=$CLUSTER_NAME --region=$CLUSTER_REGION --set-kubeconfig-context
```

### download and install Keptn CLI

```
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
```

view progess on keptn bridge. example: [https://bridge.keptn.jahn.demo.keptn.sh](https://bridge.keptn.jahn.demo.keptn.sh)

### DT install

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

### if using jmeter, add resource files

```
cd ~/keptn-onboarding/frontend
keptn onboard service frontend --project=keptnorders --chart=./charts
keptn add-resource --project=keptnorders --service=frontend --stage=staging --resource=jmeter/basiccheck.jmx --resourceUri=jmeter/basiccheck.jmx
keptn add-resource --project=keptnorders --service=frontend --stage=staging --resource=jmeter/load.jmx --resourceUri=jmeter/load.jmx
keptn add-resource --project=keptnorders --service=frontend --stage=production --resource=jmeter/basiccheck.jmx --resourceUri=jmeter/basiccheck.jmx
keptn add-resource --project=keptnorders --service=frontend --stage=production --resource=jmeter/load.jmx --resourceUri=jmeter/load.jmx

cd ~/keptn-onboarding/customer
keptn onboard service customer --project=keptnorders --chart=./charts
keptn add-resource --project=keptnorders --service=customer --stage=staging --resource=jmeter/basiccheck.jmx --resourceUri=jmeter/basiccheck.jmx
keptn add-resource --project=keptnorders --service=customer --stage=staging --resource=jmeter/load.jmx --resourceUri=jmeter/load.jmx
keptn add-resource --project=keptnorders --service=customer --stage=production --resource=jmeter/basiccheck.jmx --resourceUri=jmeter/basiccheck.jmx
keptn add-resource --project=keptnorders --service=customer --stage=production --resource=jmeter/load.jmx --resourceUri=jmeter/load.jmx

cd ~/keptn-onboarding/catalog
keptn onboard service catalog --project=keptnorders --chart=./charts
keptn add-resource --project=keptnorders --service=catalog --stage=staging --resource=jmeter/basiccheck.jmx --resourceUri=jmeter/basiccheck.jmx
keptn add-resource --project=keptnorders --service=catalog --stage=staging --resource=jmeter/load.jmx --resourceUri=jmeter/load.jmx
keptn add-resource --project=keptnorders --service=catalog --stage=production --resource=jmeter/basiccheck.jmx --resourceUri=jmeter/basiccheck.jmx
keptn add-resource --project=keptnorders --service=catalog --stage=production --resource=jmeter/load.jmx --resourceUri=jmeter/load.jmx

cd ~/keptn-onboarding/order 
keptn onboard service order --project=keptnorders --chart=./charts
keptn add-resource --project=keptnorders --service=order --stage=staging --resource=jmeter/basiccheck.jmx --resourceUri=jmeter/basiccheck.jmx
keptn add-resource --project=keptnorders --service=order --stage=staging --resource=jmeter/load.jmx --resourceUri=jmeter/load.jmx
keptn add-resource --project=keptnorders --service=order --stage=production --resource=jmeter/basiccheck.jmx --resourceUri=jmeter/basiccheck.jmx
keptn add-resource --project=keptnorders --service=order --stage=production --resource=jmeter/load.jmx --resourceUri=jmeter/load.jmx
```


### send deployment events

```
keptn send event new-artifact --project=keptnorders --service=frontend --image=robjahn/keptn-orders-front-end --tag=1
keptn send event new-artifact --project=keptnorders --service=customer --image=robjahn/keptn-orders-customer-service --tag=1
keptn send event new-artifact --project=keptnorders --service=catalog --image=robjahn/keptn-orders-catalog-service --tag=1
keptn send event new-artifact --project=keptnorders --service=order--image=robjahn/keptn-orders-order-service --tag=1
```

view progess on keptn bridge. example: [https://bridge.keptn.jahn.demo.keptn.sh](https://bridge.keptn.jahn.demo.keptn.sh)

access app @ your domain.  example:  [http://frontend.keptnorders-staging.jahn.demo.keptn.sh/](http://frontend.keptnorders-staging.jahn.demo.keptn.sh/)








