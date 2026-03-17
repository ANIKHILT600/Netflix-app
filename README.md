# Netflix-app

This guide provides a comprehensive walkthrough for deploying a Netflix clone application using modern DevOps practices. The deployment pipeline leverages Jenkins for continuous integration and continuous delivery (CI/CD), Docker for containerization, and Kubernetes for orchestration. Infrastructure and application metrics are monitored in real-time using Prometheus, Grafana, and Node Exporter, enabling proactive system observability and performance management.



# Overview

Step 1 — Launch an Ubuntu(22.04) T2 Large Instance

Step 2 — Install Jenkins, Docker and Trivy. Create a Sonarqube Container using Docker.

Step 3 — Create a TMDB API Key.

Step 4 — Install Prometheus and Grafana On the new Server.

Step 5 — Install the Prometheus Plugin and Integrate it with the Prometheus server.

Step 6 — Email Integration With Jenkins and Plugin setup.

Step 7 — Install Plugins like JDK, Sonarqube Scanner, Nodejs, and OWASP Dependency Check.

Step 8 — Create a Pipeline Project in Jenkins using a Declarative Pipeline

Step 9 — Install OWASP Dependency Check Plugins

Step 10 — Docker Image Build and Push

Step 11 — Kubernetes master and slave setup on Ubuntu (20.04)

Step 12 — Access the Netflix app on the Browser.

Step 13 — Complete pipeline script


# Step by step guide

## STEP1 : Launch an Ubuntu(24.04) T2 Large Instance
- Launch an AWS EC2 "Jenkins-Sonar"
- Instance type : T2 Large Instance. 
- Use the image as Ubuntu. 
- You can create a new key pair or use an existing one. 
- Security Group Ports to open (Inbound):
```
80 (HTTP)
443 (HTTPS)
8080 (Jenkins)
9000 (SonarQube)
9092 (SonarQube database)
```
- add at userdata
```
#!/bin/bash

exec > /var/log/user-data.log 2>&1

echo "===== DevOps Stack Installation Started ====="

###################################
# Wait for APT lock (very important)
###################################

while fuser /var/lib/dpkg/lock >/dev/null 2>&1 ; do
   echo "Waiting for apt lock..."
   sleep 5
done

###################################
# Update system
###################################

apt-get update -y

###################################
# Install base packages
###################################

apt-get install -y \
curl \
wget \
git \
unzip \
gnupg \
lsb-release \
software-properties-common \
apt-transport-https

###################################
# Install Java 17
###################################

echo "Installing Java"

apt-get install -y openjdk-17-jdk

java -version

###################################
# Clean Docker conflicts
###################################

apt-get remove -y docker docker-engine docker.io containerd runc || true

###################################
# Install Docker
###################################

echo "Installing Docker"

apt-get install -y docker.io

systemctl enable docker
systemctl start docker

usermod -aG docker ubuntu
usermod -aG docker jenkins

docker --version

###################################
# Install Trivy
###################################

echo "Installing Trivy"

wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key \
| gpg --dearmor -o /usr/share/keyrings/trivy.gpg

echo "deb [signed-by=/usr/share/keyrings/trivy.gpg] https://aquasecurity.github.io/trivy-repo/deb $(lsb_release -sc) main" \
> /etc/apt/sources.list.d/trivy.list

apt-get update -y
apt-get install -y trivy

trivy --version

###################################
# Run SonarQube
###################################

echo "Starting SonarQube container"

docker run -d \
--name sonarqube \
-p 9000:9000 \
sonarqube:lts-community

###################################
# Install Jenkins
###################################

echo "Installing Jenkins"

# FIX: Updated GPG Key from jenkins.io-2023.key to jenkins.io-2026.key
curl -fsSL https://pkg.jenkins.io/debian-stable/jenkins.io-2026.key \
| gpg --dearmor -o /usr/share/keyrings/jenkins.gpg

echo "deb [signed-by=/usr/share/keyrings/jenkins.gpg] https://pkg.jenkins.io/debian-stable binary/" \
> /etc/apt/sources.list.d/jenkins.list

apt-get update -y
apt-get install -y jenkins

systemctl enable jenkins
systemctl start jenkins

sleep 30

echo "======================================"
echo "Jenkins Password:"
cat /var/lib/jenkins/secrets/initialAdminPassword
echo "======================================"

echo "DevOps Stack Installed Successfully"

echo "Logs available at: /var/log/user-data.log"
echo "=========================================="
```

## Step 2 : Verify installation Jenkins, Docker and Trivy

Connect to your console, and enter these commands to verify tools.

Optional : You can set server hostname to easily undertand as below:
```
sudo hostnamectl set-hostname Jenkins-sonar
exec bash
```

### 2A — Run this single command to verify all installations at once:
```
echo "=== Java ===" && java --version && \
echo "=== Jenkins ===" && sudo systemctl is-active jenkins && \
echo "=== Docker ===" && docker --version && sudo systemctl is-active docker && \
echo "=== Trivy ===" && trivy --version && \
echo "=== SonarQube ===" && docker ps | grep sonarqube && \
echo "All services verified successfully!"
```
### 2B — Quick Troubleshooting (Optional)
```
# Start Jenkins
sudo systemctl start jenkins
sudo systemctl enable jenkins

# Start Docker
sudo systemctl start docker
sudo systemctl enable docker

# Check logs for errors
sudo journalctl -u jenkins -n 50
sudo journalctl -u docker -n 50
docker logs sonarqube
```

