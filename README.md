# Project which is based on deploying a app to kubernetes cluster using Jenkins CI/CD pipeline and monitoring them 
Installing jenkins

    sudo apt-get update -y && sudo apt-get upgrade -y
    wget -O - https://packages.adoptium.net/artifactory/api/gpg/key/public | tee /etc/apt/keyrings/adoptium.asc
    echo "deb [signed-by=/etc/apt/keyrings/adoptium.asc] https://packages.adoptium.net/artifactory/deb $(awk -F= '/^VERSION_CODENAME/{print$2}' /etc/os-release) main" | tee /etc/apt/sources.list.d/adoptium.list
    sudo apt update -y
    sudo apt install openjdk-11-jdk -y
    /usr/bin/java --version
    curl -fsSL https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key | sudo tee /usr/share/keyrings/jenkins-keyring.asc > /dev/null
    echo deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] https://pkg.jenkins.io/debian-stable binary/ | sudo tee /etc/apt/sources.list.d/jenkins.list > /dev/null
    sudo apt-get update -y
    sudo apt install jenkins -y
    sudo systemctl start jenkins && sudo systemctl enable jenkins
Install Docker and Run SonarQube as Container

    sudo apt-get update
    sudo apt-get install docker.io -y
    sudo usermod -aG docker ubuntu
    sudo usermod -aG docker jenkins  
    newgrp docker
    sudo chmod 777 /var/run/docker.sock
    docker run -d --name sonar -p 9000:9000 sonarqube:lts-community
Install trivy
    
    sudo apt-get install wget apt-transport-https gnupg lsb-release -y
    wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | gpg --dearmor | sudo tee /usr/share/keyrings/trivy.gpg > /dev/null
    echo "deb [signed-by=/usr/share/keyrings/trivy.gpg] https://aquasecurity.github.io/trivy-repo/deb $(lsb_release -sc) main" | sudo tee -a /etc/apt/sources.list.d/trivy.list
    sudo apt-get update
    sudo apt-get install trivy -y

# Installing prometheus, Grafana & node exporter
Prometheus

    sudo useradd --system --no-create-home --shell /bin/false prometheus
    wget https://github.com/prometheus/prometheus/releases/download/v2.47.1/prometheus-2.47.1.linux-amd64.tar.gz
    tar -xvf prometheus-2.47.1.linux-amd64.tar.gz
    cd prometheus-2.47.1.linux-amd64/
    sudo mkdir -p /data /etc/prometheus
    sudo mv prometheus promtool /usr/local/bin/
    sudo mv consoles/ console_libraries/ /etc/prometheus/
    sudo mv prometheus.yml /etc/prometheus/prometheus.yml
    sudo chown -R prometheus:prometheus /etc/prometheus/ /data/
    sudo cat > /etc/systemd/system/prometheus.service << EOF
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
    EOF
    sudo systemctl enable prometheus
    sudo systemctl start prometheus

Node Exporter 

    sudo useradd --system --no-create-home --shell /bin/false node_exporter
    wget https://github.com/prometheus/node_exporter/releases/download/v1.6.1/node_exporter-1.6.1.linux-amd64.tar.gz
    tar -xvf node_exporter-1.6.1.linux-amd64.tar.gz
    sudo mv node_exporter-1.6.1.linux-amd64/node_exporter /usr/local/bin/
    rm -rf node_exporter*
    sudo cat > /etc/systemd/system/node_exporter.service << EOF
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
    ExecStart=/usr/local/bin/node_exporter --collector.logind

    [Install]
    WantedBy=multi-user.target
    EOF
    sudo systemctl enable node_exporter
    sudo systemctl start node_exporter

To add node exporter and Jenkins in prometheus 

create a static target, you need to add job_name with static_configs.

    sudo vim /etc/prometheus/prometheus.yml
prometheus.yml

      - job_name: node_export
        static_configs:
          - targets: ['Ip address:9100']
      - job_name: 'jenkins'
          metrics_path: '/prometheus'
          static_configs:
            - targets: ['IP-Address:8080']
Check if the config is valid

    promtool check config /etc/prometheus/prometheus.yml
POST request to reload the config

    curl -X POST http://localhost:9090/-/reload

Grafana

    sudo apt-get update
    sudo apt-get install -y apt-transport-https software-properties-common
    wget -q -O - https://packages.grafana.com/gpg.key | sudo apt-key add -
    echo "deb https://packages.grafana.com/oss/deb stable main" | sudo tee -a /etc/apt/sources.list.d/grafana.list
    sudo apt-get update
    sudo apt-get -y install grafana
    sudo systemctl enable grafana-server
    sudo systemctl start grafana-server
To check the status of the all three

    sudo systemctl status prometheus
    sudo systemctl status node_exporter
    sudo systemctl status grafana-server


