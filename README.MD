#### 概要
この記事では*Minikube*で動く単一のクラスタ上に、Python負荷検証ツール*Locust*を**複数**構築する。テスト対象サーバに対して、複数のLocustから**一斉**に負荷をかけられるようになるまでの環境構築手順を記す。

# 導入と背景（読み飛ばしてOK）
GKE上に存在するサービスの負荷検証をLocustで実施しておりました。対象サービスに対してAPIを実行しまくって、パルス状の負荷をかけた時に、設定通りにpodが増加してくれるかどうかを確認したかったのですが、どうやらローカルのLocustだけではそれを実現することができませんでした（ユーザ数を1000人にしてもダメ）。

どうにかAPIを使用してパルス状の負荷をかけたくて調査していると、Kubernatesにpodを複数生成して、その複数podからLocustで負荷をかける方法があることを知りました。しかし、著者はKubernates弱者であり、いきなり開発環境にあれやこれやするのは困難であるため、まずはLocalで扱えるMinikubeで同等なことができるかを検証したいな、と考えあれやこれや試しました。

この記事は上記の事情から書かれたものであり、きっと不備や不足している箇所があるかと思います。もしそのようなものを見つけた場合には、みなさんの知識・知見等をぜひとも共有していただけると幸いです🙇

# 準備
## 前提条件
次がローカルの環境に備わっていることが前提になる：
- Docker
- kubectl
- Minikube
- Python3

## シンプルな紹介

今回使用する主要なサービスについて非常に簡易的に紹介する。

### Locust
Pythonで書かれた負荷検証サービスである。

https://locust.io/

### Minikube
現状、コンテナオーケストレーションサービスのデファクトスタンダードである*Kubernates*をローカルで構築可能にしたサービス。

https://minikube.sigs.k8s.io/docs/

# 構成

