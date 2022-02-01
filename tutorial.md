# **マルチクラスタ ゲートウェイ (Multi-cluster Gateways) ハンズオン**

<walkthrough-watcher-constant key="region-1" value="asia-northeast1"></walkthrough-watcher-constant>
<walkthrough-watcher-constant key="region-2" value="asia-northeast2"></walkthrough-watcher-constant>
<walkthrough-watcher-constant key="zone-1" value="asia-northeast1-b"></walkthrough-watcher-constant>
<walkthrough-watcher-constant key="zone-2" value="asia-northeast2-b"></walkthrough-watcher-constant>
<walkthrough-watcher-constant key="cluster-name-1" value="mc-gw-tokyo"></walkthrough-watcher-constant>
<walkthrough-watcher-constant key="cluster-name-2" value="mc-gw-osaka"></walkthrough-watcher-constant>
<walkthrough-watcher-constant key="config-cluster-name" value="mc-gw-config-cluster"></walkthrough-watcher-constant>
<walkthrough-watcher-constant key="instance-type" value="n1-standard-1"></walkthrough-watcher-constant>

## ハンズオンの内容

Google Kubernetes Engine (GKE) をマルチリージョン、マルチクラスタ構成で構築し、Kubernetes の Gateway API を利用して、Google Cloud Load Balancing (GCLB) でトラフィックを各クラスタに分散させる方法を学びます。

## 1. 環境準備

### Google Cloud Platform（GCP）プロジェクトの選択

ハンズオンを行う GCP プロジェクトを選択し、 **Start** をクリックしてください。

<walkthrough-project-setup>
</walkthrough-project-setup>
```

### gcloud から利用する GCP のデフォルトプロジェクトを設定する

```bash
gcloud config set project {{project-id}}
```

### Project Number を取得し、環境変数に定義する

```bash
export PROJECT_NUMBER=$( gcloud projects describe {{project-id}} --format='get(projectNumber)' )
```

### ハンズオンで利用する GCP の API を有効化する

```bash
gcloud services enable \
  container.googleapis.com \
  gkehub.googleapis.com \
  multiclusterservicediscovery.googleapis.com \
  multiclusteringress.googleapis.com \
  trafficdirector.googleapis.com
```

## 2. GKE クラスタをデプロイする

### マルチクラスタ ゲートウェイのリソースをホストする構成クラスタ（後述）を東京リージョンに作成する

```bash
gcloud container clusters create {{config-cluster-name}} \
    --zone={{zone-1}} \
    --enable-ip-alias \
    --workload-pool={{project-id}}.svc.id.goog \
    --release-channel stable \
    --machine-type {{instance-type}} \
```

### 東京リージョンのクラスターを作成する

(時間短縮のため、Cloud Shell の別タブで並行して実行した方が良いです)

```bash
gcloud container clusters create {{cluster-name-1}} \
    --zone={{zone-1}} \
    --enable-ip-alias \
    --workload-pool={{project-id}}.svc.id.goog \
    --release-channel stable \
    --machine-type {{instance-type}} \
```

### 大阪リージョンのクラスターを作成する

(時間短縮のため、Cloud Shell の別タブで並行して実行した方が良いです)

```bash
gcloud container clusters create {{cluster-name-2}} \
    --zone={{zone-2}} \
    --enable-ip-alias \
    --workload-pool={{project-id}}.svc.id.goog \
    --release-channel stable \
    --machine-type {{instance-type}} \
```

### GKE クラスターにアクセスするための認証情報を取得する

```bash
gcloud container clusters get-credentials {{config-cluster-name}} --zone={{zone-1}}
```
```bash
gcloud container clusters get-credentials {{cluster-name-1}} --zone={{zone-1}}
````
```bash
gcloud container clusters get-credentials {{cluster-name-2}} --zone={{zone-2}}
```

### 後で参照しやすいようクラスタのコンテキスト名を変更しておきます。

```bash
kubectl config rename-context gke_{{project-id}}_{{zone-1}}_{{config-cluster-name}} {{config-cluster-name}}
```
```bash
kubectl config rename-context gke_{{project-id}}_{{zone-1}}_{{cluster-name-1}} {{cluster-name-1}}
```
```bash
kubectl config rename-context gke_{{project-id}}_{{zone-2}}_{{cluster-name-2}} {{cluster-name-2}}
```

