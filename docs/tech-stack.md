# 技術スタック

## 概要

自動売買システムの技術スタック定義。

---

## アーキテクチャ図

```
┌─────────────────────────────────────┐
│  Next.js（Vercel）                  │
│  - 管理画面UI                        │
│  - Server Actions → DB操作          │
│  - Drizzle ORM                      │
└──────────────┬──────────────────────┘
               │
               ▼
┌─────────────────────────────────────┐
│  Neon（PostgreSQL）                 │
└──────────────┬──────────────────────┘
               ▲
               │
┌──────────────┴──────────────────────┐
│  Python（Cloud Run Jobs）           │
│  - 日次バッチ（買いシグナル判定）     │
│  - 約定検知ポーリング（5分毎）        │
│  - Webull API連携                   │
│  - Discord通知                      │
│  - SQLAlchemy + Alembic             │
└─────────────────────────────────────┘
```

---

## 技術選定

### フロントエンド / 管理画面

| 項目 | 技術 | 備考 |
|------|------|------|
| フレームワーク | Next.js (App Router) | Server Actions でDB操作 |
| ORM | Drizzle | スキーマはPython側に合わせる |
| ホスティング | Vercel | - |
| 認証 | 未定 | 自分だけアクセス可能にする |

### ジョブ（バッチ処理）

| 項目 | 技術 | 備考 |
|------|------|------|
| 言語 | Python 3.11+ | Webull SDK対応 |
| ORM | SQLAlchemy | - |
| マイグレーション | Alembic | DBスキーマの正はこちら |
| 実行環境 | Cloud Run Jobs | - |
| スケジューラ | Cloud Scheduler | 日次・5分毎のトリガー |
| SDK | webull-python-sdk | 公式SDK |

### データベース

| 項目 | 技術 | 備考 |
|------|------|------|
| DB | Neon | サーバーレスPostgreSQL |
| 接続 | Connection Pooling | Neon標準機能 |

### インフラ / その他

| 項目 | 技術 | 備考 |
|------|------|------|
| シークレット管理 | GCP Secret Manager | APIキー等の保存 |
| 通知 | Discord Webhook | 取引通知・アラート |
| CI/CD | GitHub Actions | 未定 |
| モニタリング | 未定 | Cloud Logging等 |

---

## ディレクトリ構成（案）

```
/
├── apps/
│   ├── web/                  # Next.js（管理画面）
│   │   ├── src/
│   │   │   ├── app/          # App Router
│   │   │   ├── components/
│   │   │   ├── lib/
│   │   │   │   └── db/       # Drizzle設定
│   │   │   └── actions/      # Server Actions
│   │   └── package.json
│   │
│   └── jobs/                 # Python（バッチ処理）
│       ├── src/
│       │   ├── models/       # SQLAlchemyモデル
│       │   ├── services/     # ビジネスロジック
│       │   ├── jobs/         # 各ジョブの実装
│       │   └── lib/
│       │       ├── webull/   # Webull API連携
│       │       └── discord/  # Discord通知
│       ├── migrations/       # Alembic マイグレーション
│       ├── alembic.ini
│       └── pyproject.toml
│
├── docs/                     # ドキュメント
└── README.md
```

---

## マイグレーション戦略

- **スキーマの正（Single Source of Truth）**: Python側（Alembic）
- **Next.js側（Drizzle）**: Python側のスキーマに合わせて定義
- **実行タイミング**: CI/CDでデプロイ前に実行

---

## 今後の検討事項

- [ ] 認証方式（NextAuth? Clerk? 独自実装?）
- [ ] モニタリング・ロギング戦略
- [ ] エラーハンドリング・リトライ戦略
- [ ] テスト戦略（単体・統合・E2E）
