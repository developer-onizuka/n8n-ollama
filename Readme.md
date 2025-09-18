# Run n8n & Ollama via Kubernetes on your macOS

# 0. Prerequisites
mac book air m3/m4 (above 24GB memory)<br>

# 1. Goal
We will run Local LLM on Kubernetes and create an Workflow for AI Agent using it using n8n.

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
### 2-6. Confirm Kubernetes Cluster
```
kubectl get nodes
```
```
$ kubectl get nodes
NAME      STATUS   ROLES           AGE   VERSION
master    Ready    control-plane   39m   v1.33.5
worker1   Ready    node            38m   v1.33.5
```
### 2-7. Setup LoadBalancer
Specify the range of IP addresses to be assigned to the load balancer.
```
kubectl apply -f metallb-ipaddress.yaml
```
### 2-8. Install CSI driver for NFS
```
./install-csi-driver.sh
```
### 2-9. Roll out StorageClass
```
kubectl apply -f storageclass-vm-nfs.yaml
kubectl apply -f storageclass-vm-nfs-n8n.yaml
```
```
$ kubectl get sc
NAME                       PROVISIONER      RECLAIMPOLICY   VOLUMEBINDINGMODE   ALLOWVOLUMEEXPANSION   AGE
nfs-vm-csi (default)       nfs.csi.k8s.io   Delete          Immediate           false                  8s
nfs-vm-csi-n8n (default)   nfs.csi.k8s.io   Delete          Immediate           false                  6s
```
### 2-10. Roll out PV
```
kubectl apply -f pvc-nfs-ollama.yaml
kubectl apply -f pvc-nfs-n8n.yaml
```
```
$ kubectl get pv
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                    STORAGECLASS     VOLUMEATTRIBUTESCLASS   REASON   AGE
pvc-6614e902-7abc-4865-acc9-f2420f338daa   5Gi        RWX            Delete           Bound    default/pvc-nfs-n8n      nfs-vm-csi-n8n   <unset>                          66s
pvc-8b0667ba-1848-40e5-89f5-0ecaaa0c901d   20Gi       RWX            Delete           Bound    default/pvc-nfs-ollama   nfs-vm-csi       <unset>                          4s
```
### 2-11. Roll out Ollama & n8n
```
kubectl apply -f ollama.yaml
kubectl apply -f n8n.yaml
```
```
$ kubectl get pods -o wide
NAME                      READY   STATUS    RESTARTS   AGE     IP              NODE      NOMINATED NODE   READINESS GATES
n8n-5967965bc6-zwg7n      1/1     Running   0          3m29s   10.10.235.131   worker1   <none>           <none>
ollama-6c988c64c6-gzsck   1/1     Running   0          4m38s   10.10.235.130   worker1   <none>           <none>
```

If you find the error below, you might need to do "sudo chown -R 1000:1000 /home/vagrant/exports-n8n" on your NFS server before applying n8n.yaml.
```
$ kubectl logs n8n-5967965bc6-zwg7n 
No encryption key found - Auto-generating and saving to: /home/node/.n8n/config
No encryption key found - Auto-generating and saving to: /home/node/.n8n/config
Error: EACCES: permission denied, mkdir '/home/node/.n8n'
```

### 2-12. Confirm Services
```
kubectl get services
```
```
$ kubectl get services -o wide
NAME         TYPE           CLUSTER-IP      EXTERNAL-IP    PORT(S)          AGE     SELECTOR
kubernetes   ClusterIP      10.96.0.1       <none>         443/TCP          48m     <none>
svc-n8n      LoadBalancer   10.96.251.15    192.168.33.2   5678:31211/TCP   4m13s   app=n8n
svc-ollama   ClusterIP      10.104.132.75   <none>         11434/TCP        5m22s   app=ollama
```
# 3. Download the model in Ollama as n8n backend
```
kubectl exec -it <your-ollama-pod-name> -- ollama pull gemma3:1b
```
```
$ kubectl exec -it pods/ollama-6c988c64c6-gzsck -- ollama pull gemma3:1b
pulling manifest 
pulling 7cd4618c1faf: 100% ▕██████████████████████████████████████▏ 815 MB                         
pulling e0a42594d802: 100% ▕██████████████████████████████████████▏  358 B                         
pulling dd084c7d92a3: 100% ▕██████████████████████████████████████▏ 8.4 KB                         
pulling 3116c5225075: 100% ▕██████████████████████████████████████▏   77 B                         
pulling 120007c81bf8: 100% ▕██████████████████████████████████████▏  492 B                         
verifying sha256 digest 
writing manifest 
success 
```
# 4. Create Workflow with n8n
### 4-1. Click Create Workflow
<img src="https://github.com/developer-onizuka/n8n-ollama/blob/main/n8n-create-workflow.png" width="720">

### 4-2. Add first step
Drow like below and select the Ollama Chat Model.:<br>
<img src="https://github.com/developer-onizuka/n8n-ollama/blob/main/n8n-myflow.png" width="720">

Set the following:
Model: "gemma3:1b"<br>
<img src="https://github.com/developer-onizuka/n8n-ollama/blob/main/n8n-model.png" width="720">

Credentials to connect with: See image below<br>
<img src="https://github.com/developer-onizuka/n8n-ollama/blob/main/n8n-ollama.png" width="720">

# 5. Testing your workflow
Type message and send it. So, you can find the picture as following and it means your workfrow works:<br>
<img src="https://github.com/developer-onizuka/n8n-ollama/blob/main/n8n-prompt.png" width="720">