## 3. GKE Hub に登録する

### Hub に登録する

これにより、各クラスタがプロジェクトのフリート（マルチクラスタ ゲートウェイのターゲットとなる GKE クラスタを含むリソース）にマッピングされます。

```bash
gcloud container hub memberships register {{config-cluster-name}} \
    --gke-cluster {{zone-1}}/{{config-cluster-name}} \
    --enable-workload-identity
```
```bash
gcloud container hub memberships register {{cluster-name-1}} \
    --gke-cluster {{zone-1}}/{{cluster-name-1}} \
    --enable-workload-identity
```
```bash
gcloud container hub memberships register {{cluster-name-2}} \
    --gke-cluster {{zone-2}}/{{cluster-name-2}} \
    --enable-workload-identity
```

### クラスタが GKE Hub に正常に登録されたことを確認

```bash
gcloud container hub memberships list
```

以下のような情報が出力されれば問題ありません

```text
NAME                  EXTERNAL_ID
{{config-cluster-name}}  97892bcd-54f2-45ec-b46c-b97cdd8f0774
{{cluster-name-1}}           e9db39b9-a1d0-4a59-99cd-2bb8fe51f38d
{{cluster-name-2}}           b10ec208-4aa8-4aec-970f-20300bdb5095
```

## 4. マルチクラスタ Service を有効にする

### 登録済みクラスタのフリートでマルチクラスタ サービスを有効化

Hub に登録されているのクラスタの MCS コントローラが有効になり、Service のリッスンとエクスポートを開始できます。

```bash
gcloud container hub multi-cluster-services enable
```

### MCS に必要な IAM 権限を付与

```bash
gcloud projects add-iam-policy-binding {{project-id}} \
    --member "serviceAccount:{{project-id}}.svc.id.goog[gke-mcs/gke-mcs-importer]" \
    --role "roles/compute.networkViewer"
```

### 登録済みクラスタで MCS が有効になっていることを確認する

登録された 3 つのクラスタのメンバーシップが表示されます。すべてのクラスタが表示されるまでに数分かかることがあります。

```bash
gcloud container hub multi-cluster-services describe
```

## 5. Gateway API CRD をインストールする

GKE でゲートウェイ リソースを使用する前に、クラスタに Gateway API カスタム リソース定義（CRD）をインストールする必要があります。

```bash
kubectl kustomize "github.com/kubernetes-sigs/gateway-api/config/crd?ref=v0.3.0" | kubectl apply --context {{config-cluster-name}} -f -
```

次の CRD がインストールされます。

```text
customresourcedefinition.apiextensions.k8s.io/backendpolicies.networking.x-k8s.io created
customresourcedefinition.apiextensions.k8s.io/gatewayclasses.networking.x-k8s.io created
customresourcedefinition.apiextensions.k8s.io/gateways.networking.x-k8s.io created
customresourcedefinition.apiextensions.k8s.io/httproutes.networking.x-k8s.io created
customresourcedefinition.apiextensions.k8s.io/tcproutes.networking.x-k8s.io created
customresourcedefinition.apiextensions.k8s.io/tlsroutes.networking.x-k8s.io created
customresourcedefinition.apiextensions.k8s.io/udproutes.networking.x-k8s.io created
```

次のステップでコントローラを有効にすると、クラスタにマルチクラスタ GatewayClass がインストールされ、マルチクラスタ ゲートウェイのデプロイが可能になります。

## 6. マルチクラスタ ゲートウェイ コントローラを有効にする

GKE Gateway コントローラは、Google が Cloud Load Balancing に実装した Gateway API です。  
Gateway API リソースの Kubernetes API を監視し、Cloud Load Balancing リソースを調整して、Gateway リソースで指定されたネットワーク動作を実装します。

