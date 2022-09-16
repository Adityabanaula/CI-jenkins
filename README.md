# CI-jenkins
CI-jenkins is a project repo for creating CI pipeline in Jenkins using SonarQube and Nexus repository.

## Procedure ##

### Step 1 ###

Create EC2 instance with Ubuntu18, machine type t2.medium and user-data from jenkins.sh

`#!/bin/bash`
`sudo apt update`
`sudo apt install openjdk-8-jdk -y`
`sudo apt install ca-certificates -y`
`sudo apt install maven git wget unzip -y`
`curl -fsSL https://pkg.jenkins.io/debian-stable/jenkins.io.key | sudo tee \`
`  /usr/share/keyrings/jenkins-keyring.asc > /dev/null`
`echo deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] \`
`  https://pkg.jenkins.io/debian-stable binary/ | sudo tee \`
`  /etc/apt/sources.list.d/jenkins.list > /dev/null`
`sudo apt-get update`
`sudo apt-get install jenkins -y`

