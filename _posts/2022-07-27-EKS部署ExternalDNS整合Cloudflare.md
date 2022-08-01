---
title: 004_EKS 部署 ExternalDNS 整合 Cloudflare
date: '2022-07-27 06:24:00 +0800'
categories: [DevOps]
tags: [cloudflare,kubernetes]     # TAG names should always be lowercase
author: owen
mermaid: true
math: true
---

# 起源
在部署、上線一個新專案時，通常我們會先寫好該專案的一些 yaml 配置檔，包含 deployment / service / ingress / configmap 等等。將服務的一系列會用的的資源定義好，最後 apply 到 cluster 裡面。

而最後一步，我們通常需要依據定義的 ingress 裡面的 host 來將 DNS Record 建立在 DNS Provider 上，有可能是在 Cloudflare / AWS Route53 / Godaddy 之類的，而每次一個新專案要上線都比需要這麼做。這個設定的工作就好像一個固定形式的 SOP，久而久之就開始感到無趣。

身為一個 devops 工程師，這種 SOP 的工作當然能希望自動(智慧)一點，因此就開始調研怎麼樣能夠達成，也就有了此篇關於 External DNS 的文件紀錄。

# 環境
基於 eks，並在 eks 裝有 ingress-nginx。採用 Cloudflare 作為 DNS Provider。

> 當部署上去之後，最終 DNS Provider 會在 DNS Record 幫我們創建一筆 CNAME Record 並指向我在架構上的 AWS NLB FQDN 位址
{: .prompt-tip }

# 關於 External DNS 工具
簡單來說就是透過部署 kubernetes 的 service 或 ingress 資源時，同時在 DNS Provider (此篇是採用 Cloudflare) 上面創建 DNS Record 來讓 DNS 能指向 EKS 的位置。

External DNS 透過 DNS 的 txt type 來辨識它所管理的 records。(這點會在部署之後，創建 record 的同時也會建立一筆 txt type 的 record)

# 準備
- Cloudflare 的 API_TOKEN (Token 的權限要對全部的 Zone 有 Read 權限、對 DNS 有 Edit 的權限)
- Cloudflare 的 Zone_id (限制在某個 Zone)

# 流程
1. 創建 cloudflare 對應權限的 API token
2. 調整 external-dns 的參數
3. 透過 helm 部署 external-dns
4. 部署一個範例服務 deployment /service / ingress 並測試可否透過 DNS 訪問到該服務
5. 驗證 (查看 cloudflare 的 dns record / 查看 external-dns 的 pod logs)

## 創建 cloudflare 對應權限的 API Token
![](/commons/image/20220727/000_externaldns.png)

## 調整 external-dns 參數，這邊是用 helm chart(版本為 1.10.1)，故為調整 values.yaml

```yaml
# 主要調整一下幾個字段

## 注入 cloudflare 對應權限的 token
env:
  - name: CF_API_TOKEN
    value: "{剛剛創建的 cloudflare api token}"

## 僅 ingress 來源就好
sources:
  - ingress

## 指定 cloudflare 這個 DNS provider (預設是 aws)
provider: cloudflare

## 可選 - 固定 service account 的名稱
serviceAccount:
  create: true
  annotations: {}
  name: "external-dns"

## 可選 - 我這邊有控制 affinity 所以有特別調整 (一般情況下可不用調整)
affinity:
  nodeAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      nodeSelectorTerms:
      - matchExpressions:
        - key: eks.amazonaws.com/capacityType
          operator: In
          values:
          - ON_DEMAND

## 控制僅有指定的 domain - 把 example.com 改成自己的
domainFilters:
  - "example.com"

## 控制 dns 生命週期的 policy - 一般情況下應該用 upsert-only 避免意外刪除 dns reocrd。
## 但若是測試環境用，可用 sync (這樣 dns 紀錄才不會留一堆已經沒再用的 dns record 在上面)
policy: upsert-only

## 設定建立 txt 規則
## txtOwnerId = 自定義的唯一 id，如果有多個 kubernetes cluster，則每個 cluster 應有自己的一個唯一 id
registry: txt
txtOwnerId: "8jLp4X1xOY" 
txtPrefix: "test"
txtSuffix: ""

## 設定限制在以下的 zone-id
extraArgs:
  - --zone-id-filter={自己的 zone id}
```

## helm 部署 `helm install -n external-dns --create-namespace -f values.yaml external-dns .`
![](/commons/image/20220727/001_externaldns.png)

> 部署 sample-app 並驗證 DNS Records
![](/commons/image/20220727/002_externaldns.png)
![](/commons/image/20220727/003_externaldns.png)
![](/commons/image/20220727/004_externaldns.png)
![](/commons/image/20220727/005_externaldns.png)

## 補充 - 在 ingress 上透過 annotations `external-dns.alpha.kubernetes.io/hostname: sample.example.com` (example.com 要換成自己的 domain) 來宣告 CNAME Record 的 Name。
{: .prompt-info }

### 最終我用來測試用的 sample-app.yml
```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: sample-app
  annotations:
    kubernetes.io/ingress.class: nginx
    external-dns.alpha.kubernetes.io/hostname: sample.example.com
    external-dns.alpha.kubernetes.io/ttl: "120"
spec:
  rules:
  - host: sample.example.com
    http:
      paths:
      - path: /
        backend:
          service:
            name: sample-app
            port:
              number: 80
        pathType: Prefix

---

apiVersion: v1
kind: Service
metadata:
  name: sample-app
spec:
  ports:
  - port: 80
    targetPort: 80
  selector:
    app: sample-app

---

apiVersion: apps/v1
kind: Deployment
metadata:
  name: sample-app
spec:
  selector:
    matchLabels:
      app: sample-app
  template:
    metadata:
      labels:
        app: sample-app
    spec:
      containers:
      - image: nginx
        name: nginx
        ports:
        - containerPort: 80
```

# 參考文件
- 參考 <https://github.com/kubernetes-sigs/external-dns/blob/master/docs/tutorials/cloudflare.md>
- 參考 <https://github.com/kubernetes-sigs/external-dns/blob/master/docs/tutorials/nginx-ingress.md>
- 常見問題 <https://github.com/kubernetes-sigs/external-dns/blob/master/docs/faq.md>