### 2C — After verification, access your services:

Jenkins: http://Jenkins-Sonar_public_IP:8080

SonarQube: http://Jenkins-Sonar_public_IP:9000


## Step 3: Create a TMDB API Key
Next, we will create a TMDB API key

- Open a new tab in the Browser and search for TMDB

- Click on the first result, you will see

- Click on the Login on the top right

- You need to create an account here

- once you create an account you will see this page.


Let’s create an API key, By clicking on your profile and clicking settings.

- Now click on API from the left side panel.

- Click on create

- Click on Developer

- Accept the terms and conditions.

- Provide basic details

- Click on submit and you will get your API key.



## Step 4 : Install Prometheus and Grafana On the new Server

### 4A - Launch an Ubuntu(24.04) T2 Large Instance
- Launch an AWS EC2 "Prometheus-Grafana"
- Instance type : T2 Large Instance. 
- Use the image as Ubuntu. 
- You can create a new key pair or use an existing one. 
- Security Group Ports to open (Inbound):
```
80 (HTTP)
443 (HTTPS)
3000 (Grafana)
9090 (Prometheus)
9100 (Node Exporter)
```
### 4B — Create Prometheus System User
Optional : You can set server hostname to easily undertand as below:
```
sudo hostnamectl set-hostname Prometheus-Grafana
exec bash
```

First of all, let’s create a dedicated Linux user sometimes called a system account for Prometheus. Having individual users for each service serves two main purposes:

It is a security measure to reduce the impact in case of an incident with the service.

It simplifies administration as it becomes easier to track down what resources belong to which service.

To create a system user or system account, run the following command:
```
sudo useradd --system --no-create-home --shell /bin/false prometheus
```

– system - Will create a system account. –no-create-home - We don’t need a home directory for Prometheus or any other system accounts in our case. –shell /bin/false - It prevents logging in as a Prometheus user. Prometheus - Will create a Prometheus user and a group with the same name.

### 4C — Install & Configure Prometheus
Let’s check the latest version of Prometheus from the download page. 

1. You can use the curl or wget command to download Prometheus:
```
wget https://github.com/prometheus/prometheus/releases/download/v2.47.1/prometheus-2.47.1.linux-amd64.tar.gz
```

2. Then, we need to extract all Prometheus files from the archive:
```
tar -xvf prometheus-2.47.1.linux-amd64.tar.gz
```

3. Usually, you would have a disk mounted to the data directory. For this tutorial, I will simply create a /data directory. Also, you need a folder for Prometheus configuration files:
```
sudo mkdir -p /data /etc/prometheus
```

4. Now, let’s change the directory to Prometheus and move some files:
```
cd prometheus-2.47.1.linux-amd64/
```

5. First of all, let’s move the Prometheus binary and a promtool to the /usr/local/bin/. promtool is used to check configuration files and Prometheus rules:
```
sudo mv prometheus promtool /usr/local/bin/
```

6. Optionally, we can move console libraries to the Prometheus configuration directory. Console templates allow for the creation of arbitrary consoles using the Go templating language. You don’t need to worry about it if you’re just getting started:
```
sudo mv consoles/ console_libraries/ /etc/prometheus/
```

7. Finally, let’s move the example of the main Prometheus configuration file:
```
sudo mv prometheus.yml /etc/prometheus/prometheus.yml
```

8. To avoid permission issues, you need to set the correct ownership for the /etc/prometheus/ and data directory:
```
sudo chown -R prometheus:prometheus /etc/prometheus/ /data/
```

9. You can delete the archive and a Prometheus folder when you are done:
```
cd
rm -rf prometheus-2.47.1.linux-amd64.tar.gz
```

10. Verify that you can execute the Prometheus binary by running the following command:
```
prometheus --version
```

11. To get more information and configuration options, run Prometheus Help.
```
prometheus --help
```
We’re going to use some of these options in the service definition.

12. We’re going to use Systemd, which is a system and service manager for Linux operating systems. For that, we need to create a Systemd unit configuration file:
```
sudo vim /etc/systemd/system/prometheus.service
```
Copy below content to Prometheus.service
```
[Unit]
Description=Prometheus
Wants=network-online.target
After=network-online.target
StartLimitIntervalSec=500
StartLimitBurst=5
[Service]
User=prometheus
Group=prometheus
Type=simple
Restart=on-failure
RestartSec=5s
ExecStart=/usr/local/bin/prometheus \
  --config.file=/etc/prometheus/prometheus.yml \
  --storage.tsdb.path=/data \
  --web.console.templates=/etc/prometheus/consoles \
  --web.console.libraries=/etc/prometheus/console_libraries \
  --web.listen-address=0.0.0.0:9090 \
  --web.enable-lifecycle
[Install]
WantedBy=multi-user.target
```
Note:

Let’s go over a few of the most important options related to Systemd and Prometheus: 
- Restart - Configures whether the service shall be restarted when the service process exits, is killed, or a timeout is reached. 
- RestartSec - Configures the time to sleep before restarting a service. User and Group - Are Linux user and a group to start a Prometheus process. 
- config.file=/etc/prometheus/prometheus.yml - Path to the main Prometheus configuration file. Complete file look like below:
```
# my global config
global:
  scrape_interval: 15s # Set the scrape interval to every 15 seconds. Default is every 1 minute.
  evaluation_interval: 15s # Evaluate rules every 15 seconds. The default is every 1 minute.
  # scrape_timeout is set to the global default (10s).

# Alertmanager configuration
alerting:
  alertmanagers:
    - static_configs:
        - targets:
          # - alertmanager:9093

# Load rules once and periodically evaluate them according to the global 'evaluation_interval'.
rule_files:
  # - "first_rules.yml"
  # - "second_rules.yml"

# A scrape configuration containing exactly one endpoint to scrape:
# Here it's Prometheus itself.
scrape_configs:
  # The job name is added as a label `job=<job_name>` to any timeseries scraped from this config.
  - job_name: "prometheus"

    # metrics_path defaults to '/metrics'
    # scheme defaults to 'http'.

    static_configs:
      - targets: ["localhost:9090"]
```
- storage.tsdb.path=/data - Location to store Prometheus data. 
- web.listen-address=0.0.0.0:9090 - Configure to listen on all network interfaces. 
- In some situations, you may have a proxy such as nginx to redirect requests to Prometheus. In that case, you would configure Prometheus to listen only on localhost. 
- web.enable-lifecycle – Allows to manage Prometheus, for example, to reload configuration without restarting the service.

13. To automatically start the Prometheus after reboot, run enable:
```
sudo systemctl enable prometheus
```

14. Then just start the Prometheus:
```
sudo systemctl start prometheus
```

15. To check the status of Prometheus run the following command:
```
sudo systemctl status prometheus
```

Suppose you encounter any issues with Prometheus or are unable to start it. The easiest way to find the problem is to use the journalctl command and search for errors.
```
journalctl -u prometheus -f --no-pager
```
16. Now we can try to access it via the browser. I’m going to be using the IP address of the Ubuntu server. You need to append port 9090 to the IP. 

Access prometheus at http://Prometheus-Grafana_public_ip:9090/targets

If you go to targets, you should see only one - Prometheus target. It scrapes itself every 15 seconds by default.

### 4D — Install & Configure Node Exporter

Install Node Exporter on Ubuntu 24.04

Next, we’re going to set up and configure Node Exporter to collect Linux system metrics like CPU load and disk I/O. Node Exporter will expose these as Prometheus-style metrics. Since the installation process is very similar, I’m not going to cover as deep as Prometheus.

1. First, let’s create a system user for Node Exporter by running the following command:
```
sudo useradd --system --no-create-home --shell /bin/false node_exporter
```

2. You can download Node Exporter from the same page. Use the wget command to download the binary:
```
wget https://github.com/prometheus/node_exporter/releases/download/v1.6.1/node_exporter-1.6.1.linux-amd64.tar.gz
```

3. Extract the node exporter from the archive:
```
tar -xvf node_exporter-1.6.1.linux-amd64.tar.gz
```

4. Move binary to the /usr/local/bin:
```
sudo mv node_exporter-1.6.1.linux-amd64/node_exporter /usr/local/bin/
```

5. Clean up, and delete node_exporter archive and a folder:
```
rm -rf node_exporter*
```

6. Verify that you can run the binary:
```
node_exporter --version
```

7. Node Exporter has a lot of plugins that we can enable. If you run Node Exporter help you will get all the options.
```
node_exporter --help
```
- collector.logind - We’re going to enable the login controller, just for the demo.

8. Next, create a similar systemd unit file:
```
sudo vim /etc/systemd/system/node_exporter.service
```
Copy below content in node_exporter.service
```
[Unit]
Description=Node Exporter
Wants=network-online.target
After=network-online.target
StartLimitIntervalSec=500
StartLimitBurst=5
[Service]
User=node_exporter
Group=node_exporter
Type=simple
Restart=on-failure
RestartSec=5s
ExecStart=/usr/local/bin/node_exporter \
    --collector.logind
[Install]
WantedBy=multi-user.target
```
Replace Prometheus user and group to node_exporter, and update the ExecStart command.

9. To automatically start the Node Exporter after reboot, enable the service:
```
sudo systemctl enable node_exporter
```
10. Then start the Node Exporter:
```
sudo systemctl start node_exporter
```

11. Check the status of Node Exporter with the following command:
```
sudo systemctl status node_exporter
```

If you have any issues, check logs with journalctl
```
journalctl -u node_exporter -f --no-pager
```

Note:
- At this point, we have only a single target in our Prometheus. 
- There are many different service discovery mechanisms built into Prometheus. For example, Prometheus can dynamically discover targets in AWS, GCP, and other clouds based on the labels. 

12. To create a static target, you need to add job_name with static_configs:
```
sudo vim /etc/prometheus/prometheus.yml
```
Copy and paste below content in prometheus.yml at bottom 
```

- job_name: node_export
    static_configs:
      - targets: ["localhost:9100"]
```

