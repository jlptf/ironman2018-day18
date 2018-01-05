## Day 18 - Rolling

### 本日共賞

* Rolling
* Rolling Update
* Rolling Back

### 希望你知道
* [與 k8s 溝通: kubectl](https://ithelp.ithome.com.tw/articles/10193502)
* [yaml](https://ithelp.ithome.com.tw/articles/10193509)

<br/>

#### Rolling

滾動升級/回滾，在 k8s 中算是很方便的功能。你可以利用滾動升級正在運作的系統且不需要停機！另外，如果新的部署有發生問題的時候，透過回滾機制，你也可以簡單把系統回復。在 [Day 10 - 建構組件](https://ithelp.ithome.com.tw/articles/10193513) 我們曾經提到過 Deployment 這個物件裡面會包含了 ReplicaSet，然後再透過 ReplicaSet 來掌管 Pod 運行數。而 k8s 中回滾機制 (Rolling Update / Back)，就是透過 ReplicasSet 來達到。如下圖，每次的更新都會產生新的 ReplicaSet 來管控 Pod，你也可以把它想成產生一個新版本。而回滾則是把現有版本切換到上一個或指定的版本。

![](https://ithelp.ithome.com.tw/upload/images/20180105/20107062jukvnPxf7h.png)

#### Rolling Update

首先，我們先部署 [Day 10 - 建構組件](https://ithelp.ithome.com.tw/articles/10193513) 用到過的 `simple.yaml` 到 k8s 中

```bash
# kubectl apply -f simple.yaml
deployment "nginx" created

$ kubectl get replicaset
NAME              DESIRED   CURRENT   READY     AGE
nginx-969897d56   3         3         0         26s

$ kubectl get pods
NAME                    READY     STATUS    RESTARTS   AGE
nginx-969897d56-dkfdk   1/1       Running   0          1m
nginx-969897d56-mj7z9   1/1       Running   0          1m
nginx-969897d56-r4bx5   1/1       Running   0          1m

<=== 其實看字首就能分辨是由哪個 ReplicaSet 管控

$ kubectl describe po nginx-969897d56-mj7z9 | grep ReplicaSet
Created By:     ReplicaSet/nginx-969897d56
Controlled By:  ReplicaSet/nginx-969897d56
```

當部署個 Deployment 時，同時會產生一個 ReplicaSet (即 `nginx-969897d56`)，然後再透過這個 ReplicaSet 在產生三個 Pod (即 `Created By:     ReplicaSet/nginx-969897d56`)。

底下是滾動升級的相關設定

```
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxSurge: 1
    maxUnavailable: 1
minReadySeconds: 5
```

* `minReadySeconds`：有容器建立後需要一些時間啟動服務，這時候就可以設置這個參數。k8s 會等待設定的時間後再進行升級。如果沒有設定，則 k8s 會在容器一啟動後就直接進行升級服務。
* `maxSurge`：1 表示當一個新 Pod 被建立後才會刪除一個舊 Pod，2 表示在更新過程最多會多兩個 Pod 被建立以此類推。
* `maxUnavailable`：表示在升級過程中，最多可以有幾個 Pod 無法服務。請注意，當 `maxSurge` 不為 `0` 的時候，`maxUnavailable ` 也不能為 `0`

底下是完整的 yaml

```
# simple.rolling.yaml

---
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: nginx
spec:
  replicas: 5               <=== 這裡我們將數量從 3 增加為 5 個
  strategy:                 <=== 從這裡是 Rolling Update 的設定
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1           <=== 表示升級過程最多會有 6 個 Pod
      maxUnavailable: 1     <=== 最多允許一個 Pod 不能服務
  minReadySeconds: 5        <=== 容器啟動後 5 秒再開始進行升級
  template:
    metadata:
      labels: 
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.10.2    <=== 這裡指定更新後的容器
        ports:
        - containerPort: 80

```

如果要進行滾動升級 (Rolling Update)

```bash
$ kubectl apply -f simple.rolling.yaml --record
```

> `--record` 參數通知 k8s 把指令記錄起來以方便之後檢查
> 
> 你可以透過 `$ kubectl rollout history deployment [name]` 來查看，`[name]` 請記得換成 Deployment 的名稱。


當你下達指令後，你會發現有新的 ReplicaSet 被產生，然後 Pod 也會發生變化

```bash
$ kubectl get replicaset
NAME               DESIRED   CURRENT   READY     AGE
nginx-5bffddbbdb   2         2         0         25s   <=== 新建立的
nginx-969897d56    4         4         3         19m   <=== 原本的

$ kubectl get pods
NAME                     READY     STATUS              RESTARTS   AGE
nginx-5bffddbbdb-bbt9c   1/1       Running             0          21s
nginx-5bffddbbdb-fg6dg   1/1       Running             0          1m
nginx-5bffddbbdb-jwlvj   1/1       Running             0          1m
nginx-5bffddbbdb-pv6zj   1/1       Running             0          15s
nginx-5bffddbbdb-qt6v6   0/1       ContainerCreating   0          10s
nginx-969897d56-dkfdk    0/1       Terminating         0          20m
nginx-969897d56-mj7z9    0/1       Terminating         0          20m
nginx-969897d56-r4bx5    1/1       Running             0          20m
nginx-969897d56-xt6wr    0/1       Terminating         0          1m
```

從 Pod 的字首應該就能分辨出是哪個 ReplicaSet 建立的 Pod，上面就可以看到舊的 ReplicaSet `nginx-969897d56` 管控的 Pod 正在被刪除中，而新建立的 ReplicaSet `nginx-5bffddbbdb` 則是根據 `simple.rolling.yaml` 所指定的內容正在建立 Pod。


你可以透過指令觀察一下更新的紀錄

```bash
$ kubectl rollout history deployment nginx
deployments "nginx"
REVISION  CHANGE-CAUSE
1         <none>
2         kubectl apply --filename=simple.rolling.yaml --record=true
```

`CHANGE-CAUSE` 就會把你當時下達的命令記錄下來。不過要使用上面的指令，你必須要在每次下達更新的時候帶上 `--record` 的參數，不然你可能會看到

```bash
$ kubectl rollout history deployment nginx
deployments "nginx"
REVISION  CHANGE-CAUSE
1         <none>
2         <none>
```

這樣是不是比較不清楚發生什麼事情，只知道有 `1` 跟 `2` 兩個版本。

另外，你還可以透過指令更新映像檔

```bash
$ kubectl set image deployment [name] [container]=[image] --record

<=== 例如將 nginx 更新為 1.11.1 版
$ kubectl set image deployment nginx nginx=nginx:1.11.1 --record
deployment "nginx" image updated	


<=== 再次查看就會發現多了一筆紀錄
$ kubectl rollout history deployment nginx
deployments "nginx"
REVISION  CHANGE-CAUSE
1         <none>
2         kubectl apply --filename=simple.rolling.yaml --record=true
3         kubectl set image deployment nginx nginx=nginx:1.11.1 --record=true
```


#### Rolling Back

當更新後發現系統有問題時，可以利用回滾功能把系統回復。你可以

* 回到上一版本

```bash
$ kubectl rollout undo deployment [name]
```

* 回到指定版本

```
$ kubectl rollout undo deployment [name] --to-revision=[version]
```

底下舉個例子，剛剛我們有更新過一次，如果要回到版本 1，我們可以利用下面指令

```bash
$ kubectl rollout undo deployment nginx --to-revision=1
deployment "nginx" rolled back

$ kubectl get replicaset
NAME               DESIRED   CURRENT   READY     AGE
nginx-5bffddbbdb   1         1         1         12m
nginx-969897d56    5         5         4         31m

$ kubectl get pods
NAME                     READY     STATUS              RESTARTS   AGE
nginx-5bffddbbdb-bbt9c   0/1       Terminating         0          11m
nginx-5bffddbbdb-fg6dg   0/1       Terminating         0          12m
nginx-5bffddbbdb-jwlvj   1/1       Running             0          12m
nginx-5bffddbbdb-pv6zj   0/1       Terminating         0          11m
nginx-969897d56-4cn5g    0/1       ContainerCreating   0          14s
nginx-969897d56-7jtj4    1/1       Running             0          37s
nginx-969897d56-jr2kq    1/1       Running             0          26s
nginx-969897d56-sbn4m    1/1       Running             0          37s
nginx-969897d56-ttg28    1/1       Running             0          19s
```

下達更新回滾後，你就會發現 ReplicaSet 與 Pod 又會開始發生變化，原本的 `nginx-969897d56` 又會開始建立 Pod，而 `nginx-5bffddbbdb` 管控的 Pod 則會被刪除。

本文與部署檔案同步發表於 [https://jlptf.github.io/ironman2018-day18/](https://jlptf.github.io/ironman2018-day18/)

