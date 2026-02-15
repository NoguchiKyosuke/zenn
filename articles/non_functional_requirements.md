# SIRISA 非機能要求定義書

> **最終更新日**: 2026-02-15  
> **対象システム**: SIRISA（AI搭載Q&Aプラットフォーム for 中学生向け学習支援）  
> **ドキュメントバージョン**: 1.0

---

## 目次

1. [可用性](#1-可用性)
2. [性能・拡張性](#2-性能拡張性)
3. [セキュリティ](#3-セキュリティ)
4. [信頼性・耐障害性](#4-信頼性耐障害性)
5. [運用・保守性](#5-運用保守性)
6. [インフラストラクチャ](#6-インフラストラクチャ)
7. [データ管理](#7-データ管理)
8. [外部連携](#8-外部連携)
9. [ログ・監査](#9-ログ監査)
10. [国際化・ローカライゼーション](#10-国際化ローカライゼーション)
11. [今後の改善候補](#11-今後の改善候補)

---

## 1. 可用性

### 1.1 稼働時間

| 項目 | 現状 |
|:--|:--|
| 稼働形態 | GCE単一インスタンス（24時間365日稼働） |
| 冗長構成 | なし（単一サーバ、ロードバランサなし） |
| 自動再起動 | systemdによる `Restart=on-failure`（Gunicorn: 5秒後、Celery: 10秒後） |
| SSL証明書更新 | Let's Encrypt + certbot systemdタイマーによる自動更新 |

### 1.2 サービス依存関係

| サービス | 起動順序 | 用途 |
|:--|:--|:--|
| PostgreSQL 16 | 1番目 | データベース |
| Redis 7.0 | 2番目 | Celeryブローカー / 結果バックエンド |
| Gunicorn | 3番目 | WSGIアプリケーションサーバ（3ワーカー） |
| Celery | 4番目 | 非同期タスクワーカー（並行度2） |
| nginx 1.24 | 最前段 | リバースプロキシ / 静的ファイル配信 |

---

## 2. 性能・拡張性

### 2.1 サーバリソース

| リソース | スペック |
|:--|:--|
| マシンタイプ | GCE `e2-standard-2`（2 vCPU / 8 GB RAM） |
| CPU | Intel Xeon 2.20GHz（1物理コア / 2論理コア） |
| メモリ | 7.8 GB（スワップ 2.0 GB） |
| ディスク | 29 GB（使用率約31%） |
| OS | Ubuntu 24.04 LTS |

### 2.2 アプリケーション性能

| 項目 | 設定値 |
|:--|:--|
| Gunicornワーカー数 | 3 |
| Gunicornタイムアウト | 120秒 |
| Celery並行度 | 2 |
| nginxプロキシ接続タイムアウト | 300秒 |
| nginxプロキシ読み取りタイムアウト | 300秒 |
| 最大アップロードサイズ（メインサイト） | 100 MB |
| 最大アップロードサイズ（コンテンツサブドメイン） | 10 MB |
| Django FILE_UPLOAD_MAX_MEMORY_SIZE | 110 MB |

### 2.3 AI応答性能

| パラメータ | AI回答生成 | AI返信生成 | 注釈生成 | 補足生成 |
|:--|:--|:--|:--|:--|
| モデル | Gemini 2.5 Pro | Gemini 2.5 Flash | Gemini 2.5 Flash | Gemini 2.5 Flash |
| temperature | 0.7 | 0.7 | 0.5 | 0.3 |
| max_output_tokens | 65,536 | 2,048 | 1,024 | 2,048 |
| リトライ回数 | 最大3回（10秒間隔） | 最大2回（5秒間隔） | — | — |
| 利用制限 | 100回/日/ユーザー | 同左（共通カウント） | — | — |
| マルチモーダル | 画像・音声・動画対応 | テキストのみ | テキストのみ | テキストのみ |

### 2.4 キャッシュ

| 項目 | 設定 |
|:--|:--|
| 静的ファイル (nginx) | `expires 30d` / `Cache-Control: public, immutable` |
| メディアファイル (nginx) | `expires 7d` |
| Djangoキャッシュフレームワーク | 未構成（デフォルトローカルメモリ） |
| セッションバックエンド | データベース（30日有効 / リクエスト毎保存） |

### 2.5 データベース設定

| パラメータ | 値 |
|:--|:--|
| PostgreSQL バージョン | 16.11 |
| max_connections | 100 |
| shared_buffers | 128 MB |
| 認証方式 | ローカル: peer / TCP(localhost): scram-sha-256 |
| リモートアクセス | 不可（localhostのみ） |
| レプリケーション | 未構成 |
| デフォルトPK | BigAutoField |

---

## 3. セキュリティ

### 3.1 認証・認可

| 項目 | 実装 |
|:--|:--|
| 認証基盤 | Firebase Authentication |
| 認証方式 | Googleサインイン（リダイクト方式） / メールリンク認証 |
| クライアントSDK | Firebase Web SDK v10（compat） |
| サーバサイド検証 | Firebase Admin SDK `verify_id_token()` |
| authDomain | `sirisa.net`（nginx経由でFirebaseにプロキシ） |
| カスタム認証バックエンド | `FirebaseAuthBackend`（プライマリ） + `ModelBackend`（フォールバック） |
| ユーザーモデル | カスタム `accounts.User` |

### 3.2 通信セキュリティ

| 項目 | 設定 |
|:--|:--|
| SSL/TLS | Let's Encrypt証明書 |
| TLSバージョン | 1.2 / 1.3 のみ |
| 暗号スイート | ECDHE + CHACHA20-POLY1305 / AES-GCM |
| DHパラメータ | `/etc/letsencrypt/ssl-dhparams.pem` |
| SSLセッション | 共有キャッシュ10MB / タイムアウト1440分 / セッションチケット無効 |
| HSTS | `max-age=63072000; includeSubDomains; preload` |
| HTTP→HTTPS | 強制リダイレクト（nginx + Django `SECURE_SSL_REDIRECT`） |

### 3.3 Webセキュリティヘッダ

| ヘッダ | 値 | 設定元 |
|:--|:--|:--|
| `X-Content-Type-Options` | `nosniff` | nginx + Django |
| `X-Frame-Options` | `DENY`（メインサイト） | nginx + Django |
| `X-XSS-Protection` | `1; mode=block` | nginx |
| `Strict-Transport-Security` | `max-age=63072000; includeSubDomains; preload` | nginx |
| `X-Forwarded-Proto` | プロキシヘッダ設定 | Django `SECURE_PROXY_SSL_HEADER` |

### 3.4 CSRF / Cookie

| 項目 | 設定 |
|:--|:--|
| CSRF検証 | Django `CsrfViewMiddleware` |
| CSRF信頼オリジン | `sirisa.net`, `www.sirisa.net`, `content.sirisa.net` |
| SESSION_COOKIE_SECURE | `True` |
| CSRF_COOKIE_SECURE | `True` |

### 3.5 機密情報管理

| 項目 | 方式 |
|:--|:--|
| シークレット管理 | `.env`ファイル + `python-dotenv` |
| `.env`ファイル配置 | `/opt/sirisa/.env` |
| Gitignore | `.env`, `credentials/` を除外 |
| GCPクレデンシャル | GCEデフォルトサービスアカウント（Vertex AI用） |

**管理される環境変数（17項目）:**

| 変数名 | 用途 |
|:--|:--|
| `SECRET_KEY` | Djangoシークレットキー |
| `DEBUG` | デバッグモード |
| `ALLOWED_HOSTS` | 許可ホスト |
| `DB_NAME` / `DB_USER` / `DB_PASSWORD` / `DB_HOST` / `DB_PORT` | PostgreSQL接続 |
| `GCLOUD_API_KEY` | Google Safe Browsing API |
| `EMAIL_HOST_USER` / `EMAIL_HOST_PASSWORD` | Gmail SMTP認証 |
| `GMAIL_SENDER_EMAIL` | メール送信元 |
| `CELERY_BROKER_URL` / `CELERY_RESULT_BACKEND` | Redis接続 |
| `FIREBASE_PROJECT_ID` / `FIREBASE_API_KEY` / `FIREBASE_AUTH_DOMAIN` | Firebase認証 |

### 3.6 レート制限

| レイヤー | 制限 |
|:--|:--|
| nginx | 未構成 |
| アプリケーション（AI API） | 100回/日/ユーザー（`AIUsageLog`モデルで `F()` アトミックインクリメント） |
| アプリケーション（通報） | 同一対象に対し1通報/24時間/ユーザー |

### 3.7 コンテンツセキュリティ

| 対策 | 内容 |
|:--|:--|
| HTMLサニタイズ | `bleach`ライブラリによるAI生成HTML浄化 |
| Shadow DOM隔離 | AI回答をShadow DOM内でレンダリング（CSS/JS分離） |
| サンドボックスドメイン | `content.sirisa.net` で別CSP適用 |
| 外部リンク安全検査 | Google Safe Browsing API v4（脅威4種検出、タイムアウト5秒 fail-open） |
| onclick変換 | Shadow DOM内で `onclick` 属性を `addEventListener` に自動変換 |
| 関数エクスポート | IIFE内の関数宣言を `window` にエクスポート（Shadow DOM onclick対応） |

---

## 4. 信頼性・耐障害性

### 4.1 プロセス管理

| プロセス | 復旧方式 | 復旧間隔 |
|:--|:--|:--|
| Gunicorn | `Restart=on-failure` (systemd) | 5秒 |
| Celery | `Restart=on-failure` (systemd) | 10秒 |
| nginx | systemd管理 | OS標準 |
| PostgreSQL | systemd管理 | OS標準 |
| Redis | systemd管理 | OS標準 |

### 4.2 AI生成タスクの耐障害性

| 項目 | 設定 |
|:--|:--|
| 回答生成リトライ | 最大3回 / 10秒間隔 |
| 返信生成リトライ | 最大2回 / 5秒間隔 |
| ステータス管理 | `pending` → `completed` / `failed`（画面即時反映用） |
| 2バリアント生成 | 通常回答 + スライド回答を並行生成 |
| Celeryシリアライズ | JSON only（安全なデシリアライズ） |

### 4.3 データ保全

| 項目 | 方式 |
|:--|:--|
| 論理削除 | 全主要モデルで `SoftDeleteMixin` + `SoftDeleteManager` |
| 物理削除 | 禁止（ソフトデリートのみ） |
| 削除監査 | `DeletionLog` モデルで削除者・日時・理由を記録 |
| メディアファイル | `django-cleanup` による孤立ファイル自動削除 |

---

## 5. 運用・保守性

### 5.1 デプロイ構成

| 項目 | 内容 |
|:--|:--|
| デプロイ方式 | GCE直接デプロイ（コンテナ化なし） |
| プロセス管理 | systemd（`sirisa.service`, `sirisa-celery.service`） |
| 実行ユーザー | `nk`（グループ: `sirisa`） |
| 作業ディレクトリ | `/opt/sirisa/src` |
| 仮想環境 | `/opt/sirisa/venv`（Python 3.12） |
| 設定ファイル分割 | `settings/base.py` / `development.py` / `production.py` |
| ソースコード管理 | Git → GitHub（`NoguchiKyosuke/sirisa`, public） |

### 5.2 静的ファイル管理

| 項目 | 設定 |
|:--|:--|
| ソース | `/opt/sirisa/src/static/` + 各アプリ内staticディレクトリ |
| 収集先 | `/opt/sirisa/staticfiles/`（`collectstatic`） |
| nginx配信 | `/static/` → staticfiles（30日キャッシュ） |
| CDN | 未使用（GCEインスタンスから直接配信） |

### 5.3 メディアファイル管理

| 項目 | 設定 |
|:--|:--|
| ルート | `/opt/sirisa/media/` |
| ディレクトリ構成 | `questions/{pk}/`, `answers/{pk}/`, `replies/{pk}/` |
| nginx配信 | `/media/` → media（7日キャッシュ） |
| 孤立ファイル | `django-cleanup` で自動削除 |

### 5.4 メール配信

| 項目 | 設定 |
|:--|:--|
| バックエンド | Django SMTPEmailBackend |
| SMTPサーバ | Gmail（`smtp.gmail.com:587` TLS） |
| 用途 | Firebase メールリンク認証 / 通知 |

---

## 6. インフラストラクチャ

### 6.1 クラウド構成

```
┌─────────────────────────────────────────────────────────┐
│  Google Cloud Platform (Project: sirisa)                │
│  Region: us-central1 / Zone: us-central1-c              │
│                                                         │
│  ┌───────────────────────────────────────────────────┐  │
│  │  GCE Instance: e2-standard-2                      │  │
│  │  Static IP: 34.28.41.172                          │  │
│  │  OS: Ubuntu 24.04 LTS                             │  │
│  │                                                   │  │
│  │  ┌─────────┐  ┌──────────┐  ┌─────────────────┐  │  │
│  │  │  nginx   │→│ Gunicorn │→│ Django (WSGI)   │  │  │
│  │  │ :80/:443 │  │ (unix    │  │ Python 3.12     │  │  │
│  │  │          │  │  socket) │  │ Django 4.2 LTS  │  │  │
│  │  └─────────┘  └──────────┘  └─────────────────┘  │  │
│  │                               ↕          ↕        │  │
│  │                    ┌──────────┐  ┌────────────┐   │  │
│  │                    │PostgreSQL│  │   Redis    │   │  │
│  │                    │  16.11   │  │   7.0.15   │   │  │
│  │                    └──────────┘  └────────────┘   │  │
│  │                                       ↕           │  │
│  │                               ┌────────────┐      │  │
│  │                               │  Celery    │      │  │
│  │                               │  Worker    │      │  │
│  │                               └────────────┘      │  │
│  └───────────────────────────────────────────────────┘  │
│                                                         │
│  External: Vertex AI (Gemini), Firebase Auth,           │
│            Safe Browsing API, Gmail SMTP                │
└─────────────────────────────────────────────────────────┘
```

### 6.2 ドメイン構成

| ドメイン | 用途 | 備考 |
|:--|:--|:--|
| `sirisa.net` | メインサイト | `X-Frame-Options: DENY` |
| `content.sirisa.net` | AI生成コンテンツ（iframe用） | `X-Frame-Options` なし（iframe許可） |

### 6.3 nginx設定概要

| 項目 | メインサイト | コンテンツサブドメイン |
|:--|:--|:--|
| アップストリーム | unix socket | 同左 |
| `client_max_body_size` | 100 MB | 10 MB |
| プロキシタイムアウト | 300秒 | 300秒 |
| 静的ファイル配信 | ○（30日） | × |
| メディアファイル配信 | ○（7日） | × |
| Firebaseプロキシ | ○（`/__/auth/`, `/__/firebase/`） | × |
| HSTS | ○ | ○ |

---

## 7. データ管理

### 7.1 データモデル方針

| 方針 | 実装 |
|:--|:--|
| 主キー | `BigAutoField` (64bit) |
| 論理削除 | `SoftDeleteMixin`（`is_deleted`, `deleted_at`, `deleted_by`） |
| タイムスタンプ | `created_at`, `updated_at` 自動付与 |
| 削除監査 | `DeletionLog` による操作記録 |

### 7.2 バックアップ

| 項目 | 状態 |
|:--|:--|
| データベースバックアップ | 未構成（要件として認識） |
| メディアファイルバックアップ | 未構成 |
| GCPスナップショット | 未確認 |

### 7.3 Redis設定

| パラメータ | 値 |
|:--|:--|
| バージョン | 7.0.15 |
| 動作モード | スタンドアロン |
| maxmemory | 無制限 |
| maxmemory-policy | noeviction |
| 用途 | Celeryブローカー + 結果バックエンド（DB 0共用） |

---

## 8. 外部連携

### 8.1 Google Cloud サービス

| サービス | 用途 | 認証方式 |
|:--|:--|:--|
| Vertex AI (Gemini 2.5 Pro) | AI回答生成 | GCEデフォルトサービスアカウント |
| Vertex AI (Gemini 2.5 Flash) | AI返信 / 注釈 / 補足生成 | 同上 |
| Safe Browsing API v4 | 外部リンク安全性検査 | APIキー（環境変数） |
| Compute Engine | ホスティング | — |

### 8.2 Firebase

| コンポーネント | 用途 | バージョン |
|:--|:--|:--|
| Firebase Web SDK (Client) | 認証UI・トークン取得 | v10 compat |
| Firebase Admin SDK (Server) | トークン検証・ユーザー管理 | ≥7.0.0 |
| Firebase Project | `sirisa-f5a1f` | — |

### 8.3 外部CDN依存

| ライブラリ | 用途 | 配信元 |
|:--|:--|:--|
| KaTeX | 数式レンダリング | jsDelivr |
| Mermaid.js | フローチャート・ダイアグラム | jsDelivr |
| Chart.js | グラフ描画 | jsDelivr |
| Bootstrap 5.3 | UIフレームワーク | CDN |

---

## 9. ログ・監査

### 9.1 ログ体系

| ログ種別 | ファイルパス | レベル | 内容 |
|:--|:--|:--|:--|
| アプリケーション | `/opt/sirisa/logs/django.log` | INFO | Django全般 |
| 削除監査 | `/opt/sirisa/logs/deletion.log` | INFO | 論理削除操作の記録 |
| Celeryタスク | `/opt/sirisa/logs/celery.log` | INFO | 非同期タスク実行状況 |
| Gunicornアクセス | `/opt/sirisa/logs/gunicorn_access.log` | — | HTTPリクエスト |
| Gunicornエラー | `/opt/sirisa/logs/gunicorn_error.log` | — | Gunicornエラー |
| nginxアクセス | `/var/log/nginx/sirisa_access.log` | — | リバースプロキシアクセス |
| nginxエラー | `/var/log/nginx/sirisa_error.log` | — | nginxエラー |
| nginx(content) | `/var/log/nginx/sirisa_content_*.log` | — | コンテンツサブドメイン |

### 9.2 アプリケーション監査

| 対象 | 方式 |
|:--|:--|
| データ削除 | `DeletionLog` モデル（削除者・日時・理由・復元可否） |
| AI利用量 | `AIUsageLog` モデル（ユーザー毎の日次API呼び出し数） |
| ユーザー通報 | `Report` モデル（通報者・対象・カテゴリ・日時） |

### 9.3 外部監視

| 項目 | 状態 |
|:--|:--|
| APM / エラートラッキング | 未導入（Sentry等なし） |
| インフラ監視 | 未導入（Prometheus / Datadog等なし） |
| アラート通知 | 未構成 |
| ヘルスチェック | 未構成 |

---

## 10. 国際化・ローカライゼーション

| 項目 | 設定 |
|:--|:--|
| デフォルト言語 | 日本語 (`ja`) |
| タイムゾーン | `Asia/Tokyo` |
| USE_TZ | `True`（UTC保存 / 表示時変換） |
| 多言語対応 | 未実装（日本語のみ） |

---

## 11. 今後の改善候補

以下は現状の構成から識別される改善ポイントです。

### 高優先度

| # | 項目 | 現状 | 推奨 |
|:--|:--|:--|:--|
| 1 | データベースバックアップ | 未構成 | pg_dump定期実行 + GCSバケット保存 |
| 2 | 外部監視・アラート | 未導入 | Cloud Monitoring / Sentry導入 |
| 3 | ヘルスチェックエンドポイント | 未構成 | `/health/` エンドポイント追加 |

### 中優先度

| # | 項目 | 現状 | 推奨 |
|:--|:--|:--|:--|
| 4 | nginxレート制限 | 未構成 | `limit_req_zone` によるリクエスト制限 |
| 5 | Djangoキャッシュ | 未構成 | Redis利用のページ / フラグメントキャッシュ |
| 6 | Redis maxmemory | 無制限 | 適切な上限設定（例: 512MB） |
| 7 | PostgreSQL shared_buffers | 128MB（デフォルト） | RAM の 25%（約2GB）に調整 |
| 8 | CSRF_COOKIE_HTTPONLY | 未設定 | `True` に設定 |

### 低優先度

| # | 項目 | 現状 | 推奨 |
|:--|:--|:--|:--|
| 9 | CDN導入 | 未使用 | Cloud CDNまたはCloudflare |
| 10 | CeleryブローカーDB分離 | ブローカー / 結果が同一Redis DB 0 | 別DBに分離（DB 0 / DB 1） |
| 11 | コンテナ化 | なし | Docker + Cloud Run移行検討 |
| 12 | ログローテーション | 手動 | logrotate設定追加 |

---

## 付録: 主要依存ライブラリ

| カテゴリ | パッケージ | バージョン |
|:--|:--|:--|
| フレームワーク | Django | 4.2.28 |
| WSGI | gunicorn | 25.0.3 |
| タスクキュー | celery | 5.6.2 |
| ブローカー | redis | 7.1.0 |
| データベース | psycopg2-binary | 2.9.11 |
| AI (Vertex) | google-cloud-aiplatform | 1.137.0 |
| AI (GenAI) | google-genai | 1.63.0 |
| 認証 | firebase-admin | ≥7.0.0 |
| HTMLサニタイズ | bleach | 6.3.0 |
| PDF出力 | weasyprint | 68.1 |
| Excel出力 | openpyxl | 3.1.5 |
| Markdown | Markdown | 3.10.1 |
| 画像処理 | pillow | 12.1.0 |
| htmx | django-htmx | 1.27.0 |
| ファイル管理 | django-cleanup | 9.0.0 |
| 環境変数 | python-dotenv | 1.2.1 |
| HTTP | httpx | 0.28.1 |
| 暗号化 | cryptography | 46.0.4 |
| 圧縮 | brotli / zopfli | 1.2.0 / 0.4.0 |
| WebSocket | websockets | 15.0.1 |
