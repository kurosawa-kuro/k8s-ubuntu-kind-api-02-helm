# Kubernetes API Sample with Helm Chart

このプロジェクトは、Kubernetes上で動作するAPIサンプルアプリケーションをHelmチャートとしてデプロイするためのものです。

## 前提条件

- Kubernetesクラスタ（Kind、Minikube、EKS等）
- Helm 3.x
- kubectl
- Docker（イメージビルド用）

## プロジェクト構造

まず、以下のコマンドでHelmチャートを作成します。

```bash
cd ~/dev/k8s-ubuntu-kind-api-02-helm
helm create k8s-api-sample
```

その後、プロジェクト構造は次のようになります：

```
k8s-api-sample/
├── templates/           # Helmテンプレート
│   ├── deployment.yaml  # デプロイメント設定
│   ├── service.yaml     # サービス設定
│   ├── ingress.yaml     # Ingress設定
│   ├── hpa.yaml         # 水平Pod自動スケーリング
│   └── serviceaccount.yaml  # ServiceAccount設定
├── values.yaml          # 設定値
└── Chart.yaml           # チャート定義
```

## 実装詳細

### 1. Chart.yaml

```yaml
apiVersion: v2
name: k8s-api-sample
description: A Helm chart for Kubernetes
type: application
version: 0.1.0
appVersion: "1.16.0"
```

### 2. values.yaml

```yaml
replicaCount: 1

image:
  repository: 503561449641.dkr.ecr.ap-northeast-1.amazonaws.com/k8s-api-sample
  pullPolicy: Always
  tag: latest

service:
  type: ClusterIP
  port: 80
  targetPort: 3000

ingress:
  enabled: true
  className: nginx
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
  hosts:
    - host: localhost
      paths:
        - path: /
          pathType: Prefix
  tls: []

resources:
  requests:
    cpu: "100m"
    memory: "128Mi"
  limits:
    cpu: "200m"
    memory: "256Mi"

imagePullSecrets:
  - name: regcred

serviceAccount:
  create: true
  name: ""
  automount: true

autoscaling:
  enabled: true
  minReplicas: 1
  maxReplicas: 3
  targetCPUUtilizationPercentage: 80
  targetMemoryUtilizationPercentage: 80
```

