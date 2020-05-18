今後、ローカル開発環境として利用してほしいため Kubernetes in Docker を利用する
# Pre Setup
下記は自分の検証環境で入っているversionなので特にこだわりなく入れてもらって大丈夫だと思います。
* git
* [docker](https://docs.docker.com/install/) 18.09.7
* [kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/) 1.16.3
* [Golang](https://golang.org/doc/install) 1.13.1

### これでいい感じにダウンロードできろ
研修環境と割り切ってroot で実行する愚行を許せ()
``` 
apt-get install git-core -y
apt-get remove docker docker-engine docker.io containerd runc -y
apt-get update -y
apt-get install -y \
    apt-transport-https \
    ca-certificates \
    curl \
    gnupg-agent \
    software-properties-common
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | apt-key add -
add-apt-repository \
   "deb [arch=amd64] https://download.docker.com/linux/ubuntu \
   $(lsb_release -cs) \
   stable"
apt-get update
apt-get install docker-ce docker-ce-cli containerd.io -y
add-apt-repository ppa:longsleep/golang-backports -y
apt update -y
apt install golang-go -y
curl -LO https://storage.googleapis.com/kubernetes-release/release/v1.18.0/bin/linux/amd64/kubectl
chmod +x ./kubectl
mv ./kubectl /usr/local/bin/kubectl
kubectl version --client
```
# Install kind
`/usr/bin/` 以下に配置しておいた方が取りまわしが良い事も多い
`export` はbashrc やzshrcに入れておくと最終的に楽かも
```shell-session
# export GOPATH=/usr/local/bin/go/bin
# export PATH=$PATH:$GOPATH
# go get -u sigs.k8s.io/kind
```

# Create Cluster
```
# kind create cluster --loglevel debug
Creating cluster "kind" ...
 ✓ Ensuring node image (kindest/node:v1.17.0) 🖼
 ✓ Preparing nodes 📦
 ✓ Writing configuration 📜
 ✓ Starting control-plane 🕹️
 ✓ Installing CNI 🔌
 ✓ Installing StorageClass 💾
Set kubectl context to "kind-kind"
You can now use your cluster with:

kubectl cluster-info --context kind-kind

Thanks for using kind!
```

# setting kubectl
```
# kind get kubeconfig > kubeconfig.yaml
# export KUBECONFIG=./kubeconfig.yaml:~/.kube/config
# kubectl cluster-info
Kubernetes master is running at https://127.0.0.1:38017
KubeDNS is running at https://127.0.0.1:38017/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy

To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.
```

# はじめてのデプロイ [What happens when ... Kubernetes](https://github.com/jamiehannaford/what-happens-when-k8s) 風
詳細が知りたければ[what happens when k8s journy](https://speakerdeck.com/nnao45/what-happens-when-k8s-journy) を読んでください。

## 実行
今、デプロイしたKubernetes cluster にnginx をデプロイします
```
# kubectl run nginx --image=nginx --replicas=3
kubectl run --generator=deployment/apps.v1 is DEPRECATED and will be removed in a future version. Use kubectl run --generator=run-pod/v1 or kubectl create instead.
deployment.apps/nginx created
```

## 確認
今、デプロイしたものを確認します
```
# kubectl get deploy
NAME              READY   UP-TO-DATE   AVAILABLE   AGE
nginx             3/3     3            3           49s

# kubectl get rs
NAME                         DESIRED   CURRENT   READY   AGE
nginx-6db489d4b7             3         3         3       68s

# kubectl get pod
NAME                               READY   STATUS    RESTARTS   AGE
nginx-6db489d4b7-2djdm             1/1     Running   0          5m44s
nginx-6db489d4b7-2vhs8             1/1     Running   0          5m44s
nginx-6db489d4b7-lrgcd             1/1     Running   0          5m44s
```

## 削除
Pod を削除する
```
# kubectl delete pod nginx-6db489d4b7-2djdm nginx-6db489d4b7-2vhs8 nginx-6db489d4b7-lrgcd
pod "nginx-6db489d4b7-2djdm" deleted
pod "nginx-6db489d4b7-2vhs8" deleted
pod "nginx-6db489d4b7-lrgcd" deleted
```
削除したリソースを確認する
```
# kubectl get pod
NAME                               READY   STATUS    RESTARTS   AGE
nginx-6db489d4b7-6sgnz             1/1     Running   0          16s
nginx-6db489d4b7-nhfvh             1/1     Running   0          16s
nginx-6db489d4b7-tl7m8             1/1     Running   0          16s
```
Podを削除しても上位のリソースが存在しているので残り続ける。次にreplicaset を削除する
```
# kubectl get rs
NAME                         DESIRED   CURRENT   READY   AGE
nginx-6db489d4b7             3         3         3       29m

# 次はrs を削除する
# kubectl delete rs/nginx-6db489d4b7
replicaset.apps "nginx-6db489d4b7" deleted

# kubectl get rs
NAME                         DESIRED   CURRENT   READY   AGE
nginx-6db489d4b7             3         3         3       78s

# 上位リソースが削除されたのでpod も削除されました
# kubectl get pod
NAME                               READY   STATUS    RESTARTS   AGE
nginx-6db489d4b7-8cq4x             1/1     Running   0          74s
nginx-6db489d4b7-cgknc             1/1     Running   0          74s
nginx-6db489d4b7-ghm7w             1/1     Running   0          74s
# 区切るのが面倒なのでこのまま進める
#kubectl get deployments.apps
NAME              READY   UP-TO-DATE   AVAILABLE   AGE
nginx             3/3     3            3           34m

# deployments の削除
kubectl delete deployments/nginx
deployment.apps "nginx" deleted

#他のリソースも削除されているので確認してみてください
```

# install yamls 
nginx-deployment.yaml をデプロイします。
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.7.9
        ports:
        - containerPort: 80
```

resourceの取得
```
kubectl get pods rs
```

詳細の取得
```
kubectl describe pod/<name>
kubectl describe deployments <name>
```

resourceの削除
```
kubectl delete pods rs
```

# install [WordPress](https://kubernetes.io/docs/tutorials/stateful-application/mysql-wordpress-persistent-volume/)
* [Example: Deploying WordPress and MySQL with Persistent Volumes](https://kubernetes.io/docs/tutorials/stateful-application/mysql-wordpress-persistent-volume/)を参考にデプロイしてください


そして、デプロイしたものがこちらになります(ローカルのstorageに不安がある場合には事前に`  resources.requests.storage`を少し減らしておくのをオススメします。)
```
●kubectl get -k ./
NAME                           TYPE     DATA   AGE
secret/mysql-pass-c57bb4t7mf   Opaque   1      24m

NAME                      TYPE           CLUSTER-IP     EXTERNAL-IP   PORT(S)        AGE
service/wordpress-mysql   ClusterIP      None           <none>        3306/TCP       27m
service/wordpress         LoadBalancer   10.96.188.51   <pending>     80:30242/TCP   34m

NAME                              READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/wordpress-mysql   1/1     1            1           27m
deployment.apps/wordpress         1/1     1            1           34m

NAME                                   STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
persistentvolumeclaim/mysql-pv-claim   Bound    pvc-4febfa7f-4b5e-4d74-9fe3-a04347f1688d   1Gi       RWO            standard       27m
persistentvolumeclaim/wp-pv-claim      Bound    pvc-fcfa1af2-cb95-4182-b635-273741652a75   1Gi        RWO            standard       34m
```

# install [myapp](https://kind.sigs.k8s.io/docs/user/quick-start/#loading-an-image-into-your-cluster)
KindにはローカルのDockerイメージを読み込む機能が存在している。この章では自分で構築したアプリをコンテナ化して実際にデプロイいたしましす。
main.go
```
package main

import (
  "fmt"
  "net/http"
  "os"
)

func main(){
  http.HandleFunc("/", handler)
  http.ListenAndServe(":8080", nil)
}

func handler(w http.ResponseWriter, r *http.Request){
  msg := os.Getenv("ENENENVVV")
  fmt.Fprintf(w, msg)
}
```

```
FROM golang:latest AS builder

WORKDIR /work
COPY main.go .
RUN go build -o web .

FROM alpine

WORKDIR /exec
COPY --from=builder /work/web .
CMD ["./web"]
```


build
```
docker build -t webweb:v1 .
```
Load
```
kind load docker-image webweb:v1
```

課題
* [ ] myappの[Deployment](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)を書きましょう
* [ ] myappの[Service](https://kubernetes.io/docs/concepts/services-networking/service/)を書きましょう
* [ ] myappのトップページに対して[Liveness](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/) を設定しましょう(HTTPでもTCPでも可)
* [ ] myappに対して[kubectl proxy](https://kubernetes.io/docs/tasks/access-kubernetes-api/http-proxy-access-api/) を利用してアクセスできるようにしましょう

Challenge 課題
* [ ] [METALLB](https://metallb.universe.tf/) をインストールして`type LoadBalancer` としてデプロイしなおしましょう
* [ ] 自分が最近、書いた[cron](https://kubernetes.io/docs/concepts/workloads/controllers/cron-jobs/) をKubernetes として実行してみましょう


# オマケ install [Sock Shop](https://github.com/microservices-demo/microservices-demo)
今後、マイクロサービスをはじめるにあたって[Sock Shop](https://github.com/microservices-demo/microservices-demo)を紹介しておきます。
## 検証環境を構築
```
git clone https://github.com/microservices-demo/microservices-demo
cd microservices-demo/deploy/kubernetes/
kubectl create namespace sock-shop
kubectl apply -f complete-demo.yaml 
kubectl get -f complete-demo.yaml
```
Kind で立ち上げた場合には`service/front-end`には`NodePort` が利用できないのでClusterIP にしてKubernetes からのアクセスができるようになってください