By default, Node Exporter will be exposed on port 9100. The complete prometheus.yml should look like below:
```
# my global config
global:
  scrape_interval: 15s # Set the scrape interval to every 15 seconds. Default is every 1 minute.
  evaluation_interval: 15s # Evaluate rules every 15 seconds. The default is every 1 minute.
  # scrape_timeout is set to the global default (10s).

# Alertmanager configuration
alerting:
  alertmanagers:
    - static_configs:
        - targets:
          # - alertmanager:9093

# Load rules once and periodically evaluate them according to the global 'evaluation_interval'.
rule_files:
  # - "first_rules.yml"
  # - "second_rules.yml"

# A scrape configuration containing exactly one endpoint to scrape:
# Here it's Prometheus itself.
scrape_configs:
  # The job name is added as a label `job=<job_name>` to any timeseries scraped from this config.
  - job_name: "prometheus"

    # metrics_path defaults to '/metrics'
    # scheme defaults to 'http'.

    static_configs:
      - targets: ["localhost:9090"]

  - job_name: node_export
    static_configs:
      - targets: ["localhost:9100"]
```

Since we enabled lifecycle management via API calls, we can reload the Prometheus config without restarting the service and causing downtime.

13. Before, restarting check if the config is valid:
```
promtool check config /etc/prometheus/prometheus.yml
```

14. Then, you can use a POST request to reload the config:
```
curl -X POST http://localhost:9090/-/reload
```

15. Check the targets section

http://Prometheus-Grafana_public_ip:9090/targets

If you go to targets, you should see - Prometheus and node_exporter targets now.

### 4E — Install Grafana

Install Grafana on Ubuntu 24.04

To visualize metrics we can use Grafana. There are many different data sources that Grafana supports, one of them is Prometheus.

1. First, let’s make sure that all the dependencies are installed:
```
sudo apt-get install -y apt-transport-https software-properties-common
```

2. Next, add the GPG key:
```
wget -q -O - https://packages.grafana.com/gpg.key | sudo apt-key add -
```

3. Add this repository for stable releases:
```
echo "deb https://packages.grafana.com/oss/deb stable main" | sudo tee -a /etc/apt/sources.list.d/grafana.list
```

4. After you add the repository, update and install Garafana:
```
sudo apt-get update
sudo apt-get -y install grafana
```

5. To automatically start the Grafana after reboot, enable the service:
```
sudo systemctl enable grafana-server
```

6. Then start the Grafana.
```
sudo systemctl start grafana-server
```

7. To check the status of Grafana, run the following command:
```
sudo systemctl status grafana-server
```

8. Go to http://Prometheus-Grafana_public_ip:3000 and log in to the Grafana using default credentials. The username is admin, and the password is admin as well.
```
username admin
password admin
```
Note: When you log in for the first time, you get the option to change the password.

9. To visualize metrics, you need to add a data source first:

- Click Add data source and select Prometheus.
- Name it as prometheus
- For Prometheus Server URL, enter Prometheus-Grafana_public_ip:9090 
- click Save and test. 
- You can see Data source is working.

10. Let’s add Dashboard for a better view:

- Click on Import Dashboard 
- paste this code 1860 and click on load
- Select the Datasource (prometheus) and click on Import
- You will see output


## Step 5 : Install the Prometheus Plugin and Integrate it with the Prometheus server

### 5A — Jenkins configure
Let’s configure JENKINS SYSTEM to monitor via prometheus and grafana.

Note: Make sure Jenkins server up and running (Jenkins-sonar)

1. Install prometheus plugin:
- Goto Manage Jenkins –> Plugins –> Available Plugins
- Search for Prometheus metrics and install it

2. Once that is done set/verify Prometheus path in system configurations
- Manage Jenkins --> system
- search for prometheus
- Nothing to change click on apply and save

3. To create a static target, you need to add job_name with static_configs. go to Prometheus server:
```
sudo vim /etc/prometheus/prometheus.yml
```

Copy and paste below code at bottom of prometheus.yml file:
```
- job_name: 'jenkins'
    metrics_path: '/prometheus'
    static_configs:
      - targets:['Jenkin-sonar_public_IP:8080']
```

Optional to check all config in prometheus.yml that we updates should look like below now:
```
global:
  scrape_interval: 15s # Set the scrape interval to every 15 seconds. Default is every 1 minute.
  evaluation_interval: 15s # Evaluate rules every 15 seconds. The default is every 1 minute.
  # scrape_timeout is set to the global default (10s).

# Alertmanager configuration
alerting:
  alertmanagers:
    - static_configs:
        - targets:
          # - alertmanager:9093

# Load rules once and periodically evaluate them according to the global 'evaluation_interval'.
rule_files:
  # - "first_rules.yml"
  # - "second_rules.yml"

# A scrape configuration containing exactly one endpoint to scrape:
# Here it's Prometheus itself.
scrape_configs:
  # The job name is added as a label `job=<job_name>` to any timeseries scraped from this config.
  - job_name: "prometheus"

    # metrics_path defaults to '/metrics'
    # scheme defaults to 'http'.

    static_configs:
      - targets: ["localhost:9090"]

  - job_name: node_export
    static_configs:
      - targets: ["localhost:9100"]

  - job_name: "jenkins"
    metrics_path: "/prometheus"
    static_configs:
      - targets: ["3.15.240.199:8080"]
```