### 3. templates/deployment.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Release.Name }}
  labels:
    app: {{ .Release.Name }}
    app.kubernetes.io/managed-by: {{ .Release.Service }}
  annotations:
    meta.helm.sh/release-name: {{ .Release.Name }}
    meta.helm.sh/release-namespace: {{ .Release.Namespace }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      app: {{ .Release.Name }}
  template:
    metadata:
      labels:
        app: {{ .Release.Name }}
        app.kubernetes.io/managed-by: {{ .Release.Service }}
    spec:
      imagePullSecrets: {{ .Values.imagePullSecrets | toYaml | nindent 8 }}
      containers:
        - name: {{ .Release.Name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          ports:
            - containerPort: {{ .Values.service.targetPort }}
          resources:
            requests:
              cpu: {{ .Values.resources.requests.cpu }}
              memory: {{ .Values.resources.requests.memory }}
            limits:
              cpu: {{ .Values.resources.limits.cpu }}
              memory: {{ .Values.resources.limits.memory }}
```

### 4. templates/service.yaml

```yaml
apiVersion: v1
kind: Service
metadata:
  name: {{ .Release.Name }}
  labels:
    app: {{ .Release.Name }}
    app.kubernetes.io/managed-by: {{ .Release.Service }}
  annotations:
    meta.helm.sh/release-name: {{ .Release.Name }}
    meta.helm.sh/release-namespace: {{ .Release.Namespace }}
spec:
  selector:
    app: {{ .Release.Name }}
  ports:
    - protocol: TCP
      port: {{ .Values.service.port }}
      targetPort: {{ .Values.service.targetPort }}
      name: http
  type: {{ .Values.service.type }}
```

### 5. templates/ingress.yaml

```yaml
{{- if .Values.ingress.enabled -}}
{{- $fullName := include "k8s-api-sample.fullname" . -}}
{{- $svcPort := .Values.service.port -}}
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: {{ $fullName }}
  labels:
    {{- include "k8s-api-sample.labels" . | nindent 4 }}
    app.kubernetes.io/managed-by: {{ .Release.Service }}
  annotations:
    {{- with .Values.ingress.annotations }}
    {{- toYaml . | nindent 4 }}
    {{- end }}
    meta.helm.sh/release-name: {{ .Release.Name }}
    meta.helm.sh/release-namespace: {{ .Release.Namespace }}
{{- if and .Values.ingress.className (semverCompare ">=1.18-0" .Capabilities.KubeVersion.GitVersion) }}
  ingressClassName: {{ .Values.ingress.className }}
{{- end }}
spec:
  rules:
    {{- range .Values.ingress.hosts }}
    - host: {{ .host | quote }}
      http:
        paths:
          {{- range .paths }}
          - path: {{ .path }}
            pathType: {{ .pathType }}
            backend:
              service:
                name: {{ $fullName }}
                port:
                  number: {{ $svcPort }}
          {{- end }}
    {{- end }}
{{- end }}
```

### 6. templates/hpa.yaml

```yaml
{{- if .Values.autoscaling.enabled }}
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: {{ include "k8s-api-sample.fullname" . }}
  labels:
    {{- include "k8s-api-sample.labels" . | nindent 4 }}
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: {{ include "k8s-api-sample.fullname" . }}
  minReplicas: {{ .Values.autoscaling.minReplicas }}
  maxReplicas: {{ .Values.autoscaling.maxReplicas }}
  metrics:
    {{- if .Values.autoscaling.targetCPUUtilizationPercentage }}
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: {{ .Values.autoscaling.targetCPUUtilizationPercentage }}
    {{- end }}
    {{- if .Values.autoscaling.targetMemoryUtilizationPercentage }}
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: {{ .Values.autoscaling.targetMemoryUtilizationPercentage }}
    {{- end }}
{{- end }}
```

### 7. templates/serviceaccount.yaml

```yaml
{{- if .Values.serviceAccount.create -}}
apiVersion: v1
kind: ServiceAccount
metadata:
  name: {{ include "k8s-api-sample.serviceAccountName" . }}
  labels:
    {{- include "k8s-api-sample.labels" . | nindent 4 }}
  {{- with .Values.serviceAccount.annotations }}
  annotations:
    {{- toYaml . | nindent 4 }}
  {{- end }}
automountServiceAccountToken: {{ .Values.serviceAccount.automount }}
{{- end }}
```

### 8. templates/_helpers.tpl

```yaml
{{/*
Expand the name of the chart.
*/}}
{{- define "k8s-api-sample.name" -}}
{{- default .Chart.Name .Values.nameOverride | trunc 63 | trimSuffix "-" }}
{{- end }}

{{/*
Create a default fully qualified app name.
*/}}
{{- define "k8s-api-sample.fullname" -}}
{{- if .Values.fullnameOverride }}
{{- .Values.fullnameOverride | trunc 63 | trimSuffix "-" }}
{{- else }}
{{- $name := default .Chart.Name .Values.nameOverride }}
{{- if contains $name .Release.Name }}
{{- .Release.Name | trunc 63 | trimSuffix "-" }}
{{- else }}
{{- printf "%s-%s" .Release.Name $name | trunc 63 | trimSuffix "-" }}
{{- end }}
{{- end }}
{{- end }}

{{/*
Common labels
*/}}
{{- define "k8s-api-sample.labels" -}}
helm.sh/chart: {{ include "k8s-api-sample.chart" . }}
{{ include "k8s-api-sample.selectorLabels" . }}
{{- if .Chart.AppVersion }}
app.kubernetes.io/version: {{ .Chart.AppVersion | quote }}
{{- end }}
app.kubernetes.io/managed-by: {{ .Release.Service }}
{{- end }}

{{/*
Selector labels
*/}}
{{- define "k8s-api-sample.selectorLabels" -}}
app.kubernetes.io/name: {{ include "k8s-api-sample.name" . }}
app.kubernetes.io/instance: {{ .Release.Name }}
{{- end }}
```

## デプロイ手順

### 1. イメージのビルドとプッシュ

```bash
# ECRログイン
aws ecr get-login-password --region ap-northeast-1 | docker login --username AWS --password-stdin 503561449641.dkr.ecr.ap-northeast-1.amazonaws.com

# イメージビルド
docker build -t k8s-api-sample .

# タグ付け
docker tag k8s-api-sample:latest 503561449641.dkr.ecr.ap-northeast-1.amazonaws.com/k8s-api-sample:latest

# プッシュ
docker push 503561449641.dkr.ecr.ap-northeast-1.amazonaws.com/k8s-api-sample:latest
```

### 2. ECR認証情報の設定

```bash
# ECRの認証情報をKubernetesシークレットとして作成
kubectl create secret docker-registry regcred \
  --docker-server=503561449641.dkr.ecr.ap-northeast-1.amazonaws.com \
  --docker-username=AWS \
  --docker-password=$(aws ecr get-login-password --region ap-northeast-1)
```

### 3. Helmチャートのデプロイ

```bash
# 既存のリリースを削除（必要な場合）
helm uninstall k8s-api-sample

# チャートのインストール
helm install k8s-api-sample ./k8s-api-sample
```

### 4. デプロイ状態の確認

```bash
# すべてのリソースの確認
kubectl get all

# ログの確認
kubectl logs deployment/k8s-api-sample

# Ingressの確認
kubectl get ingress
```

## APIエンドポイント

デプロイ後、以下のエンドポイントでAPIにアクセスできます：

- `http://localhost/posts` - 投稿一覧の取得
- `http://localhost/posts/:id` - 特定の投稿の取得
- `http://localhost/posts` (POST) - 新規投稿の作成
- `http://localhost/posts/:id` (PUT) - 投稿の更新
- `http://localhost/posts/:id` (DELETE) - 投稿の削除

## トラブルシューティング

1. **イメージプルエラー**
   - ECR認証情報が正しく設定されているか確認
   - `kubectl get secret regcred`でシークレットの存在を確認

2. **Ingress接続エラー**
   - Ingressコントローラーが正しくインストールされているか確認
   - `kubectl get ingress`でIngressの状態を確認

3. **Pod起動エラー**
   - `kubectl describe pod <pod-name>`で詳細なエラー情報を確認
   - `kubectl logs <pod-name>`でログを確認

## アンインストール

```bash
# Helmリリースの削除
helm uninstall k8s-api-sample

# 関連リソースの削除
kubectl delete secret regcred
```

以下は、Helmでよく使用される設定パターンの整理された説明です。これらの設定を活用することで、Kubernetes環境でのアプリケーションの運用が最適化され、安定性とスケーラビリティを確保できます。

---

### 1. レプリカ数 (replicaCount)
- **目的**: サービスの可用性とスケーラビリティを確保するため。
- **理想状態**: `replicaCount` は、障害発生時に他のPodがリクエストを処理できるように設定します。通常は2以上に設定し、可用性を確保します。

```yaml
replicaCount: 2
```

---

### 2. CPUリソースとメモリリソース (resources)
- **目的**: アプリケーションが適切に動作するため、Kubernetesがリソースを適切にスケジューリングできるようにするため。
- **理想状態**: `requests` はPodが正常に動作するために必要な最小限のリソースを、`limits` はリソース使用量を制限します。

```yaml
resources:
  requests:
    cpu: "100m"
    memory: "128Mi"
  limits:
    cpu: "200m"
    memory: "256Mi"
```

---

### 3. ポート番号 (service.port / service.targetPort)
- **目的**: 外部からのアクセスと内部での通信を正しくルーティングするため。
- **理想状態**: `service.port` は外部からアクセスする公開ポート、`service.targetPort` はコンテナ内でリッスンしているポートを指定します。

```yaml
service:
  type: ClusterIP
  port: 80
  targetPort: 3000
```

---

### 4. Ingress設定 (ingress)
- **目的**: アプリケーションを外部からアクセス可能にし、URLベースのルーティングを実現するため。
- **理想状態**: `ingress.enabled: true` でIngressを有効にし、外部からのアクセスを許可します。

```yaml
ingress:
  enabled: true
  className: nginx
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
  hosts:
    - host: localhost
      paths:
        - path: /
          pathType: Prefix
  tls: []
```

---

### 5. 自動スケーリング (Horizontal Pod Autoscaling, HPA)
- **目的**: リソースの需要に応じて自動でPodのレプリカ数をスケールさせるため。
- **理想状態**: `autoscaling.enabled: true` で自動スケーリングを有効にし、負荷に応じてPodの数を増減させます。

```yaml
autoscaling:
  enabled: true
  minReplicas: 1
  maxReplicas: 3
  targetCPUUtilizationPercentage: 80
  targetMemoryUtilizationPercentage: 80
```

---

### 6. ServiceAccountの設定
- **目的**: Kubernetesのリソースにアクセスするための認証情報を管理するため。
- **理想状態**: `serviceAccount.create: true` に設定することで、必要なServiceAccountを自動的に作成します。

```yaml
serviceAccount:
  create: true
  name: ""
  automount: true
```

---

### 7. Image Pull Secrets
- **目的**: プライベートレジストリからイメージをプルする際に認証を行うため。
- **理想状態**: `imagePullSecrets` を設定し、プライベートリポジトリから認証情報を使用してイメージをプルします。

```yaml
imagePullSecrets:
  - name: regcred
```

---

### 8. 環境変数 (env)
- **目的**: アプリケーションが動作する環境に合わせた設定を管理するため。
- **理想状態**: 環境変数を使用して、アプリケーションに必要な設定を外部から提供します。

```yaml
env:
  - name: DATABASE_URL
    value: "postgres://user:password@localhost:5432/mydb"
  - name: APP_ENV
    value: "production"
```

---

### 9. アノテーション (annotations)
- **目的**: Kubernetesリソースにメタデータを追加して、動作のカスタマイズや管理のために利用するため。
- **理想状態**: IngressやService、Podなどにアノテーションを追加して、動作を調整します。

```yaml
service:
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-backend-protocol: "http"
```

---

### 10. リソースのリスタートポリシー (restartPolicy)
- **目的**: Podが終了した際の挙動を制御するため。
- **理想状態**: `restartPolicy: Always` を使用して、Podが終了すると自動的に再起動されるようにします。

```yaml
restartPolicy: Always
```

---

### 11. PVC (PersistentVolumeClaim)
- **目的**: 永続的なストレージを確保するため。
- **理想状態**: `persistentVolumeClaim` を使用し、データベースなどの永続的なデータを保存します。

```yaml
persistence:
  enabled: true
  size: 5Gi
  storageClass: standard
```

---

### 12. タグ付きのイメージ (image.tag)
- **目的**: デプロイするコンテナイメージのバージョンを管理するため。
- **理想状態**: イメージのタグを指定して、安定したバージョンを運用します。

```yaml
image:
  repository: myapp/repo
  tag: "v1.0.0"
```

---

### 13. ログレベル (logLevel)
- **目的**: アプリケーションのログ出力を制御するため。
- **理想状態**: ログの詳細度を調整することで、デバッグや運用中の情報収集を柔軟に行えます。

```yaml
env:
  - name: LOG_LEVEL
    value: "debug"
```

---

### 14. タイムアウト設定 (timeout)
- **目的**: アプリケーションやサービスのタイムアウトを管理するため。
- **理想状態**: タイムアウトを設定することで、レスポンスが遅延することなく、サービスが適切に終了するようにします。

```yaml
timeoutSeconds: 30
```
