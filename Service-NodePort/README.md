## port-forward

`port-forward` 簡單來說就是把本機的某一個 Port 與 Pod 所開放對外的 Port 做映射，就像是我們在 Docker 跑 container 時會使用 -p 來連結機器與 container 的 port 一樣～

使用的方法也很簡單且方便，使用 `kubectl port-forward <pod> <external-port>:<pod-port>`，我們拿 [Kubernetes 基本篇](https://pin-yi.me/k8s)最後的範例，來做說明：

```sh
$ kubectl port-forward kubernetes-demo-pod 3000:3000

Forwarding from 127.0.0.1:3000 -> 3000
Forwarding from [::1]:3000 -> 3000
Handling connection for 3000
```

我們把 kubernetes-demo-pod 這個 pod，用 port-forward 設定本機 3000 port 與 pod  3000 port 做映射，當我們瀏覽 `http://localhost:3000` 就可以看到 pod 裡面的內容了！

雖然很方便，可以馬上就開好要映射的 port，但缺點就是每次建立 pod 時都需要手動去打指令來設定 port，且時間久了，也會忘記本機上哪些 port 有被使用到，因此這邊推薦使用 `Service` 來取代 `port-forward`，那我們來看看 `Service` 是什麼吧。

<br>

## 什麼是 Service ?

`service` 他其實就是建立的一個網路連線通道，可以讓應用程式正確的連結到正在運行的 pods，而 `service` 又有4種的表現形式，我們接下來會一個一個簡單介紹：

### ClusterIP

它是 service 的預設值，所以沒有設定時，預設就是使用該方式做連線，它代表這個 service 只能在相同的 cluster 內使用，無法讓外部做使用。

<br>

### NodePort

簡單來說它可以從外部連線到內部使用。假設本機有其他服務，例如：nginx 之類的服務，還有架一個 K8s 的 cluster ，這時候只要設定好 NodePort，就可以讓本機使用 K8s cluster 來使用內部的服務。

<br>

### ExternalName

主要是為了讓不同 namespace ，以 ClusterIP 所生成的 service 可以利用 ExternalName 設定外部名稱，藉以連到指定的 namespace Service。

<br>

### LoadBalancer

這個屬性是強化版的 NodePort，除了 NodePort 可以讓外部連線的優點以外，同時也建立負載平衡的機制來分散流量，很可惜 LoadBalaner 只提供雲端服務，例如：GCP、AWS 等等都有支援，目前 minikube 要使用  LoadBalancer 需要先啟動 `tunnel` 才能做使用。`tunnel` 是什麼呢？我們後面會說明！

<br>

![圖片](https://raw.githubusercontent.com/880831ian/kubernetes-tutorial/master/images/service.png)

<br>

### NodePort 實作

那我們來用 Service (NodePort) 改寫[基本篇](https://pin-yi.me/k8s)的連線問題 ：([Github 程式碼連結](https://github.com/880831ian/kubernetes-tutorial/tree/master/Service-NodePort))

* service.yaml
```yaml
apiVersion: v1
kind: Service
metadata:
  name: kubernetes-demo-service
  labels:
    app: demo
spec:
  type: NodePort
  ports:
    - protocol: TCP
      port: 3000
      targetPort: 3000
      nodePort: 30001
  selector:
    app: demo
```
結構與 kubernetes-demo.yaml 相同，以下簡單說明不同之處：

kind

該元件的屬性，此設定檔的類型是：Service

spec

* type：指定此 Service 要使用的方法，這邊我們使用 NodePort。
* ports.protocol：此為連線的網路協議，預設值為 `TCP`，當然也可以使用 `UDP`。
* ports.port：此為建立好的 Service 要以哪個 Port 連接到 Pod 上。
* ports.targetPort：此為目標 Pod 的 Port ，通常 port 跟 targetPort 一樣。
*  ports.nodePort：此為機器上的 Port 要對應到該 Service 上，這個設定要 nodePort 形式的 Service 才會有效果，假設今天沒有設定 nodePort ，Kubernetes 會自動開一個機器上的 Port 來對應該 Service ，範圍是在 30000 - 37267之間。
* selector.app：如果要使 Service 連接到正確的 Pod 就必須利用 `selector`，只要原封不動的把 Pod 的 `Labels` 複製上去即可。

<br>

接下來使用 `kubectl apply` 建立 service：

```sh
$ kubectl apply -f service.yaml
```

<br>

![圖片](https://raw.githubusercontent.com/880831ian/kubernetes-tutorial/master/images/service-4.png)

<br>

接下來取得 minikube Node IP，可以使用：


```sh
$ minikube ip

192.168.64.11
```

<br>

打開瀏覽器搜尋 `192.168.64.11:30001` 就可以看到我們可愛的柴犬囉 ><

<br>

![圖片](https://raw.githubusercontent.com/880831ian/kubernetes-tutorial/master/images/Shiba-Inu-1.png)

<br>
