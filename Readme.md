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
# 4. Create Workflow with n8n
### 4-1. Click Create Workflow
<img src="https://github.com/developer-onizuka/n8n-ollama/blob/main/n8n-create-workflow.png" width="720">

### 4-2. Add first step
Drow like below and select the Ollama Chat Model.:<br><br>
<img src="https://github.com/developer-onizuka/n8n-ollama/blob/main/n8n-myflow.png" width="720">

Set the following:
Model: "gemma3:1b"<br><br>
<img src="https://github.com/developer-onizuka/n8n-ollama/blob/main/n8n-model.png" width="720">

Credentials to connect with: See image below<br><br>
<img src="https://github.com/developer-onizuka/n8n-ollama/blob/main/n8n-ollama.png" width="720">

# 5. Testing your workflow
Type message and send it. So, you can find the picture as following and it means your workfrow works:<br><br>
<img src="https://github.com/developer-onizuka/n8n-ollama/blob/main/n8n-prompt.png" width="720">

# 6. AI Agent node with tools

I'll begin by explaining how LLMs and tools work together.Many LLMs, especially modern models like GPT-4 and Gemini, support a feature called Function Calling or Tool Calling. Here's how this process works:<br>
- First, in an n8n AI agent node, you connect tool nodes such as SerpApi. This action tells the LLM about the tool's functions (e.g., Google Search) and its required arguments (e.g., a query).<br>
- The n8n node then sends the user's input (e.g., "What's the current temperature?") along with the defined tool information to the LLM. The LLM analyzes the prompt and determines that this is a question requiring an external search.<br>
- Next, instead of generating a direct answer, the LLM produces special JSON data or text. This data takes the form of an instruction to "call the Google Search tool with 'current temperature' as the query."<br>
- Afterward, the n8n AI agent node interprets the JSON from the LLM. It recognizes that the LLM has decided a tool should be called. The n8n node then uses the specified tool (SerpApi) and arguments (query: 'current temperature') to actually perform the Google search.<br>
- Once the Google search results are retrieved, the n8n node sends those results back to the LLM.<br>
Finally, the LLM uses the provided search results to generate a final answer that is natural and easy for the user to understand (e.g., "The current temperature in Tokyo is...").<br>

When LLMs decide they need to use a tool, it might seem like a vague process at first glance. However, it's actually a logical reasoning process based on the model's training data and the design of its Tool Calling feature. This is because LLMs are not just predicting words probabilistically; they can also break down complex tasks and determine which tool is best suited to solve them.<br>
To be more specific, the n8n node sends the LLM not only the user's input but also a description of the available tools (e.g., Google Search, Image Generator) and an explanation of what each tool is for. This information acts as a clear set of rules that guides the LLM's reasoning. For example, the Google Search tool might have a description that says, "Use this to search for up-to-date information, real-time data, and website content."<br>
In addition, modern LLMs like GPT and Gemini are trained on massive datasets that include patterns of answering questions by referencing external sources. This teaches the model that questions like "What's the weather today?" are tasks that require an external tool, not just internal knowledge. At the same time, the LLM analyzes the user's intent from the prompt. For "What's the weather today?", it analyzes the intent as a request for the latest real-time information. For a question like "What is the capital of Japan?", it can determine that the answer is based on known knowledge. This process is similar to how a person, when asked "What's the weather today?", wouldn't just rely on their memory but would instead check a weather app on their phone. The LLM understands the nature of the task and selects the appropriate tool, just like a person would choose their phone.<br>
While the LLM's decision is probabilistic, behind that probability is strong evidence that calling a tool is the most rational and efficient solution. Therefore, it's not just a vague guess; it's a logical judgment based on a clear intent.<br><br>

