# Kubernetes API Sample with Helm Chart

このプロジェクトは、Kubernetes上で動作するAPIサンプルアプリケーションをHelmチャートとしてデプロイするためのものです。

## 前提条件

- Kubernetesクラスタ（Kind、Minikube、EKS等）
- Helm 3.x
- kubectl
- Docker（イメージビルド用）

## プロジェクト構造

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

## 設定値（values.yaml）

主要な設定項目：

```yaml
replicaCount: 1  # レプリカ数

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

resources:
  requests:
    cpu: "100m"
    memory: "128Mi"
  limits:
    cpu: "200m"
    memory: "256Mi"

imagePullSecrets:
  - name: regcred

autoscaling:
  enabled: true
  minReplicas: 1
  maxReplicas: 3
  targetCPUUtilizationPercentage: 80
  targetMemoryUtilizationPercentage: 80
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