# Run n8n & Ollama via Kubernetes on your macOS

# 0. Prerequisites
mac book air m3/m4 (above 24GB memory)<br>

# 1. Goal
We will run Local LLM on Kubernetes and create an AI Agent using it using n8n.

<img src="https://github.com/developer-onizuka/n8n-ollama/blob/main/n8n-agent.png" width="960">

# 2. Procedure
### 2-1. Install Hypervisor
>https://www.oracle.com/jp/virtualization/technologies/vm/downloads/virtualbox-downloads.html

### 2-2. Install Vagrant
>https://developer.hashicorp.com/vagrant/install

### 2-3. Install git & git clone
>https://git-scm.com/downloads
```
git clone https://github.com/developer-onizuka/n8n-ollama
cd n8n-ollama
```
### 2-4. Roll out Virtual Machine
### 2-4-1. Master node / Worker node
```
cd kubernetes
vagrant up
cd ..
```
### 2-4-2. NFS Server
```
cd nfs
vagrant up
cd ..
```
### 2-5. Login Master node & git clone
```
cd kubernetes
vagrant ssh master
git clone https://github.com/developer-onizuka/n8n-ollama
cd n8n-ollama
```



<img src="https://github.com/developer-onizuka/n8n-ollama/blob/main/n8n-ollama.png" width="720">
<img src="https://github.com/developer-onizuka/n8n-ollama/blob/main/n8n-model.png" width="720">


```
$ kubectl exec -it <your-ollama-pod-name> -- ollama pull gemma3:1b
```
