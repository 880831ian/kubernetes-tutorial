# StatefulSet

## 什麼是 Stateless 和 Stateful

### Stateless

Stateless 顧名思義就是無狀態，我們可以想成我們每次與伺服器要資料的過程中都不會被伺服器記錄狀態，**每一次的 Request 都是獨立的**，彼此是沒有關聯性的，也就是我們當下獲得的資料只能當下使用沒有辦法保存，靜態網頁通常都是一種 Stateless 的應用。

舉個例子來說：今天我想要查詢火車時刻表，我可以藉由 Google 搜尋火車時刻表，並點選連結，或是直接在瀏覽器輸入 `https://tip.railway.gov.tw/` ，這兩種結果最後都會一致，並不會因為我的操作不同而產生不同結果，這就是一種 Stateless 的表現。

<br>

### Stateful

Stateful 就是 Stateless 的相反，也就是每次的 Request 都會被記錄下來，日後都可以進行存取，Stateful 最常見的例子是資料庫，所以我們可以理解成 Stateful 背後一定會有一個負責更新內容的儲存空間。

幾個例子來說：今天我們想要查看 Google 雲端硬碟的內容，我們必須先登入自己的帳號才可以查看內容，這種有操作先後順序才會有結果的就是一種 Stateful 的表現。

<br>

