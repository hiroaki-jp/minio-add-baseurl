# CLAUDE.md

このファイルは、Claude Code (claude.ai/code) がこのリポジトリで作業する際のガイダンスを提供します。

## プロジェクトの状態

このプロジェクトは**メンテナンスモード**です。新機能、機能強化、プルリクエストは受け付けていません。重大なセキュリティ修正のみがケースバイケースで評価される可能性があります。

## ビルドコマンド

### ソースからのビルド

```bash
# バイナリのビルド
make build

# レース検出付きビルド（テスト用）
make install-race

# $GOPATH/binにインストール
make install

# 異なるプラットフォーム向けのクロスコンパイル
make crosscompile
```

### テスト

```bash
# 全テストを検証とビルドとともに実行
make test

# レース検出付きでテストを実行
make test-race

# ユニットテストのみを実行（完全な検証なし）
MINIO_API_REQUESTS_MAX=10000 CGO_ENABLED=0 go test -v -tags kqueue,dev ./...

# 特定のテストパッケージを実行
go test -v -tags kqueue,dev ./cmd/...

# 単一のテストを実行
go test -v -tags kqueue,dev -run TestName ./cmd
```

### リントとコード品質

```bash
# すべての検証を実行（リントとコード生成チェックを含む）
make verifiers

# リントのみを実行
make lint

# 自動修正付きでリントを実行
make lint-fix

# コミットされていない生成コードをチェック
make check-gen
```

### Docker

```bash
# Dockerイメージのビルド（minioバイナリが存在する必要があります）
docker build -t myminio:minio .

# コンテナの実行
docker run -p 9000:9000 -p 9001:9001 myminio:minio server /tmp/minio --console-address :9001
```

## 高レベルアーキテクチャ

### コアコンポーネント

MinIOは、以下のアーキテクチャレイヤーを持つ高性能なS3互換オブジェクトストレージシステムです：

#### 1. エントリーポイントとメインパッケージ
- `main.go`: 最小限のエントリーポイント。`internal/init`をインポート（必須：最初にインポート）し、`cmd.Main()`を呼び出す
- `cmd/`パッケージ: すべてのサーバーロジックを含む（約25万行のGoコード）

#### 2. オブジェクトレイヤーアーキテクチャ
- **ObjectLayerインターフェース** (`cmd/object-api-interface.go`): すべてのオブジェクト操作の中心的な抽象化
- **Erasure Objects** (`cmd/erasure.go`): Reed-Solomonイレージャーコーディングを使用したメイン実装
- **XL Storage** (`cmd/xl-storage.go`): xl.metaファイルを使用した低レベルディスクストレージレイヤー

#### 3. イレージャーコーディングシステム
- Reed-Solomonイレージャーコーディングを使用（最大256シャード：データ128 + パリティ128）
- イレージャーセット: セットあたり2〜16ドライブ、GCDアルゴリズムで自動計算
- デフォルト設定: N/2データブロックとN/2パリティブロック
- オブジェクトは一貫性ハッシュを使用してオブジェクトレベル（ボリュームレベルではない）でシャーディング
- 書き込み/読み取りクォーラムはイレージャーセット内でのみ必要
- オブジェクト配置は`sipHashMod(objectName)`で決定され、一貫性のある分散を実現

#### 4. サーバープールと分散アーキテクチャ
- 独立したイレージャーセットを持つ複数のサーバープールをサポート
- 重み付けランダム選択による比例的な空き容量に基づいたプール選択
- 各プールは独自のSLAとパリティカウントを維持
- 拡張: データ移行なしで新しいプールを追加可能
- 分散: 可用性を最大化するためにノード全体にドライブを均等に分散

#### 5. 主要サブシステム

**HTTPハンドラー** (`cmd/*-handlers.go`):
- `object-handlers.go`: S3 API操作（オブジェクトのGET/PUT/DELETE）
- `admin-handlers.go`: 設定と管理のための管理API
- `bucket-handlers.go`: バケットレベルの操作
- `api-router.go`: リクエストルーティングとミドルウェア

**ストレージバックエンド** (`cmd/xl-storage.go`):
- アラインドバッファによるDirect I/O最適化
- 小さいファイル（<128KB）: メタデータがデータと同梱
- 大きいファイル（>128MB）: 先読み最適化
- オブジェクトメタデータに`xl.meta`を使用、バックアップに`xl.meta.bkp`を使用

**レプリケーション** (`cmd/bucket-replication*.go`):
- マルチサイトレプリケーションサポート
- 自動ヒーリング付きサイト間レプリケーション
- バッチレプリケーションジョブ

**IAMと認証** (`cmd/admin-handlers-idp-*.go`):
- LDAP統合
- OpenID Connectサポート
- カスタムIDマネジメントプラグイン

