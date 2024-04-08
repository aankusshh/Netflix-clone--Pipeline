## Kubectl is to be installed 
```bash
sudo apt update
sudo apt install curl
curl -LO https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
kubectl version --client
```
## Both Master & Node
```bash
sudo apt-get update
sudo apt-get install -y docker.io
sudo usermod –aG docker Ubuntu
newgrp docker
sudo chmod 777 /var/run/docker.sock
```
```bash
sudo apt-get update
```
```bash
sudo apt-get install -y apt-transport-https ca-certificates curl gpg
```
```bash
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.29/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
```
```bash
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.29/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
```
```bash
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

## Master
```bash
sudo kubeadm init --pod-network-cidr=10.244.0.0/16
# in case your in root exit from it and run below commands
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```
Paste the token on worker node

## Install Node_exporter on both master and worker
First, let’s create a system user for Node Exporter by running the following command:
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
Next, create a similar systemd unit file.
```bash
sudo vim /etc/systemd/system/node_exporter.service
```
###node_exporter.service
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
Replace Prometheus user and group to node_exporter, and update the ExecStart command.

To automatically start the Node Exporter after reboot, enable the service
```bash
sudo systemctl enable node_exporter
sudo systemctl start node_exporter
sudo systemctl status node_exporter
```

# Go to prometheus server 
```bash
sudo vim /etc/prometheus/prometheus.yml
```
###prometheus.yml
```bash
- job_name: node_export_masterk8s
    static_configs:
      - targets: ["<master-ip>:9100"]
  - job_name: node_export_workerk8s
    static_configs:
      - targets: ["<worker-ip>:9100"]
```

![image](https://github.com/aankusshh/Netflix-clone--Pipeline/assets/108898889/916b9b03-afdf-40e9-ac20-222bb0457300)

```bash
promtool check config /etc/prometheus/prometheus.yml
curl -X POST http://localhost:9090/-/reload
```

Check the targets section: http://<ip>:9090/targets

## Final stage of PIPELINE
```bash
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
