It is a security measure to reduce the impact in case of an incident with the service.
It simplifies administration as it becomes easier to track down what resources belong to which service.

## Install Prometheus
Create a system user or system account:
```bash
sudo useradd \
    --system \
    --no-create-home \
    --shell /bin/false prometheus
```
–system – Will create a system account.
–no-create-home – We don’t need a home directory for Prometheus or any other system accounts in our case.
–shell /bin/false – It prevents logging in as a Prometheus user.
Prometheus – Will create a Prometheus user and a group with the same name.

Wget command to download Prometheus
```bash
wget https://github.com/prometheus/prometheus/releases/download/v2.47.1/prometheus-2.47.1.linux-amd64.tar.gz
```
we need to extract all Prometheus files from the archive.
```bash
tar -xvf prometheus-2.47.1.linux-amd64.tar.gz
```
```bash
sudo mkdir -p /data /etc/prometheus
cd prometheus-2.47.1.linux-amd64/
```
First of all, let’s move the Prometheus binary and a promtool to the /usr/local/bin/. promtool is used to check configuration files and Prometheus rules.
```bash
sudo mv prometheus promtool /usr/local/bin/
```
Optionally, we can move console libraries to the Prometheus configuration directory. Console templates allow for the creation of arbitrary consoles using the Go templating language. You don’t need to worry about it if you’re just getting started.
```bash
sudo mv consoles/ console_libraries/ /etc/prometheus/
```
Finally, let’s move the example of the main Prometheus configuration file.
```bash
sudo mv prometheus.yml /etc/prometheus/prometheus.yml
```
To avoid permission issues, you need to set the correct ownership for the /etc/prometheus/ and data directory.
```bash
sudo chown -R prometheus:prometheus /etc/prometheus/ /data/
```
You can delete the archive and a Prometheus folder when you are done.
```bash
cd
rm -rf prometheus-2.47.1.linux-amd64.tar.gz
prometheus --version
```
We’re going to use Systemd, which is a system and service manager for Linux operating systems. For that, we need to create a Systemd unit configuration file.
```bash
sudo vim /etc/systemd/system/prometheus.service
```
Paste this on the file ###prometheus.service
```bash
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
To automatically start the Prometheus after reboot, run enable.
```bash
sudo systemctl enable prometheus
sudo systemctl start prometheus
sudo systemctl status prometheus
```

Now we can try to access it via the browser. I’m going to be using the IP address of the Ubuntu server. You need to append port 9090 to the IP.

## Install Node Exporter on Ubuntu 22.04
Next, we’re going to set up and configure Node Exporter to collect Linux system metrics like CPU load and disk I/O. Node Exporter will expose these as Prometheus-style metrics. Since the installation process is very similar, I’m not going to cover as deep as Prometheus.

Let’s create a system user for Node Exporter by running the following command:
```bash
sudo useradd \
    --system \
    --no-create-home \
    --shell /bin/false node_exporter
```
Use the wget command to download the binary.
```bash
wget https://github.com/prometheus/node_exporter/releases/download/v1.6.1/node_exporter-1.6.1.linux-amd64.tar.gz
```
Extract the node exporter from the archive.
```bash
tar -xvf node_exporter-1.6.1.linux-amd64.tar.gz
```
Move binary to the /usr/local/bin.
```bash
sudo mv \
  node_exporter-1.6.1.linux-amd64/node_exporter \
  /usr/local/bin/
```
Clean up, and delete node_exporter archive and a folder.
```bash
rm -rf node_exporter*
```
Verify that you can run the binary.
```bash
node_exporter --version
```
create a similar systemd unit file.
```bash
sudo vim /etc/systemd/system/node_exporter.service
```
### node_exporter.service
```bash
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
To automatically start the Node Exporter after reboot, enable the service.
```bash
sudo systemctl enable node_exporter
sudo systemctl start node_exporter
sudo systemctl status node_exporter
```

At this point, we have only a single target in our Prometheus. There are many different service discovery mechanisms built into Prometheus. For example, Prometheus can dynamically discover targets in AWS, GCP, and other clouds based on the labels. In the following tutorials, I’ll give you a few examples of deploying Prometheus in a cloud-specific environment. For this tutorial, let’s keep it simple and keep adding static targets. Also, I have a lesson on how to deploy and manage Prometheus in the Kubernetes cluster.

To create a static target, you need to add job_name with static_configs.
```bash
sudo vim /etc/prometheus/prometheus.yml
```
### prometheus.yml
```bash
- job_name: node_export
    static_configs:
      - targets: ["localhost:9100"]
```
![image](https://github.com/aankusshh/Netflix-clone--Pipeline/assets/108898889/f02777fa-a6ad-42ba-a0ca-4e9818da4828)

By default, Node Exporter will be exposed on port 9100.

Since we enabled lifecycle management via API calls, we can reload the Prometheus config without restarting the service and causing downtime.

Before, restarting check if the config is valid
```bash
promtool check config /etc/prometheus/prometheus.yml
```
Then, you can use a POST request to reload the config.
```bash
curl -X POST http://localhost:9090/-/reload
```
Check the targets section at <ip>:9090


## Install Grafana on Ubuntu 22.04
To visualize metrics we can use Grafana. There are many different data sources that Grafana supports, one of them is Prometheus.

First, let’s make sure that all the dependencies are installed.
```bash
sudo apt-get install -y apt-transport-https software-properties-common
```
```bash
wget -q -O - https://packages.grafana.com/gpg.key | sudo apt-key add -
echo "deb https://packages.grafana.com/oss/deb stable main" | sudo tee -a /etc/apt/sources.list.d/grafana.list
sudo apt-get update
sudo apt-get -y install grafana
```
To automatically start the Grafana after reboot, enable the service.
```bash
sudo systemctl enable grafana-server
sudo systemctl start grafana-server
sudo systemctl status grafana-server
```
Go to http://<ip>:3000 and log in to the Grafana using default credentials. The username is admin, and the password is admin as well.
username admin
password admin

To visualize metrics:
1. You need to add a data source first
2. Click Add data source and select Prometheus.
3. For the URL, enter localhost:9090 and click Save and test. You can see Data source is working.
4. Click on Save and Test.
5. Let’s add Dashboard for a better view (+ sign at top right corner)
6. Click on Import Dashboard paste this code 1860 and click on load
7. Select the Datasource and click on Import

You Will get the OUTPUT


```bash

```





