## Pod構成
リソースの関係は下記図である。この図において、矢印は通信の起点から接続先への到達を示すものであり、リクエストの送受信方向を全て表しているわけではない。
![](https://storage.googleapis.com/zenn-user-upload/848f1e10ffe1-20250514.png)

複数のworker-pod（図では2個）がサーバへ負荷をかけるために、APIへリクエストし続ける。


## ファイル構成
非常に単純な構成にしてある。
```
.
├── Dockerfile.locust
├── Dockerfile.server
├── fastapi-deployment.yaml
├── locust-master.yaml
├── locust-worker.yaml
├── locustfile.py
└── server.py
```


# 構築手順

## Contextを確認する

Local PCにあるMinikubeでリソース構築を確実なものにするために、まずはcontextを確認する。
下記でcontextを確認できる：
```
kubectl config current-context 
```
この結果、
```
minikube
```
と表示されれば先に進める。本番環境を誤って変更しないためにも、必ず確認されたい。

もし、contextがminikubeではない場合、次の記事を参考せよ。

https://www.skyarch.net/blog/kubernetes%E5%85%A5%E9%96%80-kubectl%E3%81%AE%E3%82%A2%E3%82%AF%E3%82%BB%E3%82%B9%E5%85%88%E3%82%B3%E3%83%B3%E3%83%86%E3%82%AD%E3%82%B9%E3%83%88%E3%82%92%E5%88%87%E3%82%8A%E6%9B%BF%E3%81%88/


## サーバをpodに構築

```python:server.py
from fastapi import FastAPI
from fastapi.responses import JSONResponse

app = FastAPI()

@app.get("/hello")
async def hello():
    return JSONResponse(content={"message": "Hello from FastAPI!"})
```

```Dockerfile:Dockerfile.server
# Dockerfile
FROM python:3.11-slim

WORKDIR /app

COPY server.py .

RUN pip install fastapi uvicorn

CMD ["uvicorn", "server:app", "--host", "0.0.0.0", "--port", "8000"]
```

Minikube はデフォルトでローカル Docker を参照しないため、以下のコマンドで切り替える：
```sh
eval $(minikube docker-env -u)
```
その後、
```sh
docker build -f Dockerfile.server -t fastapi-server:latest .
```

Minikubeにpodを作成するためのYAMLを与える：
```yaml:fastapi-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: fastapi-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      app: fastapi
  template:
    metadata:
      labels:
        app: fastapi
    spec:
      containers:
        - name: fastapi
          image: fastapi-server:latest
          imagePullPolicy: Never
          ports:
            - containerPort: 8000
---
apiVersion: v1
kind: Service
metadata:
  name: fastapi-service
spec:
  selector:
    app: fastapi
  ports:
    - protocol: TCP
      port: 8000
      targetPort: 8000
  type: ClusterIP
```

そして適用する：

```sh
kubectl apply -f fastapi-deployment.yaml
```

確認のために`kube get pods`でpodの状態を確認できる。

## Locustをpodに構築

LocustのPythonファイルは下記のようになっている。
helloというAPIを実行する。
```python:locustfile.py
from locust import HttpUser, task, between

class SimpleUser(HttpUser):
    wait_time = between(1, 2)

    @task
    def hello(self):
        self.client.get("/hello")
```

Dockerfileは下記である。
```Dockerfile:Dockerfile.locust
FROM locustio/locust

COPY locustfile.py /locustfile.py
```
サーバの時と同様に、Minikube VM内部で`docker build`するため
```
eval $(minikube docker-env -u)
```
を実行して、
```
docker build -f Dockerfile.locust -t my-locust-image .
```
とする。

次に、master/worker用のpodを作成する。pod数はそれぞれ次のようにしておく：

- master-pod: 1個
- worker-pod: 2個

worker-podの数は必ずしも2個である必要はないことには留意してほしい。Locustで4つのworker threadを立てたければ下記で示すYAMLの該当箇所を4にすればよい。

まず、master-pod用のYAMLは
```yaml:locust-master.yaml
apiVersion: v1
kind: Service
metadata:
  name: locust-master
spec:
  selector:
    app: locust
    role: master
  ports:
    - protocol: TCP
      port: 8089
      targetPort: 8089
      name: http
    - protocol: TCP
      port: 5557
      targetPort: 5557
      name: locust-task
    - protocol: TCP
      port: 5558
      targetPort: 5558
      name: locust-health
  type: NodePort

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: locust-master
spec:
  replicas: 1
  selector:
    matchLabels:
      app: locust
      role: master
  template:
    metadata:
      labels:
        app: locust
        role: master
    spec:
      containers:
        - name: locust
          image: my-locust-image
          imagePullPolicy: Never
          args: ["-f", "/locustfile.py", "--master", "--host=http://fastapi-service:8000"]
          ports:
            - containerPort: 8089
            - containerPort: 5557
            - containerPort: 5558
```

である。続いて、worker用のYAMLは

```yaml:locust-worker.yaml
# locust-worker.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: locust-worker
spec:
  replicas: 2
  selector:
    matchLabels:
      app: locust
      role: worker
  template:
    metadata:
      labels:
        app: locust
        role: worker
    spec:
      containers:
        - name: locust
          image: my-locust-image
          imagePullPolicy: Never
          args: ["-f", "/locustfile.py", "--worker", "--master-host=locust-master"]
```
である。これら2つのYAMLをMinikubeにapplyする：
```sh
kubectl apply -f locust-master.yaml
kubectl apply -f locust-worker.yaml
```

--- 

以上を実行した上で、一旦podの状況を確認する。
```
kubectl get pods
```
を実行して

![](https://storage.googleapis.com/zenn-user-upload/4a1f7e03b522-20250513.png)

のうような表示が現れればよい。注目するところは`STATUS`の欄で、全て*Running*状態であることを確認しておく。また、念のために
```sh
kubectl get services
```
コマンドを実行すると

![](https://storage.googleapis.com/zenn-user-upload/5b68609a5c9e-20250513.png)

のようになっている。

- fastapi-service: port 8000
- locust-master: port 8089, 5557, 5558

が我々がコントロールしている部分であるので、設定の間違いがないことを確認する。
※画像のkubernatesは無視してよい。



次のコマンドでport 8089に対応するアドレスを確認する。
```sh
kubectl get svc locust-master -o json | jq '.spec.ports'
```
すると
![](https://storage.googleapis.com/zenn-user-upload/60b17bcb9b12-20250513.png)
が表示されるので、port 8089に対応するURLにアクセスする。
※画像例では[http://localhost:31642/](http://localhost:31642/)にアクセス。

※jqコマンドが環境に存在しない場合はHomebrewでインストールするとよい。Macでは
```
brew install jq
```

# 結果
上記のURLにアクセスすることで、Locustを使用することができる。画像のWorkerに着目してほしい。確かに、**Workerが2と表示**されている。
![](https://storage.googleapis.com/zenn-user-upload/9441a0f0b9c3-20250513.png)


下記画像はworkerの数を4にした場合の別タブ画面である。APIをリクエストするユーザ数を10と設定しているため、それぞれのworker threadに3, 3, 2, 2とユーザ数が割り振られている。
![](https://storage.googleapis.com/zenn-user-upload/71829bca8081-20250513.png)

# 付録

## クリーンアップの方法
構築したpodは次で削除できる：

```sh
kubectl delete -f locust-master.yaml
kubectl delete -f locust-worker.yaml     
kubectl delete -f fastapi-deployment.yaml
```



# 参考文献
https://hub.docker.com/r/locustio/locust

https://zeromq.org/

https://minikube.sigs.k8s.io/docs/start/?arch=%2Fmacos%2Farm64%2Fstable%2Fbinary+download

https://zenn.dev/ry/articles/97a5c50b1ddf16

https://zenn.dev/empenguin/articles/eda7535aeb0977

# 更新履歴
- 2025-05-14: jqコマンドに対する文言を追加