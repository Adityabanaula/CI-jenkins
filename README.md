# CI-jenkins
CI-jenkins is a project repo for creating CI pipeline in Jenkins using SonarQube and Nexus repository.

## Procedure ##

### Installing Jenkins ###
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
   
### Installing SonarQube ###

* Create VM instance for installing Sonarqube server
    * Creating VM instance `sonarqube`
        ```
           gcloud compute instances create sonarqube \
             --zone=us-central1-a \
             --tags=sonarqube-server \
             --machine-type=e2-medium \
             --image-family=debian-11 \
             --image-project=debian-cloud \
             --metadata=startup-script='   #!/bin/bash
               cp /etc/sysctl.conf /root/sysctl.conf_backup
               cat <<EOT> /etc/sysctl.conf
               vm.max_map_count=262144
               fs.file-max=65536
               ulimit -n 65536
               ulimit -u 4096
               EOT
               cp /etc/security/limits.conf /root/sec_limit.conf_backup
               cat <<EOT> /etc/security/limits.conf
               sonarqube   -   nofile   65536
               sonarqube   -   nproc    409
               EOT
               sudo apt-get update -y
               sudo apt-get install openjdk-11-jdk -y
               sudo update-alternatives --config java
               java -version
               sudo apt update
               wget -q https://www.postgresql.org/media/keys/ACCC4CF8.asc -O - | sudo apt-key add -
               sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt/ `lsb_release -cs`-pgdg main" >> /etc/apt/sources.list.d/pgdg.list'
               sudo apt install postgresql postgresql-contrib -y
               sudo systemctl enable postgresql.service
               sudo systemctl start  postgresql.service
               sudo echo "postgres:admin123" | chpasswd
               runuser -l postgres -c "createuser sonar"
               sudo -i -u postgres psql -c "ALTER USER sonar WITH ENCRYPTED PASSWORD 'admin123';"
               sudo -i -u postgres psql -c "CREATE DATABASE sonarqube OWNER sonar;"
               sudo -i -u postgres psql -c "GRANT ALL PRIVILEGES ON DATABASE sonarqube to sonar;"
               systemctl restart  postgresql
               netstat -tulpena | grep postgres
               sudo mkdir -p /sonarqube/
               cd /sonarqube/
               sudo curl -O https://binaries.sonarsource.com/Distribution/sonarqube/sonarqube-8.3.0.34182.zip
               sudo apt-get install zip -y
               sudo unzip -o sonarqube-8.3.0.34182.zip -d /opt/
               sudo mv /opt/sonarqube-8.3.0.34182/ /opt/sonarqube
               sudo groupadd sonar
               sudo useradd -c "SonarQube - User" -d /opt/sonarqube/ -g sonar sonar
               sudo chown sonar:sonar /opt/sonarqube/ -R
               cp /opt/sonarqube/conf/sonar.properties /root/sonar.properties_backup
               cat <<EOT> /opt/sonarqube/conf/sonar.properties
               sonar.jdbc.username=sonar
               sonar.jdbc.password=admin123
               sonar.jdbc.url=jdbc:postgresql://localhost/sonarqube
               sonar.web.host=0.0.0.0
               sonar.web.port=9000
               sonar.web.javaAdditionalOpts=-server
               sonar.search.javaOpts=-Xmx512m -Xms512m -XX:+HeapDumpOnOutOfMemoryError
               sonar.log.level=INFO
               sonar.path.logs=logs
               EOT
               cat <<EOT> /etc/systemd/system/sonarqube.service
               [Unit]
               Description=SonarQube service
               After=syslog.target network.target
               [Service]
               Type=forking
               ExecStart=/opt/sonarqube/bin/linux-x86-64/sonar.sh start
               ExecStop=/opt/sonarqube/bin/linux-x86-64/sonar.sh stop
               User=sonar
               Group=sonar
               Restart=always
               LimitNOFILE=65536
               LimitNPROC=4096
               [Install]
               WantedBy=multi-user.target
               EOT
               systemctl daemon-reload
               systemctl enable sonarqube.service
               apt-get install nginx -y
               rm -rf /etc/nginx/sites-enabled/default
               rm -rf /etc/nginx/sites-available/default
               cat <<EOT> /etc/nginx/sites-available/sonarqube
               server{
                   listen      80;
                   server_name sonarqube.groophy.in;
                   access_log  /var/log/nginx/sonar.access.log;
                   error_log   /var/log/nginx/sonar.error.log;
                   proxy_buffers 16 64k;
                   proxy_buffer_size 128k;
                   location / {
                       proxy_pass  http://127.0.0.1:9000;
                       proxy_next_upstream error timeout invalid_header http_500 http_502 http_503 http_504;
                       proxy_redirect off;
              
                       proxy_set_header    Host            \$host;
                       proxy_set_header    X-Real-IP       \$remote_addr;
                       proxy_set_header    X-Forwarded-For \$proxy_add_x_forwarded_for;
                       proxy_set_header    X-Forwarded-Proto http;
                   }
               }
               EOT
               ln -s /etc/nginx/sites-available/sonarqube /etc/nginx/sites-enabled/sonarqube
               systemctl enable nginx.service
               sudo ufw allow 80,9000,9001/tcp
               echo "System reboot in 30 sec"
               sleep 30
               reboot'
        ```

### Installing Nessus Repository ###
* Create VM instance for installing Nexus Repository
    * Creating VM instance `nexus`
    ```
       gcloud compute instances create nessus \
          --zone=us-central1-a \
          --tags=nessus-server \
          --machine-type=e2-medium \
          --image-family=debian-11 \
          --image-project=debian-cloud \
          --metadata=startup-script='  #!/bin/bash
              yum install java-1.8.0-openjdk.x86_64 wget -y   
              mkdir -p /opt/nexus/   
              mkdir -p /tmp/nexus/                           
              cd /tmp/nexus/
              NEXUSURL="https://download.sonatype.com/nexus/3/latest-unix.tar.gz"
              wget $NEXUSURL -O nexus.tar.gz
              EXTOUT=`tar xzvf nexus.tar.gz`
              NEXUSDIR=`echo $EXTOUT | cut -d '/' -f1`
              rm -rf /tmp/nexus/nexus.tar.gz
              rsync -avzh /tmp/nexus/ /opt/nexus/
              useradd nexus
              chown -R nexus.nexus /opt/nexus 
              cat <<EOT>> /etc/systemd/system/nexus.service
              [Unit]                                                                          
              Description=nexus service                                                       
              After=network.target                                                            
                                                                  
              [Service]                                                                       
              Type=forking                                                                    
              LimitNOFILE=65536                                                               
              ExecStart=/opt/nexus/$NEXUSDIR/bin/nexus start                                  
              ExecStop=/opt/nexus/$NEXUSDIR/bin/nexus stop                                    
              User=nexus                                                                      
              Restart=on-abort                                                                
                                                                  
              [Install]                                                                       
              WantedBy=multi-user.target                                                      
              EOT
              echo 'run_as_user="nexus"' > /opt/nexus/$NEXUSDIR/bin/nexus.rc
              systemctl daemon-reload
              systemctl start nexus
              systemctl enable nexus'
     ```
