# Overview

Use Case:

* Simple demo for keptn and bridge setup with simple demo app
* The setup can be use to demo automated SLO valdiation can be performed within a pipeline tool like Azure DevOps or Atlassian bitbucket pipeline

Setup: 
* Keptn 0.7 (https://keptn.sh) on k3s on either AWS EC2 or Azure VM. 
* Demo app running on the same host as Keptn.  The Docker image is pre-built and can be deployed in k3s or just run as docker container
* Dynatrace OneAgent installed on the host
* The demo app is onboarded to keptn as a project called `demo`, one stage called `dev` and on service called `simplenodeservice`

# Keptn Setup on K3s

## 1. dynatrace

* need to have a Dynatrace tenant
* you need to make an api token for keptn installer to use
* NOTE the Keptn installer will create the autotagging rules for keptn_project, keptn_service, keptn_stage based on DT_CUSTOM_PROP values

## 2. Provison a host for Keptn

Goto the cloud provider web console and add the VM following the guide below.  You will need to SSH into the host to run all the commands below, so have your key pair ready to use.  In Azure you can use password authentication for the VM, but this is not recommended.

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

# Setup Demo App on the host

## 1. Clone repo with demo app resources

```
git clone https://github.com/dt-demos/keptn-k3s-demo.git
cd keptn-k3s-demo
```
  
## 2. Deploy Demo app 

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
  sudo docker run -d -p 8080:8080 -e DT_CUSTOM_PROP="keptn_project=demo keptn_service=simplenodeservice keptn_stage=dev" grabnerandi/simplenodeservice:1.0.0 
  ```
  
  ### verify status and in a browser
  
  ```
  echo "APP_URL = http://$(curl -s http://checkip.amazonaws.com/):8080"
  ```

</details>


# USE CASES

<details>
  <summary>Use Case #1 - Onboard Demo Application to Keptn manually</summary>

This set of steps prepares the the keptn project from a SSH session within the host.

### 1. Install the Keptn CLI 

Run this script provided by Keptn team and verify the CLI is on version 0.7

```
curl -sL https://get.keptn.sh | sudo -E bash
keptn version
```

### 2. Initialize CLI with your credentials

Setup required variables (post keptn install) for authorizing keptn CLI

```
export PUBLIC_IP=$(curl -s http://checkip.amazonaws.com/)
export KEPTN_API_URL="https://api.keptn.$PUBLIC_IP.xip.io"
export KEPTN_API_TOKEN=$(k3s kubectl get secret keptn-api-token -n keptn -ojsonpath='{.data.keptn-api-token}'  | base64 --decode )
echo "KEPTN_API_URL   = $KEPTN_API_URL"
echo "KEPTN_API_TOKEN = $KEPTN_API_TOKEN"
keptn auth --api-token "$KEPTN_API_TOKEN" --endpoint "$KEPTN_API_URL"
```

### 3. Onboard demo app to Keptn

Run manual commands to onboard project to keptn along with demo app resources files

```
keptn create project demo --shipyard=shipyard.yaml
keptn create service simplenodeservice --project=demo
keptn configure monitoring dynatrace --project=demo
keptn add-resource --project=demo --stage=dev --service=simplenodeservice --resource=slo.yaml --resourceUri=slo.yaml
keptn add-resource --project=demo --stage=dev --service=simplenodeservice --resource=sli.yaml --resourceUri=dynatrace/sli.yaml
```

### 4. Verify demo app onboarding

Open up the Keptn Bridge in a browser and you should now see the project `simplenodeservice` and and a stage `dev`

Use these command to get the URL and credentials:

```
echo "Bridge URL = https://bridge.keptn.$(curl -s http://checkip.amazonaws.com/).xip.io"
keptn configure bridge --output
```

</details>

<details>
  <summary>Use Case #2 - Test the SLO validation without a pipeline</summary>


This assumes you have completed the Keptn Onbaording already. (i.e. USE CASE #1)

### 1. Send load and invoke the keptn `start-evaluation` event using the Keptn CLI

```
./sendtraffic.sh "http://localhost:8080" 150
keptn send event start-evaluation --project=demo --stage=dev --service=simplenodeservice
```

### 2. Open up the Keptn Bridge in a browser and you should now see and monitor the evaluation under the project `simplenodeservice` and and a stage `dev`

Use these command to get the URL and credentials:

```
echo "Bridge URL = https://bridge.keptn.$(curl -s http://checkip.amazonaws.com/).xip.io"
keptn configure bridge --output
```

This set of steps prepares the the keptn project and test the SLO validation all from the SSH session within the host.

</details>

<details>
  <summary>Use Case #3 - Test the SLO validation from a pipeline in Azure Devops</summary>

This assumes you have completed the Keptn Onbaording already. (i.e. USE CASE #1)

This use case uses the Dockerized script from this project for the SLO validation.
https://github.com/keptn-sandbox/keptn-quality-gate-bash

### 1. Create a release pipeline

### 2. Add these as pipeline variables

KeptnApiUrl - https://xx.xx.xx.xx/api
KeptnApiToken
KeptnImage - robjahn/keptn-quality-gate-bash
KeptnProject - demo
KeptnService  - dev
ProcessType - ignore | fail_on_warning | pass_on_warning

### 3. Add a Powershell task

Call the task `Set Test Start Time` with this code

```
$StartTime = (get-date).ToUniversalTime().toString("yyyy-MM-ddTHH:mm:ssZ")

Write-Host "==============================================================="
Write-Host "StartTime: "$StartTime
Write-Host "==============================================================="

Write-Host ("##vso[task.setvariable variable=StartTime]$StartTime")
```

### 4. Add a Bask task

Call the task `Send Traffic` with this code

```
$(System.DefaultWorkingDirectory)/_keptn/sendtraffic.sh
http://IP TO YOUR HOST:8080 10
```


### 5. Add a Powershell task

Call the task `Set Test End Time` with this code

```
$EndTime = (get-date).ToUniversalTime().toString("yyyy-MM-ddTHH:mm:ssZ")
Write-Host "==============================================================="
Write-Host "EndTime:   "$EndTime
Write-Host "==============================================================="

Write-Host ("##vso[task.setvariable variable=EndTime]$EndTime")
```

### 5. Add a Powershell task

Call the task `Keptn Quality Gate` with this code

```
docker run -i --env KEPTN_URL=$(KeptnApiUrl) --env KEPTN_TOKEN=$(KeptnApiToken) --env START=$(StartTime) --env END=$(EndTime) --env PROJECT=$(KeptnProject) --env SERVICE=$(KeptnService) --env STAGE=$(KeptnStage) --env PROCESS_TYPE=$(ProcessType) --env SOURCE=Azure-DevOps --env DEBUG=true --env LABELS='{\"source\":\"Azure-DevOps-Inline\",\"build\":\"$(Release.ReleaseName)\"}' $(KeptnImage)
```

### 6. Open up the Keptn Bridge in a browser and you should now see and monitor the evaluation under the project `simplenodeservice` and and a stage `dev`

Use these command to get the URL and credentials:

```
echo "Bridge URL = https://bridge.keptn.$(curl -s http://checkip.amazonaws.com/).xip.io"
keptn configure bridge --output
```

</details>
