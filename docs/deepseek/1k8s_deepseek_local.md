# 在 Kubernetes上免费部署Ollama+DeepSeek

## Ollama 和 Kubernetes 的结合优势

二者结合后，我们可以快速部署 Ollama 服务器，并通过 API 与 DeepSeek 模型进行交互。


**Ollama ：通过 REST API 简化了模型服务的部署和调用，支持多种机器学习模型。**

**Kubernetes ：提供灵活的扩展性和高可用性，适合部署复杂的模型服务。**

![Alt Image Text](../images/ds1_1.png "Body image")

## 部署Ollama+ DeepSeek详细步骤：

为 Ollama 创建一个专用命名空间。

```
kubectl creat ns ollama
```

```
kubectl get ns | grep ollama
ollama            Active   22h
```

创建一个 `deploy.yaml` 文件，定义 Ollama 的部署和模型的加载逻辑：

```

apiVersion: apps/v1
kind: Deployment
metadata:
  name: ollama
  namespace: ollama
  labels:
    app: ollama
spec:
  replicas: 1
  selector:
    matchLabels:
      app: ollama
  template:
    metadata:
      labels:
        app: ollama
    spec:
      containers:
        - image: ollama/ollama:latest
          imagePullPolicy: Always
          name: ollama
          ports:
            - containerPort: 11434
              protocol: TCP
        - name: load-model
          image: curlimages/curl
          imagePullPolicy: Always
          args:
            - sleep infinity
          command:
            - /bin/sh
            - '-c'
          lifecycle:
            postStart:
              exec:
                command:
                  - /bin/sh
                  - '-c'
                  - >
                    curl -X POST http://localhost:11434/api/pull -H
                    'Content-Type: application/json' -d '{"name": "llama3.2"}'
          resources:
            limits:
              cpu: 25m
              memory: 50Mi
            requests:
              cpu: 25m
              memory: 50Mi
```

```
kubectl apply -f deploy.yaml

% kubectl get pod -n ollama
NAME                      READY   STATUS    RESTARTS      AGE
ollama-79ffc59d74-rjd8l   2/2     Running   2 (21h ago)   22h
```

### 暴露 Ollama API 服务

为了与 Ollama 的 REST API 交互，我们需要将服务暴露出来。可以选择使用 LoadBalancer、NodePort 或 ClusterIP 类型的服务。

以下是一个示例服务配置文件 service.yaml

```
apiVersion: v1
kind: Service
metadata:
  name: ollama
  namespace: ollama
  labels:
    app: ollama
spec:
  type: LoadBalancer
  ports:
  - port: 11434
    targetPort: 11434
    protocol: TCP
    name: http
  selector:
    app: ollama
```

与 Ollama + Deepseek 交互：

```
% kubectl get svc -n ollama 
NAME     TYPE           CLUSTER-IP        EXTERNAL-IP    PORT(S)           AGE
ollama   LoadBalancer   192.168.194.211   198.19.249.2   11434:30330/TCP   22h
```

### 与 Ollama + Deepseek 交互：


一旦 Ollama 部署完成，我们可以通过其 REST API 与模型交互。在之前的部署中，我们添加了load-model容器，用来与 Ollama 的API接口交互调用，我们进入容器内部，使用curl相关命令即可。'

a) 列出可用模型

**使用 GET 请求检索 Ollama 环境中所有可用模型的列表：**

```
% curl http://localhost:11434/api/tags | jq

  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   344  100   344    0     0  11075      0 --:--:-- --:--:-- --:--:-- 11096
{
  "models": [
    {
      "name": "llama3.2:latest",
      "model": "llama3.2:latest",
      "modified_at": "2025-02-07T14:01:25.492244029Z",
      "size": 2019393189,
      "digest": "a80c4f17acd55265feec403c7aef86be0c25983ab279d83f3bcd3abbcb5b8b72",
      "details": {
        "parent_model": "",
        "format": "gguf",
        "family": "llama",
        "families": [
          "llama"
        ],
        "parameter_size": "3.2B",
        "quantization_level": "Q4_K_M"
      }
    }
  ]
}
```

要下载并加载特定模型（例如，deepseek-r1:8b），发送 POST 请求到 `/api/pull`：

```
curl -X POST http://localhost:11434/api/pull \
     -H 'Content-Type: application/json' \
     -d '{
           "name": "deepseek-r1:1.5b"
         }'
```

```
{"status":"verifying sha256 digest"}
{"status":"writing manifest"}
{"status":"success"}
```

c) 使用DeepSeek模型

加载模型后，可以通过 /api/generate 接口与模型交互：

```
curl -X POST http://localhost:11434/api/generate \
     -H 'Content-Type: application/json' \
     -d '{
           "model": "deepseek-r1:1.5b",
           "prompt": "In exactly 10 words or less, explain why the sea is blue.",
           "stream": false
         }'
     
```

示例返回结果：

![Alt Image Text](../images/ds1_2.png "Body image")