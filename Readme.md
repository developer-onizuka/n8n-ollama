# Run n8n & Ollama via Kubernetes on your macOS

# 0. Prerequisites
mac book air m3/m4 (above 24GB memory)<br>

# 1. Goal
We will run Local LLM on Kubernetes and create an Workflow for AI Agent using n8n.

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

If you find the error below, you might need to do "**sudo chown -R 1000:1000 /home/vagrant/exports-n8n**" on your NFS server before applying n8n.yaml.
```
$ kubectl logs n8n-5967965bc6-zwg7n 
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
kubectl exec -it <your-ollama-pod-name> -- ollama pull llama3.2:3b
```
```
$ kubectl exec -it pods/ollama-6c988c64c6-7fxfp -- ollama pull llama3.2:3b
pulling manifest 
pulling dde5aa3fc5ff: 100% ▕██████████████████████████████████████▏ 2.0 GB                         
pulling 966de95ca8a6: 100% ▕██████████████████████████████████████▏ 1.4 KB                         
pulling fcc5a6bec9da: 100% ▕██████████████████████████████████████▏ 7.7 KB                         
pulling a70ff7e570d9: 100% ▕██████████████████████████████████████▏ 6.0 KB                         
pulling 56bb8bd477a5: 100% ▕██████████████████████████████████████▏   96 B                         
pulling 34bb5ab01051: 100% ▕██████████████████████████████████████▏  561 B                         
verifying sha256 digest 
writing manifest 
success
```
# 4. Create Workflow with n8n as tests
### 4-1. Check the tunnel URL
```
$ kubectl logs n8n-5f5c8cb87c-xwqw8 --follow
Permissions 0660 for n8n settings file /home/node/.n8n/config are too wide. Changing permissions to 0600..
Initializing n8n process
n8n ready on ::, port 5678
n8n Task Broker ready on 127.0.0.1, port 5679
[license SDK] Skipping renewal on init: license cert is not initialized
Registered runner "JS Task Runner" (VHheNdOPg6bwO6h4EnKaU) 
[Data table] The Data table module is experimental and subject to change.
Any tables added before the official release may become inaccessible at any point. Use at your own risk.

Waiting for tunnel ...
Tunnel URL: https://orange-eagle-76.hooks.n8n.cloud/
```

### 4-2. Click Create Workflow
Go to the result of above with your browser so that you can login to the n8n page.:<br><br>
<img src="https://github.com/developer-onizuka/n8n-ollama/blob/main/n8n-create-workflow.png" width="720">

### 4-3. Add first step
Drow like below and select the Ollama Chat Model.:<br><br>
<img src="https://github.com/developer-onizuka/n8n-ollama/blob/main/n8n-myflow.png" width="720">

Credentials to connect with: See image below<br><br>
<img src="https://github.com/developer-onizuka/n8n-ollama/blob/main/n8n-ollama.png" width="720">

# 5. Run your workflow
Type message and send it. So, you can find the picture as following and it means your workfrow works:<br><br>
<img src="https://github.com/developer-onizuka/n8n-ollama/blob/main/n8n-prompt.png" width="720">

# 6. AI Agent node with tools

### 6-1. Google Search for LLM
<img src="https://github.com/developer-onizuka/n8n-ollama/blob/main/n8n-SerpAPI.png" width="720">