4. Before, restarting check if the config is valid:
```
promtool check config /etc/prometheus/prometheus.yml
```

5. Then, you can use a POST request to reload the config:
```
curl -X POST http://localhost:9090/-/reload
```

6. Check the targets section:
```
http://Prometheus-Grafana_public_ip:9090/targets
```
You will see Jenkins is added to it in prometheus target now.

Note: If your jenkins target is DOWN, please add SG inbound rule on "jenkins-Sonar" and add port 8080 for prometheus-grafana_public_IP. Then again follow step 5 and 6.


### 5B — Configure Grafana dashboard

Let’s add Dashboard for a better view in Grafana

- Click On Dashboard –> + symbol –> Import Dashboard

- Use Id 9964 and click on load

- Select the data source and click on Import

- Now you will see the Detailed overview of Jenkins


## Step 6 : Email Integration With Jenkins and Plugin Setup


- Go to your Gmail and click on your profile

- Then click on Manage Your Google Account –> click on the security tab on the left side panel you will get page(provide mail password).

- 2-step verification should be enabled.

- Search for the app in the search bar you will get app passwords

- Click on other and provide your name (Jenkins) and click on Generate and copy the password

- In the new update, you will get a password


Install Email Extension Template Plugin in Jenkins

Once the plugin is installed in Jenkins, click on manage Jenkins –> configure system there under the E-mail Notification section configure the details as below:

- SMTP server : smtp@gmail.com
- Use SMTP Authentication --> User name : nikhil.ambade600@gmail.com
- Use SMTP Authentication --> password  : generated password
- tick "Use SSL"
- SMTP Port : 465

- Click on Apply and save.

Add credentials:
- Click on Manage Jenkins
- credentials and add your mail username and generated password
- ID : mail
- Description : mail


This is to just verify the mail configuration

Now under the Extended E-mail Notification section configure the details as shown in the below
- SMTP server : smtp@gmail.com
- SMTP Port : 465
- Advanced --> credentials --> select "mail"
- Click on Apply and save.

```
post {
     always {
        emailext attachLog: true,
            subject: "'${currentBuild.result}'",
            body: "Project: ${env.JOB_NAME}
" +
                "Build Number: ${env.BUILD_NUMBER}
" +
                "URL: ${env.BUILD_URL}
",
            to: 'nikhil.ambade600@gmail.com',  #change Your mail
            attachmentsPattern: 'trivyfs.txt,trivyimage.txt'
        }
    }
```
Next, we will log in to Jenkins and start to configure our Pipeline in Jenkins

## Step 7 : Install Plugins like JDK, Sonarqube Scanner, NodeJs, OWASP Dependency Check

### 7A — Install Plugin
Goto Manage Jenkins →Plugins → Available Plugins →

Install below plugins

1 → Eclipse Temurin Installer (Install without restart)

2 → SonarQube Scanner (Install without restart)

3 → NodeJs Plugin (Install Without restart)


### 7B — Configure Java and Nodejs in Global Tool Configuration
Goto Manage Jenkins → Tools → Install JDK(17) 
- JDK Installtions
- Add JDK
- Name : jdk17
- Install automatically

Goto Manage Jenkins → Tools → NodeJs(16)
- NodeJS
- Name : node16
- Install automatically

Click on Apply and Save


### 7C — Create a Job
Create a job as Netflix Name, select pipeline and click on ok.

## Step 8 : Configure Sonar Server in Manage Jenkins
Grab the Public IP Address of your EC2 Instance, Sonarqube works on Port 9000, so :9000. 

Goto your Sonarqube Server. Click on Administration → Security → Users → Click on Tokens and Update Token → Give it a name → and click on Generate Token

copy Token

Goto Jenkins Dashboard → Manage Jenkins → Credentials 
- Kind : Secret Text
- Scope : Global
- Secret : paste sonar token here
- id: Sonar-token
- description : Sonar-token

Now, go to Dashboard → Manage Jenkins → System and Add like the below
- search for SonarQube servers --> sonarquve installtions
- Name : sonar-server
- URL : Jenkins-sonar_public_IP:9000
- Server Authentication token : Sonar-token
- Click on Apply and Save

The Configure System option is used in Jenkins to configure different server

Global Tool Configuration is used to configure different tools that we install using Plugins

We will install a sonar scanner in the tools:
- search for sonarqube scanner installtions
- Name : sonar-scanner
- Install automatically

In the Sonarqube Dashboard add a quality gate also:

- Administration–> Configuration–>Webhooks->
- Click on Create
- Name : jenkins
- URL : http://Jenkins-sonar_public_IP:8080/sonarqube-webwook/
- create


Let’s go to our Pipeline and add the script in our Pipeline Script.
```
pipeline{
    agent any
    tools{
        jdk 'jdk17'
        nodejs 'node16'
    }
    environment {
        SCANNER_HOME=tool 'sonar-scanner'
    }
    stages {
        stage('clean workspace'){
            steps{
                cleanWs()
            }
        }
        stage('Checkout from Git'){
            steps{
                git branch: 'main', url: 'https://github.com/anikhilt600/Netflix-app.git'
            }
        }
        stage("Sonarqube Analysis "){
            steps{
                withSonarQubeEnv('sonar-server') {
                    sh ''' $SCANNER_HOME/bin/sonar-scanner -Dsonar.projectName=Netflix \
                    -Dsonar.projectKey=Netflix '''
                }
            }
        }
        stage("quality gate"){
           steps {
                script {
                    waitForQualityGate abortPipeline: false, credentialsId: 'Sonar-token'
                }
            }
        }
        stage('Install Dependencies') {
            steps {
                sh "npm install"
            }
        }
    }
    post {
     always {
        emailext attachLog: true,
            subject: "'${currentBuild.result}'",
            body: "Project: ${env.JOB_NAME}
" +
                "Build Number: ${env.BUILD_NUMBER}
" +
                "URL: ${env.BUILD_URL}
",
            to: 'nikhil.ambade600@gmail.com',
            attachmentsPattern: 'trivyfs.txt,trivyimage.txt'
        }
    }
}
```
Click on Build now, you will see the stage view.

To see the report, you can go to Sonarqube Server and go to Projects.

You can see the report has been generated and the status shows as passed. You can see that there are 3.2k lines it scanned. To see a detailed report, you can go to issues.

## Step 9 : Install OWASP Dependency Check Plugins

Goto Dashboard → Manage Jenkins → Plugins → OWASP Dependency-Check. Click on it and install it without restart.

First, we configured the Plugin and next, we had to configure the Tool

Goto Dashboard → Manage Jenkins → Tools
- search for Dependency-Check installtions
- Name : DP-Check
- Install Automatically
- Click on Apply and Save here.

Now go to configure → Pipeline and add this stage to your pipeline and build.

```
stage('OWASP FS SCAN') {
            steps {
                dependencyCheck additionalArguments: '--scan ./ --disableYarnAudit --disableNodeAudit', odcInstallation: 'DP-Check'
                dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
            }
        }
        stage('TRIVY FS SCAN') {
            steps {
                sh "trivy fs . > trivyfs.txt"
            }
        }
```

You will see that in status, a graph will also be generated and Vulnerabilities.


## Step 10 : Docker Image Build and Push

We need to install the Docker tool in our system, 

Goto Dashboard → Manage Plugins → Available plugins → Search for Docker and install these plugins:
```
Docker

Docker Commons

Docker Pipeline

Docker API

docker-build-step
```
and click on install without restart

Now, goto Dashboard → Manage Jenkins → Tools
- Search for Docker installations
- Name : docker
- Install Automatically

Add DockerHub Username and Password under Global Credentials
- Kind : username with password
- scope : Global
- Username : anikhilt600
- password : dockerhub token
- ID : docker


Add this stage to Pipeline Script
```
stage("Docker Build & Push"){
            steps{
                script{
                   withDockerRegistry(credentialsId: 'docker', toolName: 'docker'){
                       sh "docker build --build-arg TMDB_V3_API_KEY=Aj7ay86fe14eca3e76869b92 -t netflix ."
                       sh "docker tag netflix anikhilt600/netflix:latest "
                       sh "docker push anikhilt600/netflix:latest "
                    }
                }
            }
        }
        stage("TRIVY"){
            steps{
                sh "trivy image sevenajay/netflix:latest > trivyimage.txt"
            }
        }
```
You will see the output below, with a dependency trend.

When you log in to Dockerhub, you will see a new image is created

Now Run the container to see if the app coming up or not by adding the below stage
```
stage('Deploy to container'){
            steps{
                sh 'docker run -d --name netflix -p 8081:80 anikhilt600/netflix:latest'
            }
        }
```
stage view

You will get the output of app here in browser : <Jenkins-sonar_public-IP:8081> (Make sure to open 8081 port in SG of Jenkins-sonar server)



## Step 11 : Kuberenetes Setup

USE THIS SETUP PLEASE ( since 20.04 is not available anymore )

https://mrcloudbook.com/ultimate-guide-setting-up-a-kubernetes-cluster-with-master-and-worker-nodes-on-ubuntu-24-04-for-optimal-performance/

### 11A — Launch an Ubuntu(24.04) T2 medium Instances
Lets create 2 new EC2 instances for kubernetes master and worker.

- Open the AWS Management Console: Navigate to the EC2 Dashboard.
- Launch Instance: Click on the “Launch Instance” button.
- Choose an Amazon Machine Image (AMI): Select “Ubuntu Server 24.04 LTS (HVM), SSD Volume Type”.
- Choose an Instance Type: Select “t2.medium”.
- Configure Instance Details:
 1. Number of instances: 2
 2. Network: Default VPC
 3. Subnet: Choose an available subnet
 4. Auto-assign Public IP: Enable
 5. Add Storage: Use the default 16GB for the root volume.
 6.  Add Tags: (Optional) Add tags to identify your instances.
 7. Configure Security Group: Create a new security group or use an existing one. ( Open all Ports for Learning Purpose Only ). Add rules to allow SSH access (port 22) from your IP address.
 8. Select an existing key pair or create a new one for SSH access, 
 9. then click “Launch Instances”.

After launching, you will have two new instances ready to be configured as your Kubernetes cluster.

### 11B — Install and Configure Docker and Kubernetes

