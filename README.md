# Monitoring_CI_tool
To Monitor Jenkins, Gitlab application using Prometheus, Node exporter and Grafana

![alt text](https://raw.githubusercontent.com/Jegankumard/Monitoring_CI_tool/main/Jenkins_Prometheus_Grafana.avif)

## Install Prometheus on Ubuntu
create a system user or system account
```
sudo useradd \
    --system \
    --no-create-home \
    --shell /bin/false prometheus
```
curl or wget command to download Prometheus
```
wget https://github.com/prometheus/prometheus/releases/download/v2.47.1/prometheus-2.47.1.linux-amd64.tar.gz
tar -xvf prometheus-2.47.1.linux-amd64.tar.gz
sudo mkdir -p /data /etc/prometheus 
cd prometheus-2.47.1.linux-amd64/
sudo mv prometheus promtool /usr/local/bin/
sudo mv consoles/ console_libraries/ /etc/prometheus/
sudo mv prometheus.yml /etc/prometheus/prometheus.yml
sudo chown -R prometheus:prometheus /etc/prometheus/ /data/
cd
rm -rf prometheus-2.47.1.linux-amd64.tar.gz
prometheus --version
prometheus --help
```
We use Systemd So we create a Systemd unit configuration file.
```
sudo vim /etc/systemd/system/prometheus.service
>>>
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
<<<
```
Start Pometheus
```
sudo systemctl enable prometheus
sudo systemctl start prometheus
sudo systemctl status Prometheus
```
Debug error
```
journalctl -u prometheus -f --no-pager
<public-ip-address:9090>
```
## Install Node Exporter on Ubuntu
create a system user or system account
```
sudo useradd \
    --system \
    --no-create-home \
    --shell /bin/false node_exporter
```
curl or wget command to download Node exporter
```
wget https://github.com/prometheus/node_exporter/releases/download/v1.6.1/node_exporter-1.6.1.linux-amd64.tar.gz
tar -xvf node_exporter-1.6.1.linux-amd64.tar.gz

sudo mv node_exporter-1.6.1.linux-amd64/node_exporter /usr/local/bin/
rm -rf node_exporter*
node_exporter --version
node_exporter --help
```
We use Systemd So we create a Systemd unit configuration file.
```
sudo vim /etc/systemd/system/node_exporter.service
>>>
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
<<<
```
Start node_exporter
```
sudo systemctl enable node_exporter
sudo systemctl start node_exporter
sudo systemctl status node_exporter
```
Debug error
```
journalctl -u node_exporter -f --no-pager
<public-ip-address:9090>
```

create a static target with job_name and static_configs
```
sudo vim /etc/prometheus/prometheus.yml
>>>
  - job_name: node_export
    static_configs:
      - targets: ["localhost:9100"]
<<<
```

By default, Node Exporter will be exposed on port 9100.
Since we enabled lifecycle management via API calls, 
we can reload the Prometheus config without restarting the service
```
promtool check config /etc/prometheus/prometheus.yml
curl -X POST http://localhost:9090/-/reload
http://<ip>:9090/targets

```

## Install Grafana on Ubuntu
Dependency install, GPG key, Stable release
```
sudo apt-get install -y apt-transport-https software-properties-common
wget -q -O - https://packages.grafana.com/gpg.key | sudo apt-key add -
echo "deb https://packages.grafana.com/oss/deb stable main" | sudo tee -a /etc/apt/sources.list.d/grafana.list

```
Install Grafana
```
sudo apt-get update
sudo apt-get -y install grafana
```

Start Pometheus
```
sudo systemctl enable grafana-server
sudo systemctl start grafana-server
sudo systemctl status grafana-server
```
Login - default (username is admin, and the password is admin)
```
http://<ip>:3000
```
To visualize metrics, add a data source > prometheus > URL > save, test
URL- http://localhost:9090 or <public-ip:9090>

Add Dashboard > Click on Import Dashboard paste this code 1860 and click on load > Select the Datasource and click on Import > output

## Launch Jenkins in Ubuntu
- Step 1. Update the System
- Step 2. Install Apache Web Server
- Step 3. Install Java
- Step 4. Install Jenkins
- Step 5. Setting up Apache as a Reverse Proxy
- Step 7. Finish Installation

```
sudo apt-get update -y && sudo apt-get upgrade -y
sudo apt install apache2 -y
sudo systemctl enable apache2 && sudo systemctl start apache2
sudo systemctl status apache2

sudo apt install openjdk-11-jdk -y
java --version
```
jenkins
```
curl -fsSL https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key | sudo tee \
  /usr/share/keyrings/jenkins-keyring.asc > /dev/null

echo deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] \
  https://pkg.jenkins.io/debian-stable binary/ | sudo tee \
  /etc/apt/sources.list.d/jenkins.list > /dev/null

sudo apt update -y
sudo apt install jenkins -y
sudo systemctl start jenkins && sudo systemctl enable jenkins
sudo systemctl status jenkins
```

To access Jenkins installation via domain, we need to configure the Apache as a Reverse Proxy
```
cd /etc/apache2/sites-available/
sudo nano jenkins.conf
>>>
<Virtualhost *:80>
    ServerName        yourdomain.com
    ProxyRequests     Off
    ProxyPreserveHost On
    AllowEncodedSlashes NoDecode

    <Proxy http://localhost:8080/*>
      Order deny,allow
      Allow from all
    </Proxy>

    ProxyPass         /  http://localhost:8080/ nocanon
    ProxyPassReverse  /  http://localhost:8080/
    ProxyPassReverse  /  http://yourdomain.com/
</Virtualhost>
<<<
```
execute
```
sudo a2ensite jenkins
sudo a2enmod proxy
sudo a2enmod proxy_http
sudo a2enmod headers
sudo systemctl restart apache2
```
Admin password
```
cat /var/lib/jenkins/secrets/initialAdminPassword
```
## Monitor Jenkins

#### Plugins:
Manage Jenkins --> Plugins --> Available Plugins > Prometheus metrics > install

Manage Jenkins --> System --> set path in system configurations to /Prometheus > save

create a static target (jenkins) with job_name and static_configs
```
sudo vim /etc/prometheus/prometheus.yml
>>>
  - job_name: 'jenkins'
    metrics_path: '/prometheus'
    static_configs:
      - targets: ['<jenkins-ip>:8080']

<<<
```
check valid config and reload, Check targets(Jenkins added)
```
promtool check config /etc/prometheus/prometheus.yml
curl -X POST http://localhost:9090/-/reload
http://<ip-address>:9090/targets

```

#### Grafana Dashboard
Click On Dashboard --> + symbol --> Import Dashboard --> Use Id 9964 and click on load > Select the data source and click on Import >overview of Jenkins
