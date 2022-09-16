# CI-jenkins
CI-jenkins is a project repo for creating CI pipeline in Jenkins using SonarQube and Nexus repository.

## Procedure ##

### Step 1 ###
Select a Cloud provider for setting up of server or you can use VirtualBox or VMWare. I am using Google Cloud Platform.
* Create VM instance for Jenkins Server
   * Open GCP and activate cloud shell.
   * Write the following command and click on authorize.
       ``` 
          gcloud auth list
       ```
   * Set default region to `us-central1`
       ```
          gcloud config set compute/region us-central1
       ```
   * Set default zone to `us-central1-a`
       ```
          gcloud config set compute/zone us-central1-a
       ```
   * Create VM Instance `jenkins` to install jenkins-server
       ```
          gcloud compute instances create jenkins \
             --zone=us-central1-a \
             --tags=jenkins-server \
             --machine-type=e2-medium \
             --image-family=debian-11 \
             --image-project=debian-cloud \
             --metadata=startup-script='   #!/bin/bash
   	           sudo apt update
   	           sudo apt install openjdk-8-jdk -y
   	           sudo apt install ca-certificates -y
   	           sudo apt install maven git wget unzip -y
   	           curl -fsSL https://pkg.jenkins.io/debian-stable/jenkins.io.key | sudo tee \
     		          /usr/share/keyrings/jenkins-keyring.asc > /dev/null
   	           echo deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] \
     		          https://pkg.jenkins.io/debian-stable binary/ | sudo tee \
     		          /etc/apt/sources.list.d/jenkins.list > /dev/null
   	           sudo apt-get update
   	           sudo apt-get install jenkins -y'
       ```
   * Create a firewall rule to allow access to port `8080` By default `Jenkins` is launched at this port 
       ```
          gcloud compute firewall-rules create default-allow-http --direction=INGRESS \
             --priority=1000 \
             --network=default \
             --action=ALLOW \
             --rules=tcp:8080 \
             --source-ranges=0.0.0.0/0 \
             --target-tags=jenkins-server
       ```
   * It will take sometime to create `jenkins` VM and install jenkins-server