- Connect to the Instances
- Run the following commands on both instances (master and worker nodes):

1. To Update and Upgrade Packages: (master and worker nodes)
```
sudo apt update
sudo apt upgrade -y
```
Note: If you want to set hostnames of EC2 to see which one is master and worker, then do it as follow:
```
#on  master node
sudo hostnamectl set-hostname master
exec bash

#on worker node
sudo hostnamectl set-hostname worker
exec bash
```

2. To Disable Swap: (master and worker nodes)
```
sudo swapoff -a
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
```
3. To Load Kernel Modules: (master and worker nodes)
```
sudo tee /etc/modules-load.d/containerd.conf /dev/null 2>&1
sudo sed -i 's/SystemdCgroup \= false/SystemdCgroup \= true/g' /etc/containerd/config.toml
sudo systemctl restart containerd
sudo systemctl enable containerd
```
4. To Install Kubernetes Packages: (master and worker nodes)
```
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.28/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo 'deb &#91;signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.28/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo apt update
sudo apt install -y kubeadm=1.28.1-1.1 kubelet=1.28.1-1.1 kubectl=1.28.1-1.1
```

If the above command doesn’t work, use the Snap method: (master and worker nodes)
```
sudo snap install kubeadm=1.28.1-1.1 --classic
sudo snap install kubectl=1.28.1-1.1 --classic
sudo snap install kubelet=1.28.1-1.1 --classic
```

5. Initialize the Cluster: (master node)
On the master node, run the following command to initialize the Kubernetes cluster:
```
sudo kubeadm init
```
Note: You will get the "kubeadm join" command provided during the master node initialization on each worker node. We will be using this to join worker node to cluster in below 8th step.

6. To set up the kubeconfig for the user on the master node: (master node)
Note: In case you are in root, exit from it and run below commands
```
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```
7. Deploy a Pod Network : (master node)
To enable communication between pods, you need to deploy a pod network. Here’s how to deploy Calico, a popular networking solution for Kubernetes:
```
kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml
```
8. to Add/Join Worker Nodes to the Cluster: (worker node)
Use the kubeadm join command provided during the master node initialization on each worker node. This command will look something like this (replace the placeholders xx.xx.xx.xx with your actual values):
```
sudo kubeadm join xx.xx.xx.xx:6443 --token  --discovery-token-ca-cert-hash sha256:
# Dont forget to use SUDO
```
9. To Verify the Cluster (master node)
To verify that the worker nodes have joined the cluster, run the following command on the master node:
```
kubectl get nodes
```

### Optional (not required)
```
sudo kubeadm init --pod-network-cidr=10.244.0.0/16
# in case your in root exit from it and run below commands
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```

### 11C — Config kubernetes secret file for Jenkins

Copy the config file to Jenkins master or the local file manager and save it:
```
cd ./kube

ls

cat config
```
- Copy the whole config file content
- on you local machine --> file explorer
- go to any folder
- create txt file "Secret File"
- Paste the config file copied content to "Secret File.txt"
- save

Note: create a secret-file.txt in your file explorer save the config in it and use this at the kubernetes credential section.

### 11D — Setup Jenkins credential

Install the below Kubernetes Plugin:
```
kubernetes
kubernetes credentials
kubernetes client API
kunernetes CLI
kubernetes credentials Provider
```
Once it’s installed successfully

Goto manage Jenkins –> manage credentials –> Click on Jenkins global –> add credentials
- kind : secret file
- scope : global
- Choose File : "Secret File.txt"
- ID : k8s
- Description : k8s
- Create


### 11E — Node_exporter on Master and Worker to monitor the metrics

Let’s add Node_exporter on Master and Worker to monitor the metrics

1. First, let’s create a system user for Node Exporter by running the following command: (master and worker node)
```
sudo useradd --system --no-create-home --shell /bin/false node_exporter
```

2. You can download Node Exporter from the same page. Use the wget command to download the binary.
```
wget https://github.com/prometheus/node_exporter/releases/download/v1.6.1/node_exporter-1.6.1.linux-amd64.tar.gz
```

3. Extract the node exporter from the archive.
```
tar -xvf node_exporter-1.6.1.linux-amd64.tar.gz
```
4. Move binary to the /usr/local/bin.
```
sudo mv node_exporter-1.6.1.linux-amd64/node_exporter /usr/local/bin/
```

5. (optional) Clean up, and delete node_exporter archive and a folder.
```
rm -rf node_exporter*
```

6. Verify that you can run the binary.
```
node_exporter --version
```

Node Exporter has a lot of plugins that we can enable. If you run Node Exporter help you will get all the options.
```
node_exporter --help
```
–collector.logind We’re going to enable the login controller, just for the demo.

7. Next, create a similar systemd unit file.
```
sudo vim /etc/systemd/system/node_exporter.service
```

Replace Prometheus user and group to node_exporter, and update the ExecStart command in "node_exporter.service"
```
[Unit]
Description=Node Exporter
Wants=network-online.target
After=network-online.target
StartLimitIntervalSec=500
StartLimitBurst=5
[Service]
User=node_exporter
Group=node_exporter
Type=simple
Restart=on-failure
RestartSec=5s
ExecStart=/usr/local/bin/node_exporter \
    --collector.logind
[Install]
WantedBy=multi-user.target
```

