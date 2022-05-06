# Ingress - Domain

## 什麼是 Deployment ?

Deployment 是一種負責管理 `ReplicaSet` 以及控制 Pod 更新的物件，在先前的文章都沒有提到 Pod 的更新，是因為 Pod 無法直接做更新，必須砍掉重建才會是新的內容，有了 Deployment 之後我們就可以很方便的進行 Pod 的更新了！

由於 `ReplicaSet` 本身也會控制 Pod，所以整個看起來會像是 Deployment 控制 Pod，但其實真正控制 Pod 的是 `ReplicaSet` ~

<br>

![圖片](https://raw.githubusercontent.com/880831ian/kubernetes-tutorial/master/images/deployment.png)

<br>

### Deployment 的特性

* 部署一個應用服務

上面我們提到 Deployment 可以幫助 Pod 進行更新，通常在開發一個產品的時候一定會不斷的更新，透過 Deployment 我們可以快速的更新 Pod 內部的 container，所以通常在部署應用的時候都會使用 Deployment 來進行部署。

<br>

* 在服務升級過程中可以達成無停機服務遷移 (Zero downtime deployment)

在 Deployment 幫 Pod 內部 container 進行更新的過程有一個專有名稱叫做 `RollingUpdate` ，`RollingUpdate` 翻成中文的意思是滾動更新，在更新的過程中 Deployment 會先產生一個新的 ReplicaSet 而這個 ReplicaSet 內部的 Pod 會運行新的內容，待新的 Pod 被 Kubernetes 確認可以正常運行後 Deployment 才會將舊的 ReplicaSet 進行取代的動作，這樣就完成了無停機服務遷移了。

<br>

* 可以 Rollback 到先前版本

每一次的 Deployment 在進行 RollingUpdate 的時候都會把當前的 ReplicaSet 做一個版本控制的紀錄，就像是 git commit 一樣，所以我們也可以利用這些紀錄來快速恢復成以前的版本，這些 Pod 也就會變成先前的內容。

<br>

講完基本的介紹後，接下來要介紹的是要如何撰寫以及建立一個 Deployment：

### Deployment 搭配 ReplicaSet 寫法

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kubernetes-deployment
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 1
  minReadySeconds: 60
  revisionHistoryLimit: 10
  template:
    metadata:
      labels:
        app: demo
    spec:
      containers:
        - name: kubernetes-demo-container
          image: 880831ian/kubernetes-demo
          ports:
            - containerPort: 3000
  selector:
    matchLabels:
      app: demo
```

<br>

看到 Deployment 是不是覺得跟 Replication Controller 非常相似呢？其實 Deployment 就多了再 RollingUpdate 時的設定以及 ReplicaSet 的設定而已，下面來說明一下這些設定：

首先一開始的 apiVersion 的值已經不是 `v1` 了，改成 `apps/v1`，由於 Kubernetes 針對 Deployment、RollingUpdate、ReplicaSet 等等設定了新的 `apiVersion` 值，通常都用 `apps/v1` 都是用來設定跟應用程式有關的架設，所以 Deployment 這邊要記得改成 `apps/v1` 歐！

<br>

在 `spec` 的地方有看到 `strategy` 新的設定值，這個主要用來設定 Deployment 更新的策略，這裡的 `strategy.type` 有兩種設定：

* RollingUpdate

此為預設值，先建立新的 ReplicaSet 並控制新內容的 Pod，待新 Pod 也可以正常運作後，才會通知 ReplicaSet 將原有的 Pod 給移除，由於過程中會有新舊兩種 Pod 同時上線，因此會有一段時間是新舊內容會隨機出現的情形發生。

這邊可以看到除了 `type` 以外還寫了 `maxSurage` 以及 `maxUnavailablle`，這兩個設定值為 RollingUpdate 過程的設定，接下來一樣說明一下兩個設定的功能：

`macSurge`：代表 ReplicaSet 可以產生比 Deployment 設定中的 `replica` 所設定的數量還多幾個出來，多新增 Pod 的好處是在 RollingUpdate 過程中可以減少舊內容顯示在頁面的機率。

`macUnavailable`：代表在 RollingUpdate 過程中，可以允許多少的 Pod 無法使用，假設 `macSurge` 設定非 0，`maxUnavailable` 也要設定非 0。

<br>

* Recreate

先通知當前 ReplicaSet 把舊的 Pod 砍掉再產生新的 ReplicaSet 並控制新內容的 Pod，由於先砍掉 Pod 才建立新的 Pod ，所以中間有一段時間伺服器會無法連線。 

也因為 Recreate 會砍掉重建，因此 Recreate 無法像 RollingUpdate 設定 `maxSurge` 以及 `macUnavailable`。

<br>

講完 Deployment 的更新流程設定後，接下來要講 Deployment 完成更新後的設定，這邊有兩種設定：

* minReadySeconds

`minReadySeconds` 代表當新的 Pod 建立好並且運行的 container 沒有 crash 的情況下，最少需要多少時間可以開始接受 Request，預設為 0 秒，代表當 Pod 一被建立起來，就可以馬上開始接受 Request，假設今天 container 在剛運行的時後需要花時間做初始化，這時候就可以利用 `minReadySeconds` 讓此 Pod 不會馬上接受到 Request ，這個是選填的設定。

<br>

* revisionHistoryLimit

每次 Deployment 在進行更新的時候，都會產生一個新的 ReplicaSet 用來進行版本控制，在 Deployment 中這個專有名稱稱為 `revision`，所以這個設定就是要設定最多只會有多少個 `revision`，這個也是選填的設定。

<br>

### Deployment 建立

老樣子，使用 `apply` 來建立 Deployment，我們可使用 `kubectl get deploy` 來查看是否建立成功：

```sh
$ kubectl get deploy

NAME                    READY   UP-TO-DATE   AVAILABLE   AGE
kubernetes-deployment   3/3     3            3           8m31s
```

<br>

由於 Deployment 直接管理 ReplcaSet，因此我們可以查看 ReplcaSet 是否也有被建立起來：

```sh
$ kubectl get rs

NAME                              DESIRED   CURRENT   READY   AGE
kubernetes-deployment-dc5c59fdb   3         3         3       10m
```
可以看到 ReplcaSet 後面會自動加上一小段亂數，這邊是 Deployment 在建立 ReplicaSet 的時候加進去的，這樣之後可以更方便的利用 ReplcaSet 進行 rollback 的動作。

<br>

由於 ReplicaSet 會直接管理 Pod，因此我們也可以查看 Pod 是否有被建立起來：

```sh
$ kubectl get po

NAME                                    READY   STATUS    RESTARTS   AGE
kubernetes-deployment-dc5c59fdb-cjbz6   1/1     Running   0          12m
kubernetes-deployment-dc5c59fdb-mjvd7   1/1     Running   0          12m
kubernetes-deployment-dc5c59fdb-r92zt  1/1     Running   0          11m
```

<br>

那我們用 `minikube dashboard` 來查看一下：

<br>

![圖片](https://raw.githubusercontent.com/880831ian/kubernetes-tutorial/master/images/DeploymentReplicaSet.png)

<br>

Deployment 的最後要來說的是要如何更新底下的 Pod 呢，大家就接著往下看囉！

### 如何更新 Deployment 內部的 Pod

 大家都知道 Pod 是 Kubernetes 最小的運行單位，所以更新 Pod 的意思就是把內部運行的 container 進行更新，也就是說我們只要更新 Pod 的 image 就可以順利的讓 Pod 運行最新的內容，Deployment 就是運用這個原理才進行 Pod 的內容更新，方法也很簡單只要利用 `Set` 這個參數就可以了，可以使用 `kubectl set -h` 可以看 `set` 這個參數真正的用法。

<br>

![圖片](https://raw.githubusercontent.com/880831ian/kubernetes-tutorial/master/images/set.png)

<br>

可以看到 `set` 後面還要接 `SUBCOMMAND`，而 `SUBCOMMAND` 就是 `Available Commands` 的內容，由於我們要更新的是 image 所以這邊的 `SUBCOMMAND` 會是 `image`，完整指令是：

```sh
$ kubectl set image deployment/kubernetes-deployment kubernetes-demo-container=880831ian/kubernetes-demo:v1 --record

deployment.apps/kubernetes-deployment image updated
```
           
<br>
        
使用 set 來更新 Pod，格式是 `kubectl set image deployment/<deployment name> <pod name>=<要更新的 image>`，後面加上 `--record` 參數，這樣會紀錄每次更新的時候到底更新哪些內容，這樣日後要進行 rollback 也會比較容易知道要 rollback 回哪個 revision，由於要顯示差異，所以在 [dockerhub](https://hub.docker.com/repository/docker/880831ian/kubernetes-demo) 上又多推一個版本 `v1`，將原本柴犬的圖片改成 kubernetes logo，等於我們更新是從 `kubernetes-demo:latest` 更新至 `kubernetes-demo:v1`，我們來看看它的變化吧！

<br>

這是還沒更新 Pod 前的 Deployment 與 ReplicaSet：

![圖片](https://raw.githubusercontent.com/880831ian/kubernetes-tutorial/master/images/DeploymentReplicaSet.png)

<br>

我們用 set 下完更新指令後，可以查看 ReplicaSet 以及 Pod 在更新過程中的變化：

![圖片](https://raw.githubusercontent.com/880831ian/kubernetes-tutorial/master/images/set-1.png)

<br>

可以發現 strategy 為 RollingUpdate 的時候並不會把舊有的 Pod 移除反而會讓新舊 Pod 同時上線，以達到無停機服務的作用，但這樣在網頁中就有可能會同時出現新舊內容。

<br>

![圖片](https://raw.githubusercontent.com/880831ian/kubernetes-tutorial/master/images/set-2.png)

<br>

最後等到新的 Pod 已經建立完成，且正常運作，ReplicaSet 就會把舊的 Pod 給移掉：

<br>

![圖片](https://raw.githubusercontent.com/880831ian/kubernetes-tutorial/master/images/set-3.png)

<br>

再次瀏覽就可以發現已經變成新的內容了！代表我們完成更新動作～

<br>

![圖片](https://raw.githubusercontent.com/880831ian/kubernetes-tutorial/master/images/set-4.png)

<br>

### Deployment 回朔版本

我們講完更新後，接著要講如何 rollback 回以前的版本，首先我們必須使用 `rollout` 這個參數，一樣使用 `kubectl rollout -h` 可以查看 rollout 的用法，跟 `set` 十分相似。
 
 <br>

![圖片](https://raw.githubusercontent.com/880831ian/kubernetes-tutorial/master/images/rollout.png)

<br>

由於我們這邊要 rollback ，所以 `SUBCOMMAND` 會使用 `undo`，我們上面有用 `--record` 可以查看要 rollback 的版本，所以我們這邊寫法會長像這樣：

```sh
$ kubectl rollout undo deployment/kubernetes-deployment --to-revision=1

deployment.apps/kubernetes-deployment rolled back
```
格式是 `kubectl rollout undo deployment/<deployment name> --to-revision=<--record 版本>`
 
<br>

就可以看到又回復成以前的柴犬囉 ><

<br>
 
![圖片](https://raw.githubusercontent.com/880831ian/kubernetes-tutorial/master/images/Shiba-Inu-3.png)
 
<br>  
 
### 為甚麼不用 Replication Controller
 
最後要討論的是為什麼要不用  Replication Controller 而是改用 ReplicaSet + Deployment？

由於實際再使用 Kubernetes 時架構會比現在練習的還要複雜，所以用 ReplicaSet 讓 selector 用更彈性的方式選取 Pod 會是比較好的做法。
 