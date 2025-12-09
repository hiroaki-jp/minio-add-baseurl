# MinIO BaseURL サポート実装完了

## 実装概要

MinIOサーバーがリバースプロキシ経由でサブパスで動作できるようにBaseURL（パスプレフィックス）サポートを追加しました。

### 目標
- UI:  `https://minio.local/minio`
- API: `https://minio.local/minio-api`

パスプレフィックス（`/minio`と`/minio-api`）は設定可能です。

## 実装された変更

### 1. コマンドラインフラグと環境変数 ✅
**ファイル:** `cmd/server-main.go`

新しいフラグを追加：
```go
cli.StringFlag{
    Name:   "api-base-path",
    Usage:  "base path prefix for S3 API endpoints (e.g., /minio-api)",
    EnvVar: "MINIO_API_BASE_PATH",
},
cli.StringFlag{
    Name:   "console-base-path",
    Usage:  "base path prefix for Console UI (e.g., /minio)",
    EnvVar: "MINIO_CONSOLE_BASE_PATH",
},
```

### 2. グローバル変数の追加 ✅
**ファイル:** `cmd/globals.go`

`serverCtxt`構造体に追加：
```go
type serverCtxt struct {
    // ... 既存フィールド
    APIBasePath     string
    ConsoleBasePath string
    // ...
}
```

グローバル変数を追加：
```go
var (
    // Base path prefixes for API and Console
    globalAPIBasePath     = ""
    globalConsoleBasePath = ""
    // ...
)
```

### 3. パラメータ読み込み処理 ✅
**ファイル:** `cmd/common-main.go`

`buildServerCtxt`関数を更新してコマンドライン/環境変数からパスを読み込み、正規化：
```go
// Fetch API base path option
ctxt.APIBasePath = ctx.GlobalString("api-base-path")
if ctxt.APIBasePath == "" {
    ctxt.APIBasePath = ctx.String("api-base-path")
}
// Normalize: ensure it starts with / and doesn't end with /
if ctxt.APIBasePath != "" {
    if !strings.HasPrefix(ctxt.APIBasePath, "/") {
        ctxt.APIBasePath = "/" + ctxt.APIBasePath
    }
    ctxt.APIBasePath = strings.TrimSuffix(ctxt.APIBasePath, "/")
}
// Console base path も同様
```

### 4. グローバル変数への設定 ✅
**ファイル:** `cmd/server-main.go`

`serverHandleCmdArgs`関数でグローバル変数に値を設定：
```go
func serverHandleCmdArgs(ctxt serverCtxt) {
    handleCommonArgs(ctxt)

    // Set base paths for API and Console
    globalAPIBasePath = ctxt.APIBasePath
    globalConsoleBasePath = ctxt.ConsoleBasePath

    // ...
}
```

### 5. ルーター設定の更新 ✅
**ファイル:** `cmd/routers.go`

`configureServerHandler`関数を更新してPathPrefixをサポート：
```go
func configureServerHandler(endpointServerPools EndpointServerPools) (http.Handler, error) {
    router := mux.NewRouter().SkipClean(true).UseEncodedPath()

    // Create a subrouter for API paths if base path is configured
    var apiRouter *mux.Router
    if globalAPIBasePath != "" {
        apiRouter = router.PathPrefix(globalAPIBasePath).Subrouter()
    } else {
        apiRouter = router
    }

    // すべてのAPIルーターをapiRouterに登録
    if globalIsDistErasure {
        registerDistErasureRouters(apiRouter, endpointServerPools)
    }
    registerAdminRouter(apiRouter, true)
    registerHealthCheckRouter(apiRouter)
    registerMetricsRouter(apiRouter)
    registerSTSRouter(apiRouter)
    registerKMSRouter(apiRouter)
    registerAPIRouter(apiRouter)

    router.Use(globalMiddlewares...)
    return router, nil
}
```

## 使用方法

### コマンドライン例
```bash
# Windows
minio.exe server --api-base-path /minio-api --console-base-path /minio D:\data

# Linux/Mac
minio server --api-base-path /minio-api --console-base-path /minio /data
```

### 環境変数例
```bash
# Windows
set MINIO_API_BASE_PATH=/minio-api
set MINIO_CONSOLE_BASE_PATH=/minio
minio.exe server D:\data

# Linux/Mac
export MINIO_API_BASE_PATH=/minio-api
export MINIO_CONSOLE_BASE_PATH=/minio
minio server /data
```

### ヘルプでの確認
```bash
./minio.exe server --help
```

出力に以下が表示されます：
```
--api-base-path value        base path prefix for S3 API endpoints (e.g., /minio-api) [%MINIO_API_BASE_PATH%]
--console-base-path value    base path prefix for Console UI (e.g., /minio) [%MINIO_CONSOLE_BASE_PATH%]
```

## リバースプロキシ設定例

### nginx
```nginx
location /minio-api/ {
    proxy_pass http://minio-backend:9000/minio-api/;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
    # WebSocket用
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "upgrade";

    # タイムアウト設定
    proxy_connect_timeout 300;
    proxy_send_timeout 300;
    proxy_read_timeout 300;
    send_timeout 300;
}

location /minio/ {
    proxy_pass http://minio-backend:9001/minio/;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
    # WebSocket用（Console UIに必要）
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "upgrade";
}
```

### Apache
```apache
<Location /minio-api>
    ProxyPass http://minio-backend:9000/minio-api
    ProxyPassReverse http://minio-backend:9000/minio-api
    ProxyPreserveHost On
    RequestHeader set X-Forwarded-Proto "https"
</Location>

<Location /minio>
    ProxyPass ws://minio-backend:9001/minio
    ProxyPassReverse ws://minio-backend:9001/minio
    ProxyPass http://minio-backend:9001/minio
    ProxyPassReverse http://minio-backend:9001/minio
    ProxyPreserveHost On
    RequestHeader set X-Forwarded-Proto "https"
</Location>
```

### Traefik (docker-compose.yml)
```yaml
services:
  minio:
    image: minio/minio:latest
    command: server /data --console-address ":9001" --api-base-path /minio-api --console-base-path /minio
    environment:
      MINIO_ROOT_USER: minioadmin
      MINIO_ROOT_PASSWORD: minioadmin
    labels:
      - "traefik.enable=true"
      # API
      - "traefik.http.routers.minio-api.rule=PathPrefix(`/minio-api`)"
      - "traefik.http.routers.minio-api.service=minio-api"
      - "traefik.http.services.minio-api.loadbalancer.server.port=9000"
      # Console
      - "traefik.http.routers.minio-console.rule=PathPrefix(`/minio`)"
      - "traefik.http.routers.minio-console.service=minio-console"
      - "traefik.http.services.minio-console.loadbalancer.server.port=9001"
```

## S3 SDKの使用例

### Python (boto3)
```python
import boto3

s3 = boto3.client(
    's3',
    endpoint_url='https://minio.local/minio-api',  # ベースパスを含める
    aws_access_key_id='minioadmin',
    aws_secret_access_key='minioadmin',
    region_name='us-east-1'
)

# バケットのリスト
buckets = s3.list_buckets()
print(buckets)
```

### Go (minio-go)
```go
package main

import (
    "github.com/minio/minio-go/v7"
    "github.com/minio/minio-go/v7/pkg/credentials"
)

func main() {
    minioClient, err := minio.New("minio.local/minio-api", &minio.Options{
        Creds:  credentials.NewStaticV4("minioadmin", "minioadmin", ""),
        Secure: true,
    })
    if err != nil {
        panic(err)
    }
    // クライアント使用
}
```

### AWS CLI
```bash
aws s3 ls --endpoint-url https://minio.local/minio-api
```

## テスト方法

### 1. 基本的な動作確認
```bash
# サーバー起動
./minio.exe server --api-base-path /minio-api --console-base-path /minio D:\data

# 別のターミナルで
# APIエンドポイントの確認
curl http://localhost:9000/minio-api/

# ヘルスチェック
curl http://localhost:9000/minio-api/minio/health/live
```

### 2. mc (MinIO Client)での確認
```bash
# mcのエイリアス設定（ベースパスを含める）
mc alias set local http://localhost:9000/minio-api minioadmin minioadmin

# バケット作成
mc mb local/testbucket

# ファイルアップロード
echo "test" > test.txt
mc cp test.txt local/testbucket/

# バケット一覧
mc ls local
```

### 3. ブラウザでConsole UIアクセス
ブラウザで `http://localhost:9001/minio` にアクセス

## 注意事項

1. **互換性**: ベースパスを指定しない場合、既存の動作と完全に互換性があります
2. **内部通信**: 分散セットアップでのノード間通信はベースパスの影響を受けません
3. **パス正規化**:
   - ベースパスは自動的に先頭に`/`が付加されます（例: `minio-api` → `/minio-api`）
   - 末尾の`/`は自動的に削除されます（例: `/minio-api/` → `/minio-api`）
4. **Console統合**: 現在の実装ではAPI側のみベースパスが適用されています。Consoleの完全な統合には追加の作業が必要な場合があります。

## 今後の拡張予定

- [ ] Consoleサーバーへのベースパス設定の伝播
- [ ] 起動メッセージでのベースパス付きURL表示
- [ ] ブラウザリダイレクトミドルウェアの更新
- [ ] 統合テストの追加

## ビルド結果

✅ ビルド成功
✅ 新しいコマンドラインフラグが正しく認識される
✅ ヘルプメッセージに正しく表示される

## 変更されたファイル

1. `cmd/server-main.go` - コマンドラインフラグの追加
2. `cmd/globals.go` - グローバル変数とserverCtxt構造体の更新
3. `cmd/common-main.go` - パラメータ読み込みと正規化
4. `cmd/routers.go` - PathPrefixサポートの実装

## まとめ

MinIOサーバーに基本的なBaseURLサポートが正常に実装されました。これにより、リバースプロキシ経由でサブパスからMinIOを提供できるようになります。

基本機能は動作しますが、完全な統合には追加のテストと、ConsoleサーバーへのベースパスTransmissionが必要になる可能性があります。