8. To automatically start the Node Exporter after reboot, enable the service.
```
sudo systemctl enable node_exporter
```
Then start the Node Exporter.
```
sudo systemctl start node_exporter
```

9. Check the status of Node Exporter with the following command:
```
sudo systemctl status node_exporter
```

If you have any issues, check logs with journalctl
```
journalctl -u node_exporter -f --no-pager
```

10. To create a static target, you need to add job_name with static_configs. Go to Prometheus server (prometheus-grafana_public_IP:9090)
```
sudo vim /etc/prometheus/prometheus.yml
```

Copy and paste below in "prometheus.yml" (make sure to replace IP's)
```
- job_name: node_export_masterk8s
    static_configs:
      - targets:["master_public_IP:9100"]

  - job_name: node_export_workerk8s
    static_configs:
      - targets:["worker_public_IP:9100"]
```
By default, Node Exporter will be exposed on port 9100.

Since we enabled lifecycle management via API calls, we can reload the Prometheus config without restarting the service and causing downtime.

Before, restarting check if the config is valid.
```
promtool check config /etc/prometheus/prometheus.yml
```
Then, you can use a POST request to reload the config.
```
curl -X POST http://localhost:9090/-/reload
```

Check the targets section on prometheus-grafana server http://prometheus-grafana_public_IP:9090/targets


### 11F — final step to deploy on the Kubernetes cluster
```
stage('Deploy to kubernets'){
            steps{
                script{
                    dir('Kubernetes') {
                        withKubeConfig(caCertificate: '', clusterName: '', contextName: '', credentialsId: 'k8s', namespace: '', restrictKubeConfigAccess: false, serverUrl: '') {
                                sh 'kubectl apply -f deployment.yml'
                                sh 'kubectl apply -f service.yml'
                        }
                    }
                }
            }
        }
```
stage view in jenkins

In the Kubernetes cluster(master) give this command:
```
kubectl get all 
kubectl get svc
```

## STEP 12 : Access from a Web browser with

Chek for the browser output:  worker_public_IP:service port

Check for the Monitoring in prometheus-grafana server:  http://prometheus-grafana_public_IP:9090/targets

Check if you are receiving Mails


## 13 : Complete Pipeline
```
pipeline{
    agent any
    tools{
        jdk 'jdk17'
        nodejs 'node16'
    }
    environment {
        SCANNER_HOME=tool 'sonar-scanner'
    }
    stages {
        stage('clean workspace'){
            steps{
                cleanWs()
            }
        }
        stage('Checkout from Git'){
            steps{
                git branch: 'main', url: 'https://github.com/anikhilt600/Netflix-app.git'
            }
        }
        stage("Sonarqube Analysis "){
            steps{
                withSonarQubeEnv('sonar-server') {
                    sh ''' $SCANNER_HOME/bin/sonar-scanner -Dsonar.projectName=Netflix \
                    -Dsonar.projectKey=Netflix '''
                }
            }
        }
        stage("quality gate"){
           steps {
                script {
                    waitForQualityGate abortPipeline: false, credentialsId: 'Sonar-token' 
                }
            } 
        }
        stage('Install Dependencies') {
            steps {
                sh "npm install"
            }
        }
        stage('OWASP FS SCAN') {
            steps {
                dependencyCheck additionalArguments: '--scan ./ --disableYarnAudit --disableNodeAudit', odcInstallation: 'DP-Check'
                dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
            }
        }
        stage('TRIVY FS SCAN') {
            steps {
                sh "trivy fs . > trivyfs.txt"
            }
        }
        stage("Docker Build & Push"){
            steps{
                script{
                   withDockerRegistry(credentialsId: 'docker', toolName: 'docker'){   
                       sh "docker build --build-arg TMDB_V3_API_KEY=AJ7AYe14eca3e76864yah319b92 -t netflix ."
                       sh "docker tag netflix anikhilt600/netflix:latest "
                       sh "docker push anikhilt600/netflix:latest "
                    }
                }
            }
        }
        stage("TRIVY"){
            steps{
                sh "trivy image anikhilt600/netflix:latest > trivyimage.txt" 
            }
        }
        stage('Deploy to container'){
            steps{
                sh 'docker run -d --name netflix -p 8081:80 anikhilt600/netflix:latest'
            }
        }
        stage('Deploy to kubernets'){
            steps{
                script{
                    dir('Kubernetes') {
                        withKubeConfig(caCertificate: '', clusterName: '', contextName: '', credentialsId: 'k8s', namespace: '', restrictKubeConfigAccess: false, serverUrl: '') {
                                sh 'kubectl apply -f deployment.yml'
                                sh 'kubectl apply -f service.yml'
                        }   
                    }
                }
            }
        }

    }
    post {
     always {
        emailext attachLog: true,
            subject: "'${currentBuild.result}'",
            body: "Project: ${env.JOB_NAME}
" +
                "Build Number: ${env.BUILD_NUMBER}
" +
                "URL: ${env.BUILD_URL}
",
            to: 'nikhil.ambade600@gmail.com',
            attachmentsPattern: 'trivyfs.txt,trivyimage.txt'
        }
    }
}
```