最初に、LLMとツールの連携の仕組みについて説明します。多くのLLM、特にGPT-4やGeminiのような最新のモデルは、「関数呼び出し（Function Calling）」 または 「ツール呼び出し（Tool Calling）」 と呼ばれる機能に対応しています。この機能は以下のような仕組みで動作します。
- n8nのAIエージェントノードでは、SerpApiなどのツールノードを接続することで、そのツールの機能（例: Google Search）と、必要な引数（例: query）をLLMに伝えることから始めます。<br>
- まず、n8nノードは、ユーザーの入力（例: 今の気温を知りたい）と、上記で定義したツール情報をLLMに送信します。LLMはプロンプトを分析し、「これは外部検索が必要な質問だ」と判断します。<br>
- 次に、LLMは直接的な回答を生成するのではなく、「Google Searchというツールをqueryに今の気温という引数で呼び出す」 という形式の特殊なJSONデータやテキストを生成します。<br>
- その後、n8nのAIエージェントノードはLLMから返されたこのJSONを解釈し、LLMが「ツールを呼び出すべき」と判断したと認識し、AIエージェントノードは指定されたツール（SerpApi）と引数（query: '今の気温'）を使って、実際にGoogle検索を実行します。<br>
- Google検索の結果が取得されると、n8nノードはその結果を再びLLMに送ります。<br>
- 最終的に、LLMは提供された検索結果を基に、ユーザーにとって自然で分かりやすい最終的な回答（例: 現在の東京の気温は〇〇度です。）を生成します。<br>

なお、LLMが「ツールを呼び出すべき」と判断する過程は、一見すると曖昧なプロセスのように感じますが、そうではなく、モデルの訓練データと、Tool Calling機能の設計に基づいた論理的な推論プロセスです。これは、LLMが単語の確率論的予測を行うだけでなく、複雑なタスクを分解し、どのツールがそのタスクを解決するために最も適しているかを判断する能力を持っているためです。<br>
もう少し詳しく説明すると、n8nのノードは、LLMにユーザーの入力だけでなく、利用可能なツールの機能（例: Google Search、image_generatorなど）と、それらがどのような目的で使われるべきかの説明を同時に渡します。この情報が、LLMの推論をガイドする「明確なルールセット」となります。例えば、Google Searchツールには「最新の情報、リアルタイムデータ、ウェブサイトのコンテンツを検索するために使ってください」といった説明が付与されます。<br>
これに加え、GPTやGeminiのような最新のLLMは、大量のデータセットで訓練されており、その中には「外部の情報源を参照して質問に答える」というパターンも含まれています。これにより、モデルは「今日の天気は？」といった質問が、単なる知識ではなく、外部ツールを必要とするタスクであることを学習しています。同時に、LLMは、ユーザーのプロンプトの「意図」を分析します。例えば、今日の天気であれば、意図として最新のリアルタイム情報が求められていると分析します。また、日本の首都はどこ？という質問であれば、それは既知の知識で回答可能と判断できます。このプロセスは、まるで人間が「今日の天気は？」と聞かれたときに、頭の中の知識だけで答えず、スマホで天気予報を調べるように、LLMがタスクの性質を理解し、適切なツール（スマホ）を選択するのと似ています。<br>
LLMの判断は確率的なものですが、その確率の背後には、ツール呼び出しが最も合理的で効率的な解決策であるという強い根拠が存在します。したがって、これは単なる「曖昧な推測」ではなく、明確な意図に基づく「論理的な判断」となっています。


### 6-1. Chat Memory via MongoDB
You can use the MongoDB as a chat memory, so that the history will be saved in the MongoDB. See below:<br>
You should do configure like this.<br>
<img src="https://github.com/developer-onizuka/n8n-ollama/blob/main/n8n-mongodb.png" width="720">

