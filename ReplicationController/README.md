# Replication Controller


前面有提到 Pod 是 Stateless，所以我們可以擴充 Pod，今天這邊文章要開始介紹如何擴充 Pod ，我們從最簡單的 [Replication Controller](https://kubernetes.io/docs/concepts/workloads/controllers/replicationcontroller/) 開始說起！

## 什麼是 Replication Controller ?

大家看到 Controller 就知道 Replication Controller 也是一種 Controller 負責控制 Replication，而 Replication 翻成中文是複製的意思，在 Kubernetes 中 Replication 代表同一種 Pod 的複製品。

這邊要帶給大家認識一個重要的設定：`replica`，`replica` 就是複製品的意思，透過這個設定我們就可以快速產生一樣內容的 Pod，舉例來說：今天設定了 `replica: 3` 就代表會產生兩個內容一樣的 Pod 出來。

<br>

![圖片](https://raw.githubusercontent.com/880831ian/kubernetes-tutorial/master/images/ReplicationController.png)

<br>

### Replication Controller 用途

上面有提到 Replication Controller 可以利用設定 `replica` 的方式快速建立 Pod 數量，除了建立之外 Replication Controller 也確保 Pod 數量與我們設定的 `replica` 一致，假如今天不小心刪除其中一個 Pod，這時候 Replication Controller 會自動再產生一個新的 Pod 來補齊刪除的 Pod 空缺，所以我們可以善用 Replication Controller 來讓系統更佳穩定。

<br>

### Replication Controller 寫法

我們來修改一下之前的 kubernetes-demo.yaml Pod 檔案：([Github 程式碼連結](https://github.com/880831ian/kubernetes-tutorial/tree/master/ReplicationController))

```yaml
apiVersion: v1
kind: ReplicationController
metadata:
  name: kubernetes-demo
spec:
  replicas: 3
  selector:
    app: demo
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
```

這個設定檔看似複雜但其實很簡單，可以發現 `template` 區塊內的設定基本上就是 Pod 的設定，再加上一些屬於 Replication Controller 的設定。

由於我們要建立的是 Replication Controller，因此在一開始的 `spec` 要填的是 Replication Controller 的設定，所以 `replica` 會擺在第一個 `spec` 內。

可以再看到 `selector`，前面提到 Replication Controller 要控制的就是 Pod 的數量，所以這邊的 `selector` 就是要選取 Pod，就跟我們在 Service 要選取 Pod 的一樣。

最後一個新的設定：`template`，`template` 就是用來定義 Pod 的資訊，所以 Pod 的內容像是 `metadata`、`spec` 等等都會寫在 `template` 內，所以可以把 `template` 想像成不需要寫 `apiVersion` 跟  `kind` 的 Pod 壓模檔，有了這個觀念再來看 `template` 內的描述就很簡單，只是把 Pod 的內容複製過來而已，而 `template` 內的 `spec` 就是寫上 Pod 的 container 資訊。

<br>

### Replication Controller 建立

老樣子，也是使用 `apply` 來建立壓模檔：

```sh
$ kubectl apply -f kubernetes-demo.yaml

replicationcontroller/kubernetes-demo created
```	

<br>

來查看一下 Replication Controller 是否有成功建立起來，可以使用 `kubectl get rc` 來查詢：

```sh
$ kubectl get rc

NAME              DESIRED   CURRENT   READY   AGE
kubernetes-demo   3         3         3       28s
```

<br>

接下來可以查看 Pod 是否有出現 3 個，所以使用 `kubectl get po` 來查詢：

```sh
$ kubectl get po

NAME                    READY   STATUS    RESTARTS   AGE
kubernetes-demo-4zkxm   1/1     Running   0          37s
kubernetes-demo-cp8jt   1/1     Running   0          37s
kubernetes-demo-pt9px   1/1     Running   0          37s
```
可以看到為了不要讓名稱重複，所以 Replication Controller 會在每一個 Pod 名稱後面加入亂數。

<br>

接下來我們用 `minikube dashboard` 來測試一下，是否刪除其中一個 Pod 後，Replication Controller 會自動建立新的：


<br>

![圖片](https://raw.githubusercontent.com/880831ian/kubernetes-tutorial/master/images/rc-1.png)

<br>

當我們隨機刪除一個 Pod 時，被刪除的 Pod 會 Terminating 準備刪除，且啟動一個新的 Pod ContainerCreating：

![圖片](https://raw.githubusercontent.com/880831ian/kubernetes-tutorial/master/images/rc-2.png)

<br>

當新的 Pod 啟動成功後，舊的 Pod 才會被刪除，所以可以確保我們的服務穩定度。

![圖片](https://raw.githubusercontent.com/880831ian/kubernetes-tutorial/master/images/rc-3.png)

<br>

綜上所述，可以知道 Replication Controller 真的會控制 Pod 數量，那我們刪掉一個 Pod 他就重生一個，這樣不會永遠都刪不完嗎？其實我們可以把 Replication Controller 砍掉就好了，而 Replication Controller 刪除時，也會自動終止底下的 Pod ，最後 Pod 都會自動刪除。

但其實 Kubernetes 官方不建議使用 Replication Controller 的方式來控制 Pod，而是建議使用 Deployment 搭配 ReplicaSet 來控制，我們接下來要介紹的主題就是：`Deployment` 跟 `ReplicaSet` 。