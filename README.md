# Cloud Run Demo
 
## 0. 前提条件
- Google Cloud Project が作成されていること
- 以降の手順は Cloud Shell 上での実行を想定しています

## 1. 事前準備
### 1-1. API の有効化
Cloud Run と Cloud Build の API を有効化します
```bash
gcloud services enable cloudbuild.googleapis.com \
  run.googleapis.com
```

## 2. サンプルアプリケーションのデプロイ
### 2-1. サンプルアプリケーションの準備
サンプルアプリケーションを Clone します
```bash
git clone https://github.com/kkuchima/run-demo
cd run-demo
```

### 2-2. ソースコードからのデプロイ（認証なしアクセスを許容）
`--source` を指定し、ソースコードから直接 Cloud Run へデプロイします  
また、 `--allow-unauthenticated` を指定することにより、認証なしのアクセスを許容します
```bash
gcloud beta run deploy demo-app --source . --platform managed --region asia-northeast1 --allow-unauthenticated
```

### 2-3. 動作確認
無事デプロイした後は出力された Service URL に curl でリクエストを投げ、 `Hello World!` と表示されることを確認します
```bash
export SERVICE-URL=<デプロイしたサービスの URL>
curl ${SERVICE-URL}
```

## 3. 認証付きアクセスの構成
### 3-1. 認証付きアクセス
` --no-allow-unauthenticated` を指定しアップデートすることで、当該サービスに対する認証なしアクセスを禁止します
```bash
gcloud beta run services update demo-app --platform managed --region asia-northeast1 --no-allow-unauthenticated
```

### 3-2. 動作確認
以下コマンドを実行し、認証なしアクセスが拒否されること、Authorization Header を付けるとアクセスが可能となることを確認します
```bash
# アクセスできない
curl ${SERVICE-URL}

# アクセスできる
curl ${SERVICE-URL} \
  -H "Authorization: bearer $(gcloud auth print-identity-token)"
```

## 4. アプリケーションの更新
### 4-1. web.py の編集
アクセスすると `Hello World! v2` と返すように web.py を編集します
```bash
sed -i 's/Hello World!/Hello World! v2\\n/' web.py
```

### 4-2. 更新したサンプルアプリケーションのデプロイ
更新したアプリケーションを `green` というタグを付け Cloud Run にデプロイします  
`--no-traffic` を指定し、新バージョン側にトラフィックが来ないよう設定します
```bash
gcloud beta run deploy demo-app --source . --platform managed --region asia-northeast1 --no-allow-unauthenticated --no-traffic --tag green
```

### 4-3. 新バージョンへの段階的移行
新バージョンへ段階的に移行していきます
```bash
# タグ付き URL にアクセスし、新バージョン単体での挙動を確認
curl green---${SERVICE-URL} \
  -H "Authorization: bearer $(gcloud auth print-identity-token)"

# 全体トラフィックのうち 30% を新バージョンへ流し、残り 70% を既存サービスへ流す
gcloud run services update-traffic demo-app --to-tags green=30 --platform managed --region asia-northeast1

# 複数回アクセスし、既存サービスと新バージョンのサービスにアクセスできていることを確認
curl ${SERVICE-URL} \
  -H "Authorization: bearer $(gcloud auth print-identity-token)"

# 全てのトラフィックを新バージョンへ流す
gcloud run services update-traffic demo-app --to-tags green=100 --platform managed --region asia-northeast1

# 複数回アクセスし、全て新バージョンのサービスにルーティングされていることを確認
curl ${SERVICE-URL} \
  -H "Authorization: bearer $(gcloud auth print-identity-token)"
```

## 5. 外部からのアクセスを制御する
### 5-1. GCLB もしくは同一プロジェクト内 VPC からのトラフィックのみ許可
`--ingress internal-and-cloud-load-balancing` を指定しアップデートすることで、外部からの直アクセスを禁止し、GCLB もしくは同一プロジェクト内 VPC からのトラフィックのみ許可するよう設定します
```bash
gcloud beta run services update demo-app --platform managed --region asia-northeast1 --ingress internal-and-cloud-load-balancing
```

### 5-2. 動作確認
Cloud Shell (外部) からのアクセスは拒否され、VPC 内クライアントからのアクセスが許容されることを確認します
```bash
# Cloud Shell (外部) からはアクセス不可
curl ${SERVICE-URL} \
  -H "Authorization: bearer $(gcloud auth print-identity-token)"

# 同一プロジェクト内 VPC 上の VM からはアクセス可能
gcloud compute instances create tokyo-client --zone asia-northeast1-a
gcloud compute ssh tokyo-client --zone asia-northeast1-a -- curl ${SERVICE-URL} -H "Authorization: bearer $(gcloud auth print-identity-token)"
```

## 6. Cloud Run のマルチリージョンデプロイ
### 6-1. GCLB の準備
マルチリージョンにデプロイした Cloud Run へのアクセスを捌く GCLB をデプロイします
```bash
export LB-PREFIX="run-demo"
gcloud compute backend-services create --global ${LB-PREFIX}-bs
gcloud compute url-maps create --default-service=${LB-PREFIX}-bs ${LB-PREFIX}-um
gcloud compute target-http-proxies create --url-map=${LB-PREFIX}-um ${LB-PREFIX}-tp
gcloud compute forwarding-rules create --target-http-proxy=${LB-PREFIX}-tp --global --ports=80 ${LB-PREFIX}-fr
```

### 6-2. 別リージョンへのデプロイ
web.py を編集し、`demo-app-osaka` として asia-northeast2 にデプロイします
```bash
# web.py の編集
sed -i 's/Hello World! v2/Hello World! v2 Osaka\\n/' web.py

# 編集したコードを asia-northeast2 にデプロイ
gcloud beta run deploy demo-app-osaka --source . --platform managed --region asia-northeast2 --no-allow-unauthenticated
```

### 6-3. Serverless NEG の作成
各 Cloud Run サービス用の NEG を作成します
```bash
# asia-northeast1 用 NEG
gcloud compute network-endpoint-groups create run-neg  --region=asia-northeast1 --network-endpoint-type=SERVERLESS --cloud-run-service=demo-app

# asia-northeast2 用 NEG
gcloud compute network-endpoint-groups create run-neg-osaka  --region=asia-northeast2 --network-endpoint-type=SERVERLESS --cloud-run-service=demo-app-osaka

```

### 6-4. NEG をバックエンドとして追加
作成した NEG を GCLB バックエンドとして追加します
```bash
gcloud compute backend-services add-backend ${LB-PREFIX}-bs --global --network-endpoint-group=run-neg --network-endpoint-group-region=asia-northeast1

gcloud compute backend-services add-backend ${LB-PREFIX}-bs --global --network-endpoint-group=run-neg-osaka --network-endpoint-group-region=asia-northeast2
```

### 6-5. 動作確認
`asia-northeast1` と `asia-northeast2` のクライアントからそれぞれ同一 IP アドレスにアクセスし、それぞれ地理的に近い Cloud Run サービスにルーティングされることを確認します
```bash
# サービスアクセス用 IP アドレスの取得
gcloud compute forwarding-rules describe ${LBNAME}-fr --global --format="value(IPAddress)"
export LBIP=<GCLB IP>

# asia-northeast1 からのアクセス確認
gcloud compute ssh tokyo-client --zone asia-northeast1-a -- curl ${LBIP} -H "Authorization: bearer $(gcloud auth print-identity-token)"

# asia-northeast2 からのアクセス確認
gcloud compute instances create osaka-client --zone asia-northeast2-a
gcloud compute ssh tokyo-client --zone asia-northeast2-a -- curl ${LBIP} -H "Authorization: bearer $(gcloud auth print-identity-token)"
```

以上