I'll begin by explaining how LLMs and tools work together. Many LLMs, especially modern models like GPT-4 and Gemini, support a feature called Function Calling or Tool Calling. Here's how this process works:<br>
- First, in an n8n **AI agent node** (not Basic Chain in like #4), you connect tool nodes such as SerpApi. This action tells the LLM about the tool's functions (e.g., Google Search) and its required arguments (e.g., a query).<br>
- The n8n node then sends the user's input (e.g., "What's the current temperature?") along with the defined tool information to the LLM. The LLM analyzes the prompt and determines that this is a question requiring an external search.<br>
- Next, instead of generating a direct answer, the LLM produces special JSON data or text. This data takes the form of an instruction to "call the Google Search tool with 'current temperature' as the query."<br>
- Afterward, the n8n AI agent node interprets the JSON from the LLM. It recognizes that the LLM has decided a tool should be called. The n8n node then uses the specified tool (SerpApi) and arguments (query: 'current temperature') to actually perform the Google search.<br>
- Once the Google search results are retrieved, the n8n node sends those results back to the LLM.<br>
Finally, the LLM uses the provided search results to generate a final answer that is natural and easy for the user to understand (e.g., "The current temperature in Tokyo is...").<br>

When LLMs decide they need to use a tool, it might seem like a vague process at first glance. However, it's actually a logical reasoning process based on the model's training data and the design of its Tool Calling feature. This is because LLMs are not just predicting words probabilistically; they can also break down complex tasks and determine which tool is best suited to solve them.<br>
To be more specific, the n8n node sends the LLM not only the user's input but also a description of the available tools (e.g., Google Search, Image Generator) and an explanation of what each tool is for. This information acts as a clear set of rules that guides the LLM's reasoning. For example, the Google Search tool might have a description that says, "Use this to search for up-to-date information, real-time data, and website content."<br>
In addition, modern LLMs like GPT and Gemini are trained on massive datasets that include patterns of answering questions by referencing external sources. This teaches the model that questions like "What's the weather today?" are tasks that require an external tool, not just internal knowledge. At the same time, the LLM analyzes the user's intent from the prompt. For "What's the weather today?", it analyzes the intent as a request for the latest real-time information. For a question like "What is the capital of Japan?", it can determine that the answer is based on known knowledge. This process is similar to how a person, when asked "What's the weather today?", wouldn't just rely on their memory but would instead check a weather app on their phone. The LLM understands the nature of the task and selects the appropriate tool, just like a person would choose their phone.<br>
While the LLM's decision is probabilistic, behind that probability is strong evidence that calling a tool is the most rational and efficient solution. Therefore, it's not just a vague guess; it's a logical judgment based on a clear intent.<br><br>

### 6-2. Calculator for LLM
As you know LLM is not good at mathmatics, so you should deploy the calculator for the case of calculation.<br><br>
<img src="https://github.com/developer-onizuka/n8n-ollama/blob/main/n8n-square-root.png" width="720">

### 6-3. etc
You can find so many tools in n8n.<br>

# 7. Chat Memory via MongoDB
### 7-1. Roll out MongoDB with a perpetual disk
```
kubectl apply -f storageclass-vm-nfs-n8n.yaml
kubectl apply -f pvc-nfs-mongodb.yaml
kubectl apply -f mongodb.yaml
```

### 7-2. Connect MongoDB with AI Agent Node
You can use the MongoDB as a chat memory, so that the history will be saved in the MongoDB. See below:<br>
You should do configure like this.<br><br>
<img src="https://github.com/developer-onizuka/n8n-ollama/blob/main/n8n-mongodb.png" width="720">


After that, you can see some messages based on previous conversation with LLM.<br>
You can also find the history of the chat you've made.<br>
```
kubectl exec -it mongodb-statefulset-0 -- mongosh --username admin --password password --authenticationDatabase admin
```
```
test> show dbs
test> use n8n-chat-memory
n8n-chat-memory> show collections
n8n-chat-memory> db.n8n_chat_histories.find()
```

# 8. AI Agent for Searching Restaurants
Now, based on what we know so far, let's create a real working AI Agent.<br><br>
<img src="https://github.com/developer-onizuka/n8n-ollama/blob/main/n8n-restaurant.png" width="960">

### 8-1. Trigger manually
<img src="https://github.com/developer-onizuka/n8n-ollama/blob/main/trigger-manually.png" width="960">

### 8-2. HTTP Request
<img src="https://github.com/developer-onizuka/n8n-ollama/blob/main/http-request.png" width="960">

URL:
```
https://api.apify.com/v2/acts/lukaskrivka~google-maps-with-contact-details/run-sync-get-dataset-items?token=<your-token>
```
JSON:
```
{
    "language": "ja",
    "locationQuery": "{{ $json.location }}",
    "maxCrawledPlacesPerSearch": 3,
    "searchStringsArray": [
        "{{ $json.query }}"
    ],
    "skipClosedPlaces": false
}
```

### 8-3. Loop over items
<img src="https://github.com/developer-onizuka/n8n-ollama/blob/main/loop-over-items.png" width="960">

### 8-4. Edit Fields
<img src="https://github.com/developer-onizuka/n8n-ollama/blob/main/edit-field.png" width="960">

### 8-5. AI Agent
You should attach the Ollama Chat Model as brain and the SerAPI tool for google search like #6-1.<br><br>
<img src="https://github.com/developer-onizuka/n8n-ollama/blob/main/ai-agent.png" width="960">

System Message:
```
You are an expert on restaurant information. Using the provided information and, if necessary, internet searches, please create a response in the following format:

#1 The name of restaurant: <name>
#2 Main Menu: <menu>
#3 Target Customers: <customer base>
#4 Price Range: <price range>
#5 The nearest station: <name>

If the information is insufficient, please state clearly, "Information is insufficient and I cannot answer."
When generating the response, you may use internal thought processes and tools (e.g., search tools), but do not include their content in the final answer. 
If the input string is Unicode escaped, please decode and process it correctly. Be sure to include two line breaks at the end. Present only the final answer to the user in Japanese.
```

### 8-6. Convert to File
<img src="https://github.com/developer-onizuka/n8n-ollama/blob/main/convert-to-file.png" width="960">

### 8-7. Read/Write Files from Disk
<img src="https://github.com/developer-onizuka/n8n-ollama/blob/main/write-file.png" width="960">

### 8-8. Check the result
```
kubectl exec -it <the-pod-name-for-n8n> -- cat /tmp/file.txt
```
```
#1 レストラン香港 日本式洋食レストラン
#2 主菜 (ありますか?) 
#3 ターゲット カスタマー 
#4 価格 Range 
#5 最寄り駅 : 都営大江戸線落合南長崎駅

#1 エシャロット フランス料理店
#2 メニュは、フランスのクラシックカーニバル、海鮮デリシャス、ヨーグルトとドレスिंングのためのエッセンスが含まれます。
#3 テーケスト クラスター：エトリアン、フランス人、ファミリー
#4 Price Range: 8,000 - 20,000円（メニュー）
#5 最寄りの駅: 新宿駅（約200m）

#1 ぺいざん
#2 カモメ（カリフラワー）&トランプルカ、ハンバーグ&ポークバンマス、ウナギ、ビーフミックスボール、パスタなど
#3 ターゲットカスタマー は日本人や洋食を愛する日本国外の出身者、特にアメリカ系やイギリス系の留学生などの若い大衆
#4 Price Range 高値（約 2,000 - 5,000 JPY）から中価値（約 1,500 - 3,000 JPY）まで
#5 The nearest station 新宿駅（JR山手線・新宿線、東京メトロ日暮里線）
```

# 9. AI Agent for RAG
n8n has several mechanisms for creating chatbots that use RAG (RAG chatbots). This time, we will introduce and explain what is probably the easiest way.
Specifically, we will create a RAG chatbot using a node called "Simple Vector Store". The completed workflow is shown below.<br><br>
<img src="https://github.com/developer-onizuka/n8n-ollama/blob/main/n8n-rag.png" width="960">

### 9-1. On form submission
This is a core node that can start a workflow when form input is submitted.<br><br>
<img src="https://github.com/developer-onizuka/n8n-ollama/blob/main/rag-on-form-submission.png" width="480">

### 9-2. Simple Vector Store
Download the open embedding model with a large token context window, so that you can select LLM for simple vector store. The nomic-embed-text is an open-source, high-performance text embedding model developed by Nomic AI that is used to convert text data into a numerical representation (vectors) that can be used in tasks such as search, classification, and clustering.<br><br>
```
$ kubectl exec -it pods/ollama-6c988c64c6-qgsdm -- ollama pull nomic-embed-text:latest
pulling manifest 
pulling 970aa74c0a90: 100% ▕██████████████████████████████████████▏ 274 MB                         
pulling c71d239df917: 100% ▕██████████████████████████████████████▏  11 KB                         
pulling ce4a164fc046: 100% ▕██████████████████████████████████████▏   17 B                         
pulling 31df23ea7daa: 100% ▕██████████████████████████████████████▏  420 B                         
verifying sha256 digest 
writing manifest 
success 
```
<img src="https://github.com/developer-onizuka/n8n-ollama/blob/main/rag-embeddings-ollama.png" width="320"> <img src="https://github.com/developer-onizuka/n8n-ollama/blob/main/rag-default-data-loader.png" width="320">

### 9-3. AI Agent
This is the AI Agent which uses vector store and it works as a chatbot powered by RAG.<br><br>
<img src="https://github.com/developer-onizuka/n8n-ollama/blob/main/rag-simple-vector-store.png" width="320">

Description:
```
保存されてるナレッジを使用して、ユーザーからの質問に回答してください。
```
### 9-4. Upload pdf file and run the AI Agent
Use the pdf file below:<br>
- https://github.com/developer-onizuka/LangChain/blob/main/myFirstPdf.pdf

<img src="https://github.com/developer-onizuka/n8n-ollama/blob/main/rag-upload-pdf.png" width="320">


### 9-5. Check the result
<img src="https://github.com/developer-onizuka/n8n-ollama/blob/main/rag-result.png" width="540">
