
# Simple manifest

> It is a simple configuration for learning kubernetes.

### pod

[pod](https://kubernetes.io/docs/concepts/workloads/pods/pod/)

> Pods are the smallest deployable units of computing that can be created and managed in Kubernetes.

```sh
$ kubectl apply -f simple-pod.yaml
pod/simple-echo created
$ kubectl get pod
NAME          READY   STATUS    RESTARTS   AGE
simple-echo   2/2     Running   0          15s
```

### ReplicaSet

[ReplicaSet](https://kubernetes.io/docs/concepts/workloads/controllers/replicaset/)

```
A mechanism for adjusting and managing Pod to number of replicas based on the Pod template
```

```sh
$ kubectl apply -f simple-replica.yaml
replicaset.apps/echo created
$ kubectl get pod
NAME         READY   STATUS    RESTARTS   AGE
echo-j9cjm   2/2     Running   0          26s
echo-pw2nn   2/2     Running   0          8s
echo-xx8h6   2/2     Running   0          26s
```

### deployments

[Deployments](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)

> A Deployment controller provides declarative updates for Pods and ReplicaSets.

```sh
$ kubectl create -f simple-deploy.yaml --record
deployment.apps/echo created

$ kubectl get pod,replicaset,deployment --selector app=echo
NAME                        READY   STATUS    RESTARTS   AGE
pod/echo-6894b565b9-8cdw7   2/2     Running   0          48s
pod/echo-6894b565b9-n9w6c   2/2     Running   0          48s
pod/echo-6894b565b9-rzktg   2/2     Running   0          48s

NAME                                    DESIRED   CURRENT   READY   AGE
replicaset.extensions/echo-6894b565b9   3         3         3       48s

NAME                         DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
deployment.extensions/echo   3         3         3            3           48s

$ kubectl rollout history deployment echo
deployment.extensions/echo
REVISION  CHANGE-CAUSE
1         kubectl create --filename=simple-deploy.yaml --record=true
```

* Changing the number of pod does not generate ReplicaSet
  (podの数を変更してもReplicaSetは生成されない)
* Changing the container image updates the revision of deployments
  (コンテナイメージを変更するとdeploymentsのリビジョンは更新される)

```sh
$ kubectl rollout history deployment echo
deployment.extensions/echo
REVISION  CHANGE-CAUSE
1         kubectl apply --filename=simple/simple-deploy.yaml --record=true
2         kubectl apply --filename=simple/simple-deploy.yaml --record=true
```

## Rollback

```sh
$ kubectl rollout history deployment echo
deployment.extensions/echo
REVISION  CHANGE-CAUSE
1         kubectl apply --filename=simple/simple-deploy.yaml --record=true
2         kubectl apply --filename=simple/simple-deploy.yaml --record=true

$ kubectl rollout history deployment echo --revision=1
deployment.extensions/echo with revision #1
Pod Template:
  Labels:	app=echo
	pod-template-hash=2450612165
  Annotations:	kubernetes.io/change-cause: kubectl apply --filename=simple/simple-deploy.yaml --record=true
  Containers:
   nginx:
    Image:	gihyodocker/nginx:latest
    Port:	80/TCP
    Host Port:	0/TCP
    Environment:
      BACKEND_HOST:	localhost:8080
    Mounts:	<none>
   echo:
    Image:	gihyodocker/echo:patched
    Port:	8080/TCP
    Host Port:	0/TCP
    Environment:
      HOGE:	fuga
    Mounts:	<none>
  Volumes:	<none>
$ kubectl rollout history deployment echo --revision=2
deployment.extensions/echo with revision #2
Pod Template:
  Labels:	app=echo
	pod-template-hash=3665808783
  Annotations:	kubernetes.io/change-cause: kubectl apply --filename=simple/simple-deploy.yaml --record=true
  Containers:
   nginx:
    Image:	gihyodocker/nginx:latest
    Port:	80/TCP
    Host Port:	0/TCP
    Environment:
      BACKEND_HOST:	localhost:8080
    Mounts:	<none>
   echo:
    Image:	gihyodocker/echo:latest
    Port:	8080/TCP
    Host Port:	0/TCP
    Environment:
      HOGE:	fuga
    Mounts:	<none>
  Volumes:	<none>

# rollback
$ kubectl rollout undo deployment echo
# $ kubectl rollout undo deployment echo --to-revision=X
deployment.extensions/echo

$ kubectl rollout status deployment echo
deployment "echo" successfully rolled out

$ kubectl rollout history deployment echo
deployment.extensions/echo
REVISION  CHANGE-CAUSE
2         kubectl apply --filename=simple/simple-deploy.yaml --record=true
3         kubectl apply --filename=simple/simple-deploy.yaml --record=true
```

### Services

[services](https://kubernetes.io/docs/concepts/services-networking/service/)

> A Kubernetes Service is an abstraction which defines a logical set of Pods and a policy by which to access them - sometimes called a micro-service.

```sh
$ kubectl apply -f simple/simple-replicaset-with-label.yaml
replicaset.apps/echo-spring created
replicaset.apps/echo-summer created

$ kubectl get pod -l app=echo -l release=spring
NAME                READY   STATUS    RESTARTS   AGE
echo-spring-25xwb   2/2     Running   0          42s

$ kubectl get pod -l app=echo -l release=summer
NAME                READY   STATUS    RESTARTS   AGE
echo-summer-kncdz   2/2     Running   0          47s
echo-summer-wt4mn   2/2     Running   0          47s

$ kubectl apply -f simple/simple-services.yaml
service/echo created

$ kubectl get service echo
NAME   TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)   AGE
echo   ClusterIP   10.96.94.74   <none>        80/TCP    29s

# in containet access to http://echo/
```

### DNS

#### ClusterIP Service

ClusterIPはクラスタ上の内部IPをServiceに公開できる。
これによりPodからPod群へのアクセスをservice経由で行える

#### NodePort Service

クラスタ外からアクセスできるService。
各ノードからServiceポートへ接続のためのグローバルなポートを開ける

```bash
$ kubectl get svc echo                                                                                                            15:47:38
NAME   TYPE       CLUSTER-IP       EXTERNAL-IP   PORT(S)        AGE
echo   NodePort   10.102.240.154   <none>        80:32513/TCP   32s

$ curl http://127.0.0.1:32513
```

#### LoadBalancer Service

ローカルでは利用できない。
GCPであればCloud Load Balancing, AWSであればElastic Load Balancing が対応している。

#### ExternalName Service

selector、portもない特殊なService。
クラスタないから外部ホストを解決するためのエイリアスを提供する。

```yml
appVersion: v1
kind: Service
metadata:
  name: hogemoge
spec:
  type: ExternalName
  externalName: hogemoge.com
```

`hogemoge.com`を`hogemoge`で解決できるようになる。


### Ingress

> An API object that manages external access to the services in a cluster, typically HTTP.
> Ingress can provide load balancing, SSL termination and name-based virtual hosting.

ServiceのKubernetesクラスタの外への公開とVirtualhostやパススペースでの高度なHTTPルーティングを両立。

[kubernetes/ingress-nginx](https://github.com/kubernetes/ingress-nginx)

```bash
$ kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/mandatory.yaml
$ kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/provider/cloud-generic.yaml
$ kubectl get pods,services -n ingress-nginx
NAME                                            READY   STATUS    RESTARTS   AGE
pod/nginx-ingress-controller-777f447485-fgjcr   1/1     Running   0          11m

NAME                    TYPE           CLUSTER-IP     EXTERNAL-IP   PORT(S)                      AGE
service/ingress-nginx   LoadBalancer   10.106.2.218   localhost     80:32468/TCP,443:30701/TCP   9m
