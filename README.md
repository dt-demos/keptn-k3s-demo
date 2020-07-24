# Overview

Use Case:

* Simple demo for keptn and bridge setup with simple demo app
* The setup can be use to demo automated SLO valdiation can be performed within a pipeline tool like Azure DevOps or Atlassian bitbucket pipeline

Setup: 
* Keptn 0.7 (https://keptn.sh) on k3s on either AWS EC2 or Azure VM. 
* Simple web app running on the same host as Keptn.  The Docker image is pre-built and can be deployed in k3s or just run as docker
* Dynatrace OneAgent installed on the host
* The demo app is onboarded to keptn as a project called `demo`, one stage called `dev` and on service called `simplenodeservice`

# How to Setup

## 1. dynatrace

* need to have a Dynatrace tenant
* you need to make an api token for keptn installer to use
* NOTE the Keptn installer will create the autotagging rules for keptn_project, keptn_service, keptn_stage based on DT_CUSTOM_PROP values

## 2. Virtual Machine for Keptn


<details>
  <summary>Azure VM</summary>

  ### Add an instance with these settings

  * Ubuntu Server 18.04 LTS
  * Standard D2s v3 (2 vcpus, 8 GiB memory)
  * public inbound ports 80, 443, 22, 8080  (You can add 8080 after VM is running)
  * install the Dynatrace OneAgent on the VM (get commands from within the Dynatrace web UI)

  ### Setup required variables for the keptn-on-k3s installer
  
  *Create Dynatrace API Reference:* https://keptn.sh/docs/0.7.x/monitoring/dynatrace/install/#1-create-a-secret-with-required-credentials
  
  ```
  export DT_TENANT=YOUR TENANT WITHOUT THE HTTPS:// PREFIX (e.g. aaaaaaa.live.dynatrace.com)
  export DT_API_TOKEN=YOUR API TOKEN
  export PUBLIC_IP=$(curl -s http://checkip.amazonaws.com/) && echo "My Public IP = $PUBLIC_IP"
  ```
  
  ### download and run keptn-on-k3s installer script
  
  ```
  curl -Lsf https://raw.githubusercontent.com/keptn-sandbox/keptn-on-k3s/0.7.0/install-keptn-on-k3s.sh | bash -s - --ip $PUBLIC_IP --with-dynatrace --with-jmeter
  ```

</details>

<details>
  <summary>AWS EC2</summary>
  
  ### Add an instance with these settings
  
  * Amazon Linux 2 AMI (HVM), SSD Volume Type
  * t2.xlarge
  * Pick - Auto-assign Public IP 
  * 16 GB storage
  * open port 80, 443, 22, 8080

  ### Setup keptn-on-k3s for No Certificate

  This will make the DNS use xip.ip, for example $PUBLIC_IP.xip.io
  
    ```
    export DT_TENANT=abc12345.live.dynatrace.com
    export DT_API_TOKEN=YOURTOKEN
    curl -Lsf https://raw.githubusercontent.com/keptn-sandbox/keptn-on-k3s/support-for-keptn-0-7/install-keptn-on-k3s.sh | bash -s - --provider aws --with-dynatrace --with-jmeter
    ```

  ### Setup keptn-on-k3s with Certificate

  Assumes you have a Route53 DNS pointing to the public IP. Example FQDN value: jahn-keptn.alliances.dynatracelabs.com

  ```
  export LE_STAGE=production
  export CERT_EMAIL=noreply@dynatrace.com 
  export DT_TENANT=YOUR TENANT WITHOUT THE HTTPS:// PREFIX (e.g. abc12345.live.dynatrace.com)
  export DT_API_TOKEN=YOURTOKEN
  curl -Lsf https://raw.githubusercontent.com/keptn-sandbox/keptn-on-k3s/support-for-keptn-0-7/install-keptn-on-k3s.sh | bash -s - --provider aws --with-dynatrace --with-jmeter --letsencrypt --fqdn YOUR-FQDN
  ```

</details>

## 3. clone repo with demo app resources

```
git clone https://github.com/dt-demos/keptn-k3s-demo.git
cd keptn-k3s-demo
```
  
## 4. Setup Demo app 

<details>
  <summary>Deploy within k3s</summary>

  ### alias for kubectl -- I am lazy or efficient you decide :)

  ```
  alias k='k3s kubectl'
  alias kk='k3s kubectl -n keptn'
  alias kd='k3s kubectl -n dev'
  alias kubectl='k3s kubectl'
  ```

  ### install demo app and verify status and in a browser

  ```
  k apply -f simplenodeapp.yaml
  kd get pods
  kd get svc
  echo "APP_URL = http://$(curl -s http://checkip.amazonaws.com/):8080"
  ```
  
</details>
 
<details>
  <summary>Using Standalone Docker Image </summary>
  
  ### install docker

  Run these commands within the ec2 instance

  ```
  sudo yum update -y
  sudo amazon-linux-extras install docker
  sudo yum install docker
  sudo service docker start
  sudo usermod -a -G docker ec2-user
  docker info
  ```
  *Reference:* https://docs.aws.amazon.com/AmazonECS/latest/developerguide/docker-basics.html

  ### Start Sample Application

  Run this command to run the sample node app on port 8080

  ```
  sudo docker run -d -p 8080:8080 -e DT_CUSTOM_PROP="keptn_project=demo keptn_service=simplenodeservice keptn_stage=dev" dtdemos/simplenodeservice:1 
  ```
  
  ### verify status and in a browser
  
  ```
  echo "APP_URL = http://$(curl -s http://checkip.amazonaws.com/):8080"
  ```

</details>


## 5. Keptn Onboard Application

### Setup required variables (post keptn install) for authorizing keptn CLI

```
export PUBLIC_IP=$(curl -s http://checkip.amazonaws.com/)
export KEPTN_API_URL="https://api.keptn.$PUBLIC_IP.xip.io"
export KEPTN_API_TOKEN=$(k3s kubectl get secret keptn-api-token -n keptn -ojsonpath='{.data.keptn-api-token}'  | base64 --decode )
echo "KEPTN_API_URL   = $KEPTN_API_URL"
echo "KEPTN_API_TOKEN = $KEPTN_API_TOKEN"
keptn auth --api-token "$KEPTN_API_TOKEN" --endpoint "$KEPTN_API_URL"
```

### run manual commands to onboard project and resources files

```
keptn create project demo --shipyard=shipyard.yaml
keptn create service simplenodeservice --project=demo
keptn configure monitoring dynatrace --project=demo
keptn add-resource --project=demo --stage=dev --service=simplenodeservice --resource=slo.yaml --resourceUri=slo.yaml
keptn add-resource --project=demo --stage=dev --service=simplenodeservice --resource=sli.yaml --resourceUri=dynatrace/sli.yaml
```

### generate some app traffic and test quality gate using keptn CLI

```
./sendtraffic.sh "http://localhost:8080" 150
keptn send event start-evaluation --project=demo --stage=dev --service=simplenodeservice
echo "Bridge URL = https://bridge.keptn.$(curl -s http://checkip.amazonaws.com/).xip.io"
keptn configure bridge --output
```
