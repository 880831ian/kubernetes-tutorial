# Ingress - Domain

## 什麼是 Ingress ?

還記得我們在 Service - NodePort 時，需要打 `<ip>:<port>` ，但現在網站除了網域以外，基本上不會需要自己去打 IP 以及 Port 了吧！那為了解決這個問題，有了 Ingreess。

Ingress 可以幫助我們統一對外的 port number，並根據 hostname 或是 pathname 來決定請求要轉發到哪一個 Service 上，之後就可以利用該 Service 連接到 Pod 來處理服務。

我們先來看一下一般的 Service ：

<br>

{{< image src="/images/K8s-advanced/describe-service.png"  width="800" caption="Service 圖片來源：[[Day 19] 在 Kubernetes 中實現負載平衡 - Ingress Controller](https://ithelp.ithome.com.tw/articles/10196261)" src_s="/images/K8s-advanced/describe-service.png" src_l="/images/K8s-advanced/describe-service.png" >}}

![圖片](https://raw.githubusercontent.com/880831ian/kubernetes-tutorial/master/images/describe-service.png)

可以看到當多個 Service 同時運行時，Node 都需要有對應的 port number 去對應每個 Server 的 port number。像是 GCP 這種雲端服務，每台機器都會配置屬於自己的防火牆。這也代表，不論新增、刪除 Service 物件，都必須要額外多調整防火牆的設定，Port 的管理也想對複雜。

<br>

若是使用 Ingress，我們只需要開放一個對外的 port numer，Ingree 可以在設定檔中設定不同的路徑，決定要將使用者的請求傳送到哪一個 Service 物件上：

{{< image src="/images/K8s-advanced/describe-ingress.png"  width="900" caption="Ingress 圖片來源：[[Day 19] 在 Kubernetes 中實現負載平衡 - Ingress Controller](https://ithelp.ithome.com.tw/articles/10196261)" src_s="/images/K8s-advanced/describe-ingress.png" src_l="/images/K8s-advanced/describe-ingress.png" >}}

![圖片](https://raw.githubusercontent.com/880831ian/kubernetes-tutorial/master/images/describe-ingress.png)

這樣的設計，除了讓維運人員不需要維護多個 port 或是頻繁的更改防火牆外，可以自訂條件的功能，也使得請求的導向可以更加彈性。

<br>
	
### Ingress 功能

* 將不同"路徑"的請求對應到不同的 Service 物件

若沒有設定網域，則該機器上的所有網域只要透過此路徑都可以連接到指定的 Service 物件。

<br>

* 將不同"網域"的請求對應到不同的 Service 物件

若沒有設定路徑，則會以 `/` 路徑連接到指定的 Service 物件。

<br>

* 支援 SSL Termination

SSL 全名是傳輸層安全性協定，而網站通常都會利用 https 進行加密以確保資料安全，但 Service 與 Pod 之間的溝通都是以無加密方式傳輸，所以 Ingress 就支援解密，讓 Service 與 Pod 可以正常溝通傳遞資料。

<br>

### minikube 啟動 Ingress

由於 minikube 預設沒有啟動 Ingress 功能，因此需要額外使用 `minikube addons enable ingress` 讓 minikube 啟動 Ingress (Ingress 也需要先安裝 Hyperfix)：

<br>

{{< image src="/images/K8s-advanced/ingress.png"  width="900" caption="Ingress 圖片來源：[[Day 19] 在 Kubernetes 中實現負載平衡 - Ingress Controller](https://ithelp.ithome.com.tw/articles/10196261)" src_s="/images/K8s-advanced/ingress.png" src_l="/images/K8s-advanced/ingress.png" >}}

![圖片](https://raw.githubusercontent.com/880831ian/kubernetes-tutorial/master/images/ingress.png)


<br>

### 設定 /etc/hosts

加入 Ingress 基本上就需要網域才可以使用，但我們在本機上做練習，所以只要修改本機的 host 檔案就可以了(加入 minikube ip 以及想要的網域名稱)。

```sh
$ vim /etc/hosts

192.168.64.11 test.tw
192.168.64.11 test-test.tw
```

<br>

因為有兩種方法，第一種 (設定網域以及路徑)跟第二種 (只設定路徑沒有設定網域)，所以底下也會分成兩種來說明！

### 設定網域以及路徑

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: kubernetes-demo-ingress
spec:
  rules:
    - host: "test.tw"
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: kubernetes-demo-service
                port:
                  number: 3000
```

Ingress 整體寫法與 Service 差不多，只差在 `spec` 的細部設定，我們就來說一下 spec 設定吧：

對了～ Ingress 的 apiVersion 不像是 Pod 跟 Service 一樣使用 `v1` ，因為目前支援 Ingress 的 API 只有 networking.k8s.io/v1，詳細可以參考 [kubernetes Ingress 官網](https://kubernetes.io/docs/concepts/services-networking/ingress/)

* rules

這代表這個 Ingress 的轉發規則，此 Ingress 所有的設定都必須寫載 `rules` 內。

* host

設定可以連接到 Service 物件的網路名稱。

* path

設定可以連接到 Service 物件的路徑名稱。

* pathType

分為 `Prefix` 和 `Exact` 兩種，Prefix：前綴符合就符合規則 ; Exact：需要完全一致才行，包含大小寫。

| type | path | request path | macth | 
| :---: | :---: | :---: | :---: |
| Prefix | / | 全部路徑 | Yes |
| Exact | /aa | /aa | Yes | 
| Exact | /bb | /cc | No | 
| Prefix | /aa | /aa,/aa/ | Yes |
| Prefix | /aa/cc | /aa/ccc | No |

* service.name

設定連接到的 Service 名稱，這裡要填寫的就是 Service 中在 `metadata` 內寫的 `name`。

* service.port.number

設定要經由哪個 port 連接到 Service 物件，就像是 Service 的 Port 要連接到 Pod 的 targetPort。

<br>

### 建立 Ingress 

一樣我們分成兩個做說明

#### 設定網域以及路徑

一樣使用 `apply` 來建立：

```
$ kubectl apply -f ingress.yaml 

ingress.networking.k8s.io/kubernetes-demo-ingress created
```

<br> 

使用 `kubectl get ing` 來查詢 Ingress 狀況：
```
$ kubectl  get ing

NAME                      CLASS   HOSTS     ADDRESS         PORTS   AGE
kubernetes-demo-ingress   nginx   test.tw   192.168.64.11   80      10m
```

接下來我們分別測試寫在 /etc/host 裡面的兩個網域，使用瀏覽器搜尋 `test.tw` 跟 `test-test.tw`：


<br>

* `test.tw`

![圖片](https://raw.githubusercontent.com/880831ian/kubernetes-tutorial/master/images/Shiba-Inu-3.png)

<br>

* `test-test.tw `

![圖片](https://raw.githubusercontent.com/880831ian/kubernetes-tutorial/master/images/Shiba-Inu-4.png)

<br>

從上面結果可以知道，因為我們有設定 domain，所以只有符合的才會連接到 Service 。

<br>