![圖片](https://raw.githubusercontent.com/880831ian/kubernetes-tutorial/master/images/stateless_ful.png)

<br>

那為什麼要提到 Stateless 跟 Stateful 呢？

因為跟 Pod 有很大的關係，在 Kubernetes 中 Pod 就屬於 Stateless 的，我們前面有提到 Stateless 的特性就是每次的 Request 都是獨立的，這樣有一個好處是可以快速的擴充。

在 [Kubernetes - 基本篇中的 Pod](https://pin-yi.me/k8s/#pod) 有提到：Pod 是 Kubernetes 中最小的單位，由於 Pod 是屬於 Stateless 的，即便今天同一種內容的 Pod 有很多個也沒有關係，因為每次的 Request 都是獨立的，多個 Pod 就多個連線的端點而已。

<br>

### Kubernetes 的 Stateful

上面有說到 Kubernetes 的 Pod 是 Stateless 的，那難道 Kubernetes 沒有辦法做 Stateful 應用嗎？其實是可以的，Kubernetes 為了 Stateful 有特別開啟一個類別叫：`StatefulSet`

<br>

這邊會簡單說明一下 `StatefulSet` 的架構：

### StatefulSet

StatefulSet 一共有兩個重要的部分：

* Persistent Volume Claim

前面有說到 Stateful 背後有一個更新內容的儲存空間，在 Kubernetes 中負責管理儲存的空間是 `Volume`，作用與 Docker 的 Volume 幾乎一模一樣，但 Kuberntes 的 Volume 只是在 Pod 中暫時存放的儲存空間，當 Pod 移除之後這個儲存空間就會消失，為了要在 Kubernetes 中建立一個像是資料庫可以永久儲存的空間，這個 Volume 不能被包含在 Pod 中，而這個就是 `Persistent Volume (PV)`。

Persistent Volume Claim (PVC) 就是負責連接 Persistent Volume (PV) 的物件，所以可以想像一下今天有多少的 Persistent Volume 就會有多少的 Persistent Volume Claim。

<br>

* Headless Service

還記得在 [Service](#什麼是-service-) 有提到 ClusterIP 嗎？其實每個 Service 都會有自己一組的 ClusterIP (ExternalName 形式的除外)，所以 Headless 的意思其實就是不要有 ClusterIP，方法也很簡單，直接在設定檔中加入 `ClusterIP: None` 就可以了！

這麼做有什麼好處？由於 Headless Service 並沒有直接跟 Pod 有對應關係，因此 Service 本身沒有 ClusterIP，所以 Kubernetes 內部在溝通時就沒有辦法把我們設定好的 Service 名稱進行 IP 轉換，不過 Headless Service 會將內部的 Pod 的都建立屬於自己的 domain，所以我們可以自由的選擇要連接到哪一個 Pod。

這時候你會說可以用手動來連接呀？但因為 Service 一般是跟著 Pod 的 Label ，所以一個 Service 都會連接許多個 Pod，這樣我們就沒有辦法針對某個 Pod 來做事情，所以 Headless Service 在 Stateful 中也會被建立。

<br>

我們一直強調說 Kubernetes 最小的單位是 Pod，即便是 StatefulSet 也會有 Pod，只是這個 Pod 會歸 StatefulSet 管理，綜合上面所述可以知道一個 StatefulSet 裡面除了執行的 Pod 外還會有負責跟 Persistent Volume 連接的 Persistent Volume Claim，整體的 StatefulSet 架構會長得像這樣：

<br>

{{< image src="/images/K8s-advanced/StatefulSet.png"  width="300" caption="Kubernetes StatefulSet 架構" src_s="/images/K8s-advanced/StatefulSet.png" src_l="/images/K8s-advanced/StatefulSet.png" >}}

<br>

基本上 StatefulSet 中在 Pod 的管理上都是與 Deployment 相同，基於相同的 container spec 來進行; 而其中的差別在於 **StatefulSet controller 會為每一個 Pod 產生一個固定識別資訊，不會因為 Pod 改變後而有所變動**。


<br>

### 什麼時候需要使用 StatefulSet ?

如何研判哪些 Application 需要使用 StatefulSet 來部署？只要符合以下條件，就需要使用 StatefulSet 來進行部署

* 需要穩定 & 唯一的網路識別 (pod 改變後的 pod name & hostname 都不會變動)
* 需要穩定的 Persistent storage (pod 改變後還是能存取相同的資料，基本上用 PVC 就可以解決)
* 部署 & 擴展的時候，每個 Pod 的產生都是有其順序且逐一慢慢完成的
* 進行更新操作時，也是與上面的需求相同

<br>

### StatefulSet 有什麼限制？

* v1.5 以前版本不支援，v1.5 ~ v1.9 之間是 beta，v1.9 後正式支援
* storage 的部分一定要綁定 PVC，並綁定到特定的 StorageClass or 預先配置好的 Persistent Volume，確保 Pod 被刪除後資料依然存在。
* 需要額外定義一個 Headless Service 與 StatefulSet 搭配，確保 Pod 有固定的 network identity。

{{< admonition tip "network identity">}}
代表可以直接透過 domain name 直接取的 Pod IP; 實現的方法則是部署一個 ClusterIP=None 的 Service，讓 Cluster 內部存取 Service 時，可以直接連到 Pod 而不是 Service IP。
{{< /admonition >}}

<br>

### StatefulSet 寫法

要寫一個 StatefulSet ，有幾個重要的部分必須涵蓋：

1. Application & Persistent Volume Claim
2. Headless Service
3. `.spec.selector` 所定義的內容 (matchLabels) 必須與 `.spec.template.metadata.labels` 相同。

其他部分都與 `Deployment` 幾乎相同 ~

<br>

我們先來看看要怎麼定義 Application & Persistent Volume Claim (把 Service 跟 StatefulSet 寫在一起) ([Github 程式碼連結](https://github.com/880831ian/kubernetes-tutorial/tree/master/StatefulSet))

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx
  labels:
    app: nginx
spec:
  ports:
    - port: 80
      name: web
  clusterIP: None
  selector:
    app: nginx
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web
spec:
  serviceName: "nginx"
  replicas: 2
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
          image: k8s.gcr.io/nginx-slim:0.8
          ports:
            - containerPort: 80
              name: web
          volumeMounts:
            - name: www
              mountPath: /usr/share/nginx/html
  volumeClaimTemplates:
    - metadata:
        name: www
      spec:
        accessModes: ["ReadWriteOnce"]
        resources:
          requests:
            storage: 1Gi
```

<br>

我們先打開一個 Terminal 來觀察 StatefulSet 創建 Pod 的過程：

```sh
$ kubectl get pods -w -l app=nginx
```

<br>

我們一樣使用 `kubectl apply` 來創建定義在 `web.yaml` 中的 Headless Service 和 StatefulSet。

```sh
$ kubectl apply -f web.yaml
```

<br>

我們看一下剛剛的 StatefulSet 創建 Pod 的過程，可以發現我們擁有 N 的副本的 StatefulSet ， Pod 部署時會按照 {0 .... N-1} 的序號依序創建。

```sh
NAME    READY   STATUS    RESTARTS   AGE
web-0   0/1     Pending   0          0s
web-0   0/1     Pending   0          0s
web-0   0/1     ContainerCreating   0          0s
web-0   1/1     Running             0          2s
web-1   0/1     Pending             0          0s
web-1   0/1     Pending             0          0s
web-1   0/1     ContainerCreating   0          0s
web-1   1/1     Running             0          3s
```
可以發現 web-1 Pod 是在 web-0 Pod 處在 Running 狀態才會被啟動。此外，可以發現就跟我們上面講的一樣，StatefulSet 中的 Pod 擁有一個獨一無二的身份標記，基於 StatefulSet 控制器分配給每個 Pod 的唯一順序索引。

Pod 名稱的格式是 < statefulset name >-< ordinal index >，像我們 `web` 這個 StatefulSet 有兩個副本，所以它創建了兩個 Pod：`web-0`、`web-1`。

<br>

### StatefulSet 測試

我們知道 StatefulSet 它有使用穩定的網路身份以及 PV 的永久儲存，那我們就分別來測試看看：

#### StatefulSet 穩定的網路身份

我們先使用 `kubectl exec` 在每個 Pod 中執行 `hostname`

```
$ for i in 0 1; do kubectl exec "web-$i" -- sh -c 'hostname'; done

web-0
web-1
```

<br>

再使用 `kubectl run` 來運行一個提供 `nslookup` 命令的容器，通過對 Pod 的主機名執行 `nslookup`，我們可以檢查他在集群內部的 DNS 位置。

```sh
$ kubectl run -i --tty --image busybox:1.28 dns-test --restart=Never --rm /bin/sh
```

<br>

啟動一個新的 shell ，並運行 `nslookup web-0.nginx ` 跟 `nslookup web-1.nginx`：

```sh
/ # nslookup web-0.nginx
Server:    10.96.0.10
Address 1: 10.96.0.10 kube-dns.kube-system.svc.cluster.local

Name:      web-0.nginx
Address 1: 172.17.0.5 web-0.nginx.default.svc.cluster.local

/ # nslookup web-1.nginx
Server:    10.96.0.10
Address 1: 10.96.0.10 kube-dns.kube-system.svc.cluster.local

Name:      web-1.nginx
Address 1: 172.17.0.6 web-1.nginx.default.svc.cluster.local
```
可以看到 Headless Service 的 CHANCE 指向 SRV 記錄 (記錄每個 Running 的 Pod)。SRV 紀錄指向一個包含 Pod IP 位址的記錄表。

<br>

我們使用 `kubectl delete pod -l app=nginx` 刪除 Pod 後，會發現 Pod 的序號、主機名、SRV 條目和記錄名稱都沒有改變！

<br>

#### StatefulSet 永久儲存

我們先查看 `web-0` 跟 `web-1` 的 PersistentVolumeClaims：

```sh
$ kubectl get pvc -l app=nginx

NAME        STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
www-web-0   Bound    pvc-e9a23104-d018-4d73-8cd9-89e5ea67f96c   1Gi        RWO            standard       39m
www-web-1   Bound    pvc-00dd6c87-6d95-4d04-9c1b-49b43441f4a1   1Gi        RWO            standard       38m
```

StatefulSet 控制器創建兩個 PersistentVolumeClaims，綁定兩個 PersistentVolumes，因為我們配置是動態提供PersistentVolume，所有的PersistentVolume 都是自動創建和綁定的。

Nginx web 服務器默認會加載位於 `/usr/share/nginx/html/index.html` 的 index 文件。因此我們在 `spec` 中的 `volumeMounts` 將 `/usr/share/nginx/html` 資料夾由一個 PersistentVolume 支持。

<br>

那我們將 Pod 主機名稱寫入 `index.html` ，再刪掉 Pod 看看寫入內容是否還會存在：

```sh
$ for i in 0 1; do kubectl exec "web-$i" -- sh -c 'echo "$(hostname)" > /usr/share/nginx/html/index.html'; done
$ for i in 0 1; do kubectl exec -i -t "web-$i" -- curl http://localhost/; done

web-0
web-1
```

<br>

當我們刪除 Pod 後，如果沒有使用 PersistentVolumeClaims 去綁定 PersistentVolumes 的話，資料就會消失，那我們來看看有綁定的結果：

使用 `kubectl delete pod -l app=nginx` 刪除 Pod：

```sh
pod "web-0" deleted
pod "web-1" deleted
```

<br>

再次使用 `kubectl get pod -w -l app=nginx` 來檢查 Pod 的狀態：

```sh
web-0   0/1     ContainerCreating   0          0s
web-0   1/1     Running             0          2s
web-1   0/1     Pending             0          0s
web-1   0/1     Pending             0          0s
web-1   0/1     ContainerCreating   0          0s
web-1   1/1     Running             0          1s
```

<br>

一樣使用 `for i in 0 1; do kubectl exec -i -t "web-$i" -- curl http://localhost/; done` 來查看：

```sh
web-0
web-1
```
可以發現 `web-0`、`web-1` 雖然重新啟動，但依舊會監聽它們主機名，因為和它們的 PersistentVolumeClaim 相關聯的 PersistentVolume 被重新掛載到了各自的 `volumeMount` 上。