![architecture](https://cloud.google.com/kubernetes-engine/images/gateway-controller-architecture.svg)

### マルチクラスタ GKE Gateway コントローラを有効にして、構成クラスタを指定

マルチクラスタ ゲートウェイのリソースをホストする構成クラスタとして `{{config-cluster-name}}` を指定しています。

```bash
gcloud container hub ingress enable \
    --config-membership=/projects/{{project-id}}/locations/global/memberships/{{config-cluster-name}}
```

### 登録済みクラスタでグローバル GKE ゲートウェイ コントローラが有効になっていることを確認する

```bash
gcloud container hub ingress describe
```

### Gateway コントローラに必要な IAM 権限を付与する

```bash
gcloud projects add-iam-policy-binding {{project-id}} \
    --member "serviceAccount:service-${PROJECT_NUMBER}@gcp-sa-multiclusteringress.iam.gserviceaccount.com" \
    --role "roles/container.admin"
```

`ingress` Hub 機能が有効になると、構成クラスタでマルチクラスタ GatewayClass が使用可能になります。  
GatewayClasses のリストで、外部マルチクラスタ ゲートウェイには `gke-l7-gxlb-mc` が、内部マルチクラスタ ゲートウェイには `gke-l7-rilb-mc` が表示されます。  
これで、これらの GatewayClass を使用してマルチクラスタ ゲートウェイを作成できるようになります。

### GatewayClass の確認

```bash
kubectl get gatewayclasses --context={{config-cluster-name}}
```

出力は次のようになります。

```text
NAME             CONTROLLER
gke-l7-gxlb      networking.gke.io/gateway
gke-l7-gxlb-mc   networking.gke.io/gateway
gke-l7-rilb      networking.gke.io/gateway
gke-l7-rilb-mc   networking.gke.io/gateway
```

## 7. デモアプリケーションをデプロイする

ここでは、2 つの GKE クラスタのアプリケーション間で外部トラフィックを分散させる外部マルチクラスタ ゲートウェイを作成します。
(下図はイメージです)
![architecture](https://cloud.google.com/kubernetes-engine/images/multi-cluster-gateway-ex1.svg)


以下の手順で次の操作をします。
1. `{{cluster-name-1}}` クラスタと `{{cluster-name-2}}` クラスタにサンプル store アプリケーションをデプロイします。
2. 各クラスタに ServiceExport リソースを構成して、Service をフリートにエクスポートします。
3. `gke-l7-gxlb-mc` Gateway と HTTPRoute を構成クラスタ `{{cluster-name-1}}` にデプロイします。

アプリケーションと Gateway リソースがデプロイされると、パスベースのルーティングを使用して 2 つの GKE クラスタ間のトラフィックを制御できます。

### 東京と大阪、両方のクラスタにデモアプリケーションをデプロイ

両方のクラスタに、store Deployment と Namespace を作成します。

```bash
kubectl apply --context {{cluster-name-1}} -f https://raw.githubusercontent.com/GoogleCloudPlatform/gke-networking-recipes/master/gateway/gke-gateway-controller/multi-cluster-gateway/store.yaml
```
```bash
kubectl apply --context {{cluster-name-2}} -f https://raw.githubusercontent.com/GoogleCloudPlatform/gke-networking-recipes/master/gateway/gke-gateway-controller/multi-cluster-gateway/store.yaml
```

RUNNING 状態になったかどうかを確認します。

```bash
kubectl get pod --context {{cluster-name-1}} -n store
```
```bash
kubectl get pod --context {{cluster-name-2}} -n store
```

## 8. マルチクラスタ サービスについて（説明）

マルチクラスタ ゲートウェイ コントローラは、MCS API リソースを使用して、複数のクラスタにまたがってアドレス指定可能な Service にグループ化します。

### ServiceExport 
Kubernetes Service にマッピングされ、フリートに登録されているすべてのクラスタに、その Service のエンドポイントをエクスポートします。  
Service に対応する ServiceExport がある場合、Service はマルチクラスタ ゲートウェイでアドレス指定できます。

### ServiceImport
マルチクラスタ サービス コントローラによって自動的に生成されます。  
ServiceExport と ServiceImport はペアで提供されます。  
ServiceExport がフリートに存在する場合は、対応する ServiceImport が作成され、ServiceExport にマッピングされた Service にクラスタ間でアクセスできるようになります。  

### ServiceExport の例

store Service は `{{cluster-name-1}}` に存在し、そのクラスタ内の Pod のグループを選択します。  
クラスタに ServiceExport が作成され、`{{cluster-name-1}}` 内の Pod にフリートの他のクラスタからアクセスできるようになります。  
ServiceExport は、ServiceExport リソースと同じ name および namespace を持つ Service にマッピングされ、公開されます。

```yaml
apiVersion: v1
kind: Service
metadata:
  name: store
  namespace: store
spec:
  selector:
    app: store
  ports:
  - port: 8080
    targetPort: 8080
---
kind: ServiceExport
apiVersion: net.gke.io/v1
metadata:
  name: store
  namespace: store
```

ServiceExport と Service のペアが存在する場合、マルチクラスタの Service コントローラは、対応する ServiceImport をフリート内のすべての GKE クラスタにデプロイします。  

![ServiceExport](https://cloud.google.com/kubernetes-engine/images/multi-cluster-service-example1.svg)

マルチクラスタ ゲートウェイでは、Gateway は、別のクラスタに存在する Service または複数のクラスタにまたがる Service の論理識別子として ServiceImport を使用します。  
次の HTTPRoute は Service リソースの代わりに ServiceImport を参照します。  
ServiceImport を参照することで、1 つ以上のクラスタで実行されているバックエンド Pod のグループにトラフィックを転送していることを示します。

```yaml
kind: HTTPRoute
apiVersion: networking.x-k8s.io/v1alpha1
metadata:
  name: store-route
  namespace: store
  labels:
    gateway: multi-cluster-gateway
spec:
  hostnames:
  - "store.example.com"
  rules:
  - forwardTo:
    - backendRef:
        group: net.gke.io
        kind: ServiceImport
        name: store
      port: 8080
```

ロードバランサは、バックエンドを 1 つのバックエンド プールとして扱います。  
一方のクラスタの Pod が正常でなくなるか、到達不能であるか、トラフィック容量がない場合、トラフィック負荷はもう一方のクラスタの残りの Pod にロード バランシングされます。

![Multi Cluster Gateway](https://cloud.google.com/kubernetes-engine/images/multi-cluster-service-example2.svg)

## 9. Service の Export

### Service と ServiceExport のデプロイ

`{{cluster-name-1}}` に適用する manifest を `store-tokyo-service.yaml` という名前のファイルに保存します。
以下コピーして実行してください。

```text
cat <<EOL > store-tokyo-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: store
  namespace: store
spec:
  selector:
    app: store
  ports:
  - port: 8080
    targetPort: 8080
---
kind: ServiceExport
apiVersion: net.gke.io/v1
metadata:
  name: store
  namespace: store
---
apiVersion: v1
kind: Service
metadata:
  name: store-tokyo-1
  namespace: store
spec:
  selector:
    app: store
  ports:
  - port: 8080
    targetPort: 8080
---
kind: ServiceExport
apiVersion: net.gke.io/v1
metadata:
  name: store-tokyo-1
  namespace: store
EOL
```

デプロイします。

```bash
kubectl apply -f store-tokyo-service.yaml --context {{cluster-name-1}}
```

`{{cluster-name-2}}` に適用する manifest を `store-osaka-service.yaml` という名前のファイルに保存します。
以下コピーして実行してください。

```bash
cat <<EOL > store-osaka-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: store
  namespace: store
spec:
  selector:
    app: store
  ports:
  - port: 8080
    targetPort: 8080
---
kind: ServiceExport
apiVersion: net.gke.io/v1
metadata:
  name: store
  namespace: store
---
apiVersion: v1
kind: Service
metadata:
  name: store-osaka-1
  namespace: store
spec:
  selector:
    app: store
  ports:
  - port: 8080
    targetPort: 8080
---
kind: ServiceExport
apiVersion: net.gke.io/v1
metadata:
  name: store-osaka-1
  namespace: store
EOL
```

デプロイします。

```bash
kubectl apply -f store-osaka-service.yaml --context {{cluster-name-2}}
```

### ServiceExport の確認

クラスタに正しい ServiceExport が作成されていることを確認します。

```bash
kubectl get serviceexports --context {{cluster-name-1}} --namespace store
```
```bash
kubectl get serviceexports --context {{cluster-name-2}} --namespace store
```

以下のように出力されます。

```text
# {{cluster-name-1}}
NAME            AGE
store           2m40s
store-tokyo-1   2m40s

# {{cluster-name-2}}
NAME            AGE
store           2m25s
store-osaka-1   2m25s
```

数分後、付属の ServiceImports が、フリート内のすべてのクラスタにわたってマルチクラスタ Service コントローラによって自動的に作成されたことを確認します。

```bash
kubectl get serviceimports --context {{cluster-name-1}} --namespace store
```
```bash
kubectl get serviceimports --context {{cluster-name-2}} --namespace store
```

以下のように出力されます。

```text
# {{cluster-name-1}}
NAME            TYPE           IP                  AGE
store           ClusterSetIP   ["10.112.31.15"]    6m54s
store-osaka-1   ClusterSetIP   ["10.112.26.235"]   5m49s
store-tokyo-1   ClusterSetIP   ["10.112.16.112"]   6m54s

# gke-east-1
NAME            TYPE           IP                  AGE
store           ClusterSetIP   ["10.72.28.226"]   5d10h
store-osaka-1   ClusterSetIP   ["10.72.19.177"]   5d10h
store-tokyo-1   ClusterSetIP   ["10.72.28.68"]    4h32m
```

これは、3 つの Service がすべてフリートの両方のクラスタからアクセスできることを表しています。

## 10. Gateway と HTTPRoute をデプロイする

アプリケーションがデプロイされたら、`gke-l7-gxlb-mc` GatewayClass を使用してゲートウェイを構成できます。  
このゲートウェイは、ターゲット クラスタ間でトラフィックを分散する外部 HTTP(S) ロードバランサを作成します。

### Gateway manifest のデプロイ

`{{config-cluster-name}}` に store namespace を作成します。

```bash
kubectl create ns store --context {{config-cluster-name}}
```

`{{config-cluster-name}}` に適用する manifest を `external-http-gateway.yaml` という名前のファイルに保存します。

```text
cat <<EOL > external-http-gateway.yaml
kind: Gateway
apiVersion: networking.x-k8s.io/v1alpha1
metadata:
  name: external-http
  namespace: store
spec:
  gatewayClassName: gke-l7-gxlb-mc
  listeners:
  - protocol: HTTP
    port: 80
    routes:
      kind: HTTPRoute
      selector:
        matchLabels:
          gateway: external-http
EOL
```

デプロイします。

```bash
kubectl apply -f external-http-gateway.yaml --context {{config-cluster-name}}
```

### HTTPRoute manifest のデプロイ

`{{config-cluster-name}}` に適用する manifest を `public-store-route.yaml` という名前のファイルに保存します。

```text
cat <<EOL > public-store-route.yaml
kind: HTTPRoute
apiVersion: networking.x-k8s.io/v1alpha1
metadata:
  name: public-store-route
  namespace: store
  labels:
    gateway: external-http
spec:
  hostnames:
  - "store.example.com"
  rules:
  - forwardTo:
    - backendRef:
        group: net.gke.io
        kind: ServiceImport
        name: store
      port: 8080
  - matches:
    - path:
        type: Prefix
        value: /tokyo
    forwardTo:
    - backendRef:
        group: net.gke.io
        kind: ServiceImport
        name: store-tokyo-1
      port: 8080
  - matches:
    - path:
        type: Prefix
        value: /osaka
    forwardTo:
    - backendRef:
        group: net.gke.io
        kind: ServiceImport
        name: store-osaka-1
      port: 8080
EOL
```

デプロイします。

```bash
kubectl apply -f public-store-route.yaml --context {{config-cluster-name}} --namespace store
```

### デプロイ後、この HTTPRoute は次のルーティング動作を構成します。

* `/tokyo` へのリクエストは、`{{cluster-name-1}}` クラスタ内の `store` Pod にルーティングされます。
* `/osaka` へのリクエストは、`{{cluster-name-2}}` クラスタ内の `store` Pod にルーティングされます。
* その他のパスへのリクエストは、健全性（Health）、容量（capacity）、リクエスト元のクライアントへの近接度（proximity）に応じて、いずれかのクラスタにルーティングされます。

![Traffic routing](https://cloud.google.com/kubernetes-engine/images/multi-cluster-gateway-routing.svg)

特定のクラスタのすべての Pod が正常でない（または存在しない）場合、`store` Service へのトラフィックは、実際に `store` Pod を使用しているクラスタにのみ送信されます。

## 11. デプロイの検証

### Gateway と HTTPRoute が正常にデプロイされたことを確認

```bash
kubectl describe gateway external-http --context {{config-cluster-name}} --namespace store
```

出力は以下のようになります。

```text
Spec:
  Gateway Class Name:  gke-l7-gxlb-mc
  Listeners:
    Port:      80
    Protocol:  HTTP
    Routes:
      Group:  networking.x-k8s.io
      Kind:   HTTPRoute
      Namespaces:
        From:  Same
      Selector:
        Match Labels:
          Gateway:  external-http
Status:
  Addresses:
    Type:   IPAddress
    Value:  34.120.172.213
  Conditions:
    Last Transition Time:  1970-01-01T00:00:00Z
    Message:               Waiting for controller
    Reason:                NotReconciled
    Status:                False
    Type:                  Scheduled
Events:
  Type     Reason  Age                     From                     Message
  ----     ------  ----                    ----                     -------
  Normal   UPDATE  29m (x2 over 29m)       global-gke-gateway-ctlr  store/external-http
  Normal   SYNC    59s (x9 over 29m)       global-gke-gateway-ctlr  SYNC on store/external-http was a success
```

### Gateway から外部 IP アドレスを取得

```bash
export VIP=$( kubectl get gateway external-http -o=jsonpath="{.status.addresses[0].value}" --context {{config-cluster-name}} --namespace store )
```

### ドメインのルートパスにトラフィックを送信して挙動を確認

ロードバランサは最も近いリージョンにトラフィックを送信し、他のリージョンからのレスポンスが表示されない可能性があります。

```bash
curl -H "host: store.example.com" http://${VIP}
```

出力から、リクエストが `{{cluster-name-1}}` クラスタから Pod によって処理されたことを確認できます。

```text
{
"cluster_name": "{{cluster-name-1}}", 
"zone": "{{zone-1}}", 
"host_header": "store.example.com",
"node_name": "gke-{{cluster-name-1}}-default-pool-65059399-2f41.c.{{project-id}}.internal",
"pod_name": "store-5f5b954888-d25m5",
"pod_name_emoji": "🍾",
"project_id": "{{project-id}}",
"timestamp": "2022-01-27T17:39:15",
}
```

### トラフィックを `/tokyo` パスに送信する

```bash
curl -H "host: store.example.com" http://${VIP}/tokyo
```

出力から、リクエストが `{{cluster-name-1}}` クラスタから Pod によって処理されたことを確認できます。

```text
{
"cluster_name": "{{cluster-name-1}}", 
"zone": "{{zone-1}}", 
"host_header": "store.example.com",
"node_name": "gke-{{cluster-name-1}}-default-pool-65059399-2f41.c.{{project-id}}.internal",
"pod_name": "store-5f5b954888-d25m5",
"pod_name_emoji": "🍾",
"project_id": "{{project-id}}",
"timestamp": "2022-01-27T17:39:15",
}
```

### トラフィックを `/osaka` パスに送信する

```bash
curl -H "host: store.example.com" http://${VIP}/osaka
```

出力から、リクエストが `{{cluster-name-2}}` クラスタから Pod によって処理されたことを確認できます。

```text
{
"cluster_name": "{{cluster-name-2}}", 
"zone": "{{zone-2}}", 
"host_header": "store.example.com",
"node_name": "gke-{{cluster-name-2}}-default-pool-65059399-2f41.c.{{project-id}}.internal",
"pod_name": "store-5f5b954888-d25m5",
"pod_name_emoji": "🍾",
"project_id": "{{project-id}}",
"timestamp": "2022-01-27T17:39:15",
}
```

## Congraturations!

<walkthrough-conclusion-trophy></walkthrough-conclusion-trophy>

これにてマルチゲートウェイ体験するハンズオンは完了です！！  
TODO: cleanup
