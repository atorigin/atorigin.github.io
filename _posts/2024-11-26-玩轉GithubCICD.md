---
title: 023_Github CI/CD
date: '2024-11-26 00:00:00 +0800'
categories: [DevOps]
tags: [kubernetes]     # TAG names should always be lowercase
author: owen
mermaid: true
math: true
---

# 起因
因為公司 Github Org 的 CICD 日益漸增，加上目前組內討論想要把 CI/CD 的配置都統一管理到 Github Action，因此開始調研如何實作 ARC 有效利用測試區的 EKS 效能

# ARC 架構
![](/commons/image/20241126/arc-diagram.webp)

## 架構描述
1. 安裝完畢後，AutoScalingRunnerSet Controller 呼叫 githun API 取得這個 runner scale set 屬於哪一個 runner group ID
2. AutoScalingRunnerSet 在建立 Runner ScaleSet Listener 之前，呼叫 Github API 取得 Github Action 服務建立 runner scale set.
3. Runner ScaleSet Listener 由 AutoScalingListener Controller 佈署，Runner ScaleSet Listener 會向 github action 建立連線並進行身份驗證，且開始監聽
4. 當 Repo 發生 action event，github action 會分配獨立的 job 給 runner 或 runner scalesets，注意 `runs-on` 的 keyword 值需符合註冊 runner scaleset 的 label
5. 當 Runner ScaleSet Listener 收到 Job Available，他會確定能不能 scale up 到描述的 desired count，如果可以，Runner ScaleSet Listener 會 acknowledges 這個 message
6. Runner ScaleSet Listener 會透過 Kubernetes RBAC 發 HTTP Req 給 Kubernetes API
7. Ephemeral RunnerSet 會嘗試創建 runner pods，如果 pods 創建失敗 controller 會嘗試最多 5 次在 24hr 內，若都無法成功，則 job 會被 cancel
8. 當 runner pods 被創建成功，他會用 JIT (Just-In-Time) Token 去跟 Github Action 服務註冊並發出另外的 HTTP long poll connection 來接收要執行的 job details
9. Github action 會分發 job 給註冊過的 runner pods
10. 當 runner 把 job 執行完畢，EphemeralRunner Controller 檢查 Github Action 確認 runner 能不能 delete，如果可以 執行完畢，Ephemeral RunnerSet 刪除這個 runner

# 安裝
> 主要分兩塊，一個是如何跟 Github 驗證，一個是如何讓 Kubernetes 能跟 Github Action 交互
{: .prompt-tip }
1. [authenticating](https://docs.github.com/en/actions/hosting-your-own-runners/managing-self-hosted-runners-with-actions-runner-controller/authenticating-to-the-github-api)
2. [action runner controller installation](https://docs.github.com/en/actions/hosting-your-own-runners/managing-self-hosted-runners-with-actions-runner-controller/deploying-runner-scale-sets-with-actions-runner-controller)

## Authenticating
1. 進入組織設定頁面，進入 developer settings -> Github App
![](/commons/image/20241126/000.png)
![](/commons/image/20241126/001.png)
![](/commons/image/20241126/002.png)

2. New 一個 Github App，這邊要取得一些驗證用的關鍵資訊
    a. AppID
    b. Private Key .pem 格式
    c. installation id
![](/commons/image/20241126/003.png)

3. 在 helm chart 裡面指定一個 kubernetes secret
![](/commons/image/20241126/004.png)

4. 定義 chart values YAML

```yaml
runnerScaleSetName: "vivotek-arc" # important , must to match workflow 'runs-on'
githubConfigUrl: <your-repository-url>
githubConfigSecret: pre-defined-secret
minRunners: 0
maxRunners: 5
```

5. helm install
```bash
helm install --create-namespace -n arc-runners -f values.yaml <runner-label> ./
```

6. apply secret to arc-runners namespace
```bash
kubectl apply -f pre-defined-secret.yaml
```

> 創建 secret 用於 runner 和 github 之間驗證
{: .prompt-tip }

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: pre-defined-secret
  namespace: arc-runners
type: Opaque
data:
  github_app_id: <base64endcoded-app-id>
  github_app_installation_id: <base64encoded-installation-id>
  github_app_private_key: <base64endcoded-pem>
```
