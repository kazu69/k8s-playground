# Kubernetes

## Deployment strategy

### Rollingupdate

```sh
$ kubectl create -f echo-version.yaml
service/echo-version created
deployment.apps/echo-version created

# container for check version
$ kubectl create -f update-checker.yaml
pod/update-checker created

# login version
$ kubectl logs f update-checker

# update template version
$ kubectl patch deployment echo-version -p '{"spec": {"template":{"spec": {"containers":[{"name":"echo-version","image":"gihyodocker/echo-version:0.2.0"}]}}}}'
deployment.extensions/echo-version patched
# log change
# [Sun Feb 24 14:29:53 UTC 2019] APP_VERSION=0.1.0
# [Sun Feb 24 14:29:54 UTC 2019] APP_VERSION=0.2.0

# change  pods logs
$ kubectl get pods -l app=echo-version -w 
# NAME                           READY   STATUS    RESTARTS   AGE
# echo-version-88f59b564-wpm92   1/1     Running   0          23s   <---- current
# echo-version-675965c4b9-njt9l   0/1   Pending   0     0s
# echo-version-675965c4b9-njt9l   0/1   Pending   0     0s
# echo-version-675965c4b9-njt9l   0/1   ContainerCreating   0     0s  <---- deployed
# echo-version-675965c4b9-njt9l   1/1   Running   0     3s
# echo-version-88f59b564-wpm92   1/1   Terminating   0     33s  <---- switch pod
# echo-version-88f59b564-wpm92   0/1   Terminating   0     34s
# echo-version-88f59b564-wpm92   0/1   Terminating   0     35s
# echo-version-88f59b564-wpm92   0/1   Terminating   0     35s
```

### Rollingupdateの制御

`strategy`を設定することで制御できる。
`maxUnavailable` Rollingupdate時に同時に削除できるPodの最大数。defaultはreplicasの25%。。
`maxSurge`は新しいPodを作る個数。defaultはreplicasの25%。

ポイントは最初のうちは `maxUnavailable=1`で1つづつ入れ替えていくことでサービスインしているPodの数が減ることない。

### Healthcheck

`livenessProbe` で死活監視の設定。`readinessProbe`はコンテナ外からhttpを受けられる状態になっているかのチェックを設定。
HTTPでL7, TCPでL４レベルの監視ができる。

### BlueGreen Deployment

Rollingupdateではversion違いのPodが混在する期間ができてしまう。これにより意図しない副作用を起こす可能性がある。
これを解決するために`BlueGreen Deployment`という方法を使うことがある。
すでにデプロイされているサーバー群とは別に新たにサーバー群をデプロイし、LBやサービスディスカバリで参照先を切り替える方法。
2つのサーバー群を抱えるためリソースは多くなる。

```sh
$ kubectl apply -f echo-version-blue.yaml
deployment.apps/echo-version-blue created
$ kubectl apply -f echo-version-green.yaml
deployment.apps/echo-version-green created
$ kubectl apply -f update-checker.yaml
pod/update-checker created
$ kubectl apply -f echo-version-bluegreen-service.yaml
service/echo-version created

$ kubectl logs -f update-checker
# [Sun Feb 24 15:03:56 UTC 2019] APP_VERSION=0.1.0
# [Sun Feb 24 15:03:57 UTC 2019] APP_VERSION=0.2.0

$ kubectl patch service echo-version -p '{"spec": {"selector": {"color": "green"}}}'
```

デプロイに失敗した場合、上記の例だと `select.color`を元の値に辺区したら良い。

```sh
$ kubectl patch service echo-version -p '{"spec": {"selector": {"color": "blue"}}}'
```