**ヒーリングとバックグラウンド操作** (`cmd/background-heal-ops.go`):
- 破損/欠落データの自動ヒーリング
- バックグラウンドディスクスキャン
- 不整合バージョンの解決

#### 6. インターナルパッケージ（約35パッケージ）

主要なインターナルパッケージ:
- `internal/auth`: 認証と認証情報
- `internal/crypto`: 暗号化（SSE-C、SSE-S3、SSE-KMS）
- `internal/dsync`: 分散ロック
- `internal/event`: イベント通知システム
- `internal/grid`: 内部RPC通信レイヤー
- `internal/hash`: チェックサムと整合性（ビットロット保護のためのHighwayHash）
- `internal/config`: サーバー設定管理
- `internal/logger`: 構造化ロギング

### コード構成パターン

- **ハンドラーパターン**: `cmd/*-handlers.go`のHTTPハンドラーがObjectLayerメソッドを呼び出す
- **ミドルウェアチェーン**: 認証、暗号化、ロギングレイヤーを通じたリクエスト処理
- **生成コード**: `_gen.go`で終わるファイルは自動生成（シリアライゼーションにmsgpを使用）
- **テストファイル**: cmd/に143のテストファイルがあり、ユニットテストと統合テストをカバー
- **ビルドタグ**: 開発ビルドには`-tags kqueue,dev`を使用

### データフロー

1. HTTPリクエスト → ルーター (`api-router.go`)
2. 認証ミドルウェア (`auth-handler.go`)
3. ハンドラー関数 (`*-handlers.go`)
4. ObjectLayerインターフェースメソッド
5. Erasure Objects実装
6. XL Storageディスク操作
7. S3互換のXML/JSONでレスポンス

## 開発要件

- **Goバージョン**: 最小Go 1.24（READMEで指定）
- **ビルドタグ**: 開発時は常に`-tags kqueue,dev`を使用
- **レース検出**: レーステストには`CGO_ENABLED=1`と`-race`フラグを使用
- **コード生成**: msgpタグ付き構造体を変更した後は`go generate ./...`を実行
- **依存関係**: `go mod`で管理、`go mod tidy -compat=1.21`を使用

## テストパターン

### ユニットテスト
- ソースファイルと並べて`*_test.go`として配置
- `kqueue,dev`ビルドタグを使用
- レート制限テストには`MINIO_API_REQUESTS_MAX=10000`を設定

### 統合テスト
- `buildscripts/`と`docs/*/`ディレクトリのシェルスクリプト
- 例: `test-replication`、`test-versioning`、`test-iam`、`test-healing`
- レース検出には`install-race`ビルドが必要

### テストスクリプト
- IAMテスト: `make test-iam`（15分タイムアウト）
- レプリケーション: `make test-replication`（マルチサイトシナリオ）
- ヒーリング: `make verify-healing`（ディスク障害シミュレーション）
- デコミッション: `make test-decom`（プール削除テスト）

## コードスタイル

- [Effective Go](https://github.com/golang/go/wiki/CodeReviewComments)に従う
- `.golangci.yml`の設定でgolangci-lintにより強制
- `gofumpt`と`goimports`による自動フォーマット
- 有効なリンター: gocritic、govet、staticcheck、revive、misspell、ineffassign、unused

## 一般的なワークフロー

### 新しいS3 APIの追加
1. `cmd/object-handlers.go`または新しいファイルにハンドラーを追加
2. `cmd/api-router.go`にルートを登録
3. 必要に応じてObjectLayerインターフェースメソッドを実装
4. 対応するテストを追加
5. `make verifiers`と`make test`を実行

### イレージャーコードロジックの変更
1. 変更は通常`cmd/erasure*.go`ファイル内
2. クォーラム要件（読み取り/書き込み）を理解する
3. データ整合性のために`make verify-healing`でテスト
4. 既存のデプロイメントとの後方互換性を考慮

### ストレージレイヤーの作業
1. XL Storageインターフェースは`cmd/xl-storage.go`にあり
2. メタデータ形式: `xl.meta` JSONファイル
3. Direct I/Oには`disk.AlignedBlock()`を使用
4. ディスク障害シナリオでテスト

## 重要な注意事項

- **メンテナンスモード**: このコードベースは新機能を受け付けていません
- **ライセンス**: GNU AGPLv3 - 使用時はコンプライアンスを確保してください
- **テスト**: ストレージ/イレージャーコードの変更には常に完全なテストスイートを実行
- **生成ファイル**: `*_gen.go`ファイルを直接編集しないでください。`go generate`で再生成してください
- **ビルドタグ**: `kqueue`タグは一部の機能に必要です。開発ビルドには常に含めてください