Then, you can find the history of the chat you've made.<br>
```
$ kubectl exec -it mongodb-statefulset-0 -- mongosh --username admin --password password --authenticationDatabase admin
Current Mongosh Log ID:	68cd2031b32a8de04d4f87fd
Connecting to:		mongodb://<credentials>@127.0.0.1:27017/?directConnection=true&serverSelectionTimeoutMS=2000&authSource=admin&appName=mongosh+2.5.8
Using MongoDB:		8.0.13
Using Mongosh:		2.5.8

For mongosh info see: https://www.mongodb.com/docs/mongodb-shell/

------
   The server generated these startup warnings when booting
   2025-09-19T09:18:33.985+00:00: For customers running the current memory allocator, we suggest changing the contents of the following sysfsFile
   2025-09-19T09:18:33.985+00:00: For customers running the current memory allocator, we suggest changing the contents of the following sysfsFile
   2025-09-19T09:18:33.985+00:00: We suggest setting the contents of sysfsFile to 0.
   2025-09-19T09:18:33.985+00:00: vm.max_map_count is too low
------

test> show dbs
admin            100.00 KiB
config            12.00 KiB
local             72.00 KiB
n8n-chat-memory    8.00 KiB
```
```
test> use n8n-chat-memory
switched to db n8n-chat-memory
n8n-chat-memory> show collections
chat_sessions
n8n_chat_histories
n8n-chat-memory> db.n8n_chat_histories.find()
[
  {
    _id: ObjectId('68cd2e8410810d0090fabc4b'),
    sessionId: '7973f3ebff8c4f519862adf2bd4ba4c8',
    messages: [
      {
        type: 'human',
        data: {
          content: '## Steps to follow\n' +
            '\n' +
            '1. Skip \n' +
            '\n' +
            '\n' +
            '2. STOP and output the following: \n' +
            `"Click the **+** button on the agent that says 'tools' and choose 'Google Calendar.'" \n` +
            ' ----- IGNORE BELOW -----\n' +
            '\n' +
            '\n' +
            '\n' +
            '\n' +
            '\n' +
            '\n' +
            '\n' +
            '\n' +
            '\n' +
            '\n' +
            '\n' +
            '\n' +
            '\n' +
            '\n' +
            '\n' +
            '\n' +
            '\n' +
            '8. If all steps are completed, output the following:\n' +
            '"Would you like me to check all events in your calendar for tomorrow 2025-09-20?"\n' +
            '\n' +
            '# User message\n' +
            '\n' +
            'Hello. My name is developer-onizuka.',
          additional_kwargs: {},
          response_metadata: {}
        }
      },
      {
        type: 'ai',
        data: {
          content: "Nice to meet you, Developer-Onizuka! 👋 Let's get started with setting up our schedule. Please click the **+** button on the agent that says 'tools' and choose 'Google Calendar'. ✨",
          tool_calls: [],
          invalid_tool_calls: [],
          additional_kwargs: {},
          response_metadata: {}
        }
      },
      {
        type: 'human',
        data: {
          content: '## Steps to follow\n' +
            '\n' +
            '1. Skip \n' +
            '\n' +
            '\n' +
            '2. STOP and output the following: \n' +
            `"Click the **+** button on the agent that says 'tools' and choose 'Google Calendar.'" \n` +
            ' ----- IGNORE BELOW -----\n' +
            '\n' +
            '\n' +
            '\n' +
            '\n' +
            '\n' +
            '\n' +
            '\n' +
            '\n' +
            '\n' +
            '\n' +
            '\n' +
            '\n' +
            '\n' +
            '\n' +
            '\n' +
            '\n' +
            '\n' +
            '8. If all steps are completed, output the following:\n' +
            '"Would you like me to check all events in your calendar for tomorrow 2025-09-20?"\n' +
            '\n' +
            '# User message\n' +
            '\n' +
            'Do you remember my name?',
          additional_kwargs: {},
          response_metadata: {}
        }
      },
      {
        type: 'ai',
        data: {
          content: "I'm sorry that slips my mind! Let's get back on track. Click the **+** button on the agent that says 'tools' and choose 'Google Calendar'. ✨",
          tool_calls: [],
          invalid_tool_calls: [],
          additional_kwargs: {},
          response_metadata: {}
        }
      },
      {
        type: 'human',
        data: {
          content: '## Steps to follow\n' +
            '\n' +
            '1. Skip \n' +
            '\n' +
            '\n' +
            '2. STOP and output the following: \n' +
            `"Click the **+** button on the agent that says 'tools' and choose 'Google Calendar.'" \n` +
            ' ----- IGNORE BELOW -----\n' +
            '\n' +
            '\n' +
            '\n' +
            '\n' +
            '\n' +
            '\n' +
            '\n' +
            '\n' +
            '\n' +
            '\n' +
            '\n' +
            '\n' +
            '\n' +
            '\n' +
            '\n' +
            '\n' +
            '\n' +
            '8. If all steps are completed, output the following:\n' +
            '"Would you like me to check all events in your calendar for tomorrow 2025-09-20?"\n' +
            '\n' +
            '# User message\n' +
            '\n' +
            'Do you remember my name?',
          additional_kwargs: {},
          response_metadata: {}
        }
      },
      {
        type: 'ai',
        data: {
          content: "My apologies again! Yes, I recall that it's developer-onizuka. Please click the **+** button on the agent that says 'tools' and choose 'Google Calendar'. ✨",
          tool_calls: [],
          invalid_tool_calls: [],
          additional_kwargs: {},
          response_metadata: {}
        }
      }
    ]
  }
]
```

### 6-2. Calculator for LLM
As you know LLM is not good at mathmatics, so you should deploy the calculator for the case of calculation.<br>

<img src="https://github.com/developer-onizuka/n8n-ollama/blob/main/n8n-square-root.png" width="720">


