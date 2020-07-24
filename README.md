# Overview



# Setup

### 1. dynatrace

* need to have a Dynatrace tenant
* you need to make an api token for keptn installer to use
* Keptn installer will create the autotagging rules for keptn_project, keptn_service, keptn_stage based on DT_CUSTOM_PROP values

### 2. Virtual Machine for Keptn

<details>
  <summary>AWS EC2</summary>
  
  #### K3s on EC2 No Certificate
  
  * Amazon Linux 2 AMI (HVM), SSD Volume Type
  * t2.xlarge
  * Pick - Auto-assign Public IP 
  * 16 GB storage
  * open port 80, 443, 22, 8080

  #### K3s on EC2 No Certificate

    ```
    # No Certificate
    export DT_TENANT=abc12345.live.dynatrace.com
    export DT_API_TOKEN=YOURTOKEN
    curl -Lsf https://raw.githubusercontent.com/keptn-sandbox/keptn-on-k3s/support-for-keptn-0-7/install-keptn-on-k3s.sh | bash -s - --provider aws --with-dynatrace --with-jmeter
    ```

  #### K3s on EC2 with Certificate

  Assumes you have a DNS pointing to the public IP. Example FQDN value: jahn-keptn.alliances.dynatracelabs.com

  ```
  export LE_STAGE=production
  export CERT_EMAIL=noreply@dynatrace.com 
  export DT_TENANT=abc12345.live.dynatrace.com
  export DT_API_TOKEN=YOURTOKEN
  curl -Lsf https://raw.githubusercontent.com/keptn-sandbox/keptn-on-k3s/support-for-keptn-0-7/install-keptn-on-k3s.sh | bash -s - --provider aws --with-dynatrace --with-jmeter --letsencrypt --fqdn YOUR-FQDN
  ```

</details>


### 3. install docker

Run these commands within the ec2 instance

```
sudo yum update -y
sudo amazon-linux-extras install docker
sudo yum install docker
sudo service docker start
sudo usermod -a -G docker ec2-user
docker info
```
**Reference:** https://docs.aws.amazon.com/AmazonECS/latest/developerguide/docker-basics.html

### 4. Keptn Onboard Application

```
keptn create project demo --shipyard=shipyard.yaml
keptn create service simplenodeservice --project=demo
```

### 5. Start Sample Application

Run this command to run the sample node app on port 8080

```
sudo docker run -d -p 8080:8080 -e DT_CUSTOM_PROP="keptn_project=demo keptn_service=simplenodeservice keptn_stage=dev" dtdemos/simplenodeservice:1 
```

#### Send traffic

You can run this on the ec2 instance to make traffic to the sample application

```
./sendtraffic.sh http://IP:8080 <number of seconds>
```

### More Notes

* the Azure DevOps **Prepare Keptn environment** task DOES not onboard the service, so you DO NOT need run 'keptn onboard service'.  The task will: 
  * 'keptn create project' - shipyard is created automatically (you pass the stage name and project name)
  * 'keptn create service'
