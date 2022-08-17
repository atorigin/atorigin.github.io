---
title: 010_使用 helm 來管理 kubernetes application
date: '2022-08-17 16:09:00 +0800'
categories: [Course]
tags: [java]     # TAG names should always be lowercase
author: owen
mermaid: true
math: true
---

# 目標
隨著專案越來越多，管理 kubernetes 內部 deployment、service、ingress、configmaps 物件越來越多。對於多檔切換來切換去的管理也越來越不堪負荷，也害怕自己哪裡改壞了，沒正確上到對應的配置導致部署有問題沒發現。最近開始想要導入 helm 來做在 kubernetes 內的 application 做管理。

# Chart 基本概念

## Chart 基本結構

```yaml
mychart/
    Chart.yaml
    values.yaml
    charts/
    templates/
    ...
```

- templates 目錄存放 template 檔案，helm 在運行 chart 的同時，會將所有 templates 目錄裡面的檔案發送給 template rendering engine(模板渲染引擎)
- values.yaml 檔案對於 template 非常重要，這個檔案包含在整個 charts 的default values，當執行 `helm install` 或 `helm upgrade` 這些 values 會被用來覆寫。
- Chart.yaml 包含整個 chart 的描述，可以在 template 裡面訪問 Chart.yaml，而 charts/ 目錄會包含其他 chart，也可稱作為 subcharts

## 初探 templates/ 目錄的檔案
- NOTES.txt，一些提示字串，用來幫助安裝 chart 的時候顯示給 user 看的描述。
- deployment.yaml，基礎的 manifest，就是 kubernetes 的 deployment 物件。
- service.yaml，基礎的 manifest，就是 kubernetes 的 service 物件。
- _helpers.tpl，存放可以讓這個 chart 被重用的模板協助器。

> template 雖然沒有強制性規定命名規則，但 helm 建議還是用 `.yaml` 來作為 YAML 檔案的 suffix，且用 `.tpl` 作為模板協助器檔案的 suffix
{: .prompt-tip }

## Part1 測試基本渲染 template 的方式

### 先創建 helm chart
```bash
helm create demo
```

> demo 為 chart name 可以隨意取，完成後會依照取的名子生成一個目錄。
{: .prompt-info }


### 移除 templates/ 目錄下所有檔案
```bash
rm -rf demo/templates/*
```

### 在 templates/ 目錄下新增一個 `configmaps.yaml` 檔案
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
{% raw %}  name: {{ .Release.Name }}-configmap {% endraw %}
data:
  myvalue: "Hello World"
```

> 這邊的關鍵在 metadata 下面的 name，後面用了一個 {% raw %} `{{ .Release.Name }}` {% endraw %}。
{: .prompt-info }

### 關於 `.Release.Name` 的解讀
這邊的定義可以解讀為，從 top namespace 開始找 Release 物件，並從 Release 物件裡找到一個物件叫做 Name 的。

> Release 物件是 helm 物件的一種
{: .prompt-tip }

## helm 內建的物件
- 把定義的物件透過 template engine 傳入 template
- 測試 `with` 和 `range` statements
- 甚至有方法能直接在 template 裡面 new 一個物件，例如用 `tuple` function

> 物件可以單純是一個值，或是其它物件或 functions。
{: .prompt-tip }

### Release 物件
- Release 物件是在 template 中可以訪問的最高級物件
- Release 物件用來描述它自己本身，它內部有幾個物件
    - Release.Name 該 release 的名稱
    - Release.Namespace 該 namespace，如果 manifest 裡面沒有宣告
    - Release.Revision 該 release 的 revision 號碼，install 後為 1，往後 upgrade 會往上加，或 rollback 往下減

### Values 物件
- Values 透過 values.yaml 檔案傳入 template，預設情況下 Values 是空的

### Chart 物件
- 是 Chart.yaml 檔案的內容
- 在 Chart.yaml 檔案內的可用參數可參考 <https://helm.sh/docs/topics/charts/#the-chartyaml-file>


### Files 物件
- 用來提供在 charts 裡面訪問所有非特定 (non-special) 的檔案
- 檔案物件的使用方式，參考 <https://helm.sh/docs/chart_template_guide/accessing_files/>

### Capabilities 物件
- 提供 kubernetes cluster 支援的資訊

### Template 物件
- 包含當前被執行的 template 的資訊

> 補充，在 Go 裡面，大寫字母開頭通常被用來表示 build-in 的物件。是 Go 的命名原則。
{: .prompt-info }

## 進一步探討上面提到的 Value 物件
- 它可以透過多個來源來傳入 chart
    - 該 chart 裡面的 values.yaml 檔案
    - 如果是 subchart，來自 parent chart 的 values.yaml
    - 執行 command 時帶入的 `-f` 參數的指定 values.yaml 檔案
        - 例如 `helm install -f myvalues.yaml ./mychart`
    - 執行 command 時代入的 `--set` 參數指定的方式
        - 例如 `helm install --set foo=bar ./mychart`

### 順序優先級
- values.yaml 是最先的
- 如果 subchart 內的 values.yaml 值不存在，調用 parent 的 values.yaml 來覆蓋
- 而這些 values.yaml 檔可以被 user 定義的 `-f` 和 `--set` 參數覆蓋

> 測試為 `--set foo=bar` > `-f myvalue.yaml` > `values.yaml`
{: .prompt-tip }