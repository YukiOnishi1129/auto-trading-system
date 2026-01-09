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
| 状態管理 | TanStack Query | サーバー/クライアント両対応 |
| Lint | Biome + ESLint | 基本Biome、カスタムルールのみESLint |
| ホスティング | Vercel | - |
| 認証 | 未定 | 自分だけアクセス可能にする |

### ジョブ（バッチ処理）

| 項目 | 技術 | 備考 |
|------|------|------|
| 言語 | Python 3.11+ | Webull SDK対応 |
| アーキテクチャ | クリーンアーキテクチャ | ポート・アダプター構成 |
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

## Next.js 設計方針

参考: [Next.js App Routerで破綻しない設計](https://zenn.dev/yukionishi/articles/cd79e39ea6c172)

### レイヤー構成

```
apps/web/src/
├─ app/           # App Router: ルート・レイアウト・メタデータ（薄く保つ）
├─ features/      # ドメインごとの機能群（司令塔）
├─ shared/        # 共通UI・レイアウト・providerなど
└─ external/      # サーバーアダプタ層（dto, handler, service, repository）
```

### 原則

1. **app/ は薄く保つ**
   - ページ内ではデータフェッチを行わない
   - 型付きパラメータを features/ のテンプレートへ渡すだけ

2. **features/ はドメインごとの司令塔**
   - 画面やページではなく「機能のまとまり」で分ける
   - Container / Presenter / Hook パターンで責務分離

3. **external/ は外界との接続口**
   - features層が直接 repository/service を import しない
   - handler が「入口」として中継する
   - バックエンド差し替え時もReactコードに手を入れずに済む

4. **Lintで設計を強制**
   - カスタムESLintルールでアーキテクチャ違反を防ぐ
   - service層への直接import禁止など

### features/ ディレクトリ構成

```
features/{feature-name}/
├─ components/
│  ├─ server/        # Server Components（ページテンプレート）
│  └─ client/        # Container / Presenter / Hook層
│      └─ {Component}/
│          ├─ {Component}Container.tsx
│          ├─ {Component}Presenter.tsx
│          ├─ use{Component}.ts
│          ├─ {Component}.test.tsx
│          └─ index.ts
├─ hooks/            # TanStack Query + クライアントロジック
├─ queries/          # クエリキー + DTOヘルパー
├─ actions/          # Server Actions（薄いラッパー）
└─ types/            # 型定義・Enumなど
```

### external/ ディレクトリ構成

```
external/
├─ dto/              # Zodスキーマ＋TypeScript型
├─ handler/          # Server Action/Server Componentから呼ばれる入口
├─ service/          # ドメインサービス（ビジネスロジック）
├─ repository/       # DBアクセス（Drizzle）
└─ client/           # 外部APIクライアント
```

### データフェッチ戦略

- **Server-first**: サーバーでキャッシュを作り、クライアントにハイドレーション
- **TanStack Query**: サーバー/クライアント両方で同じクエリキーを使用
- **DTO検証**: APIレスポンスは必ずDTOを通して型検証

---

## Python 設計方針（クリーンアーキテクチャ）

### レイヤー構成（ポート・アダプター）

```
apps/jobs/src/
├─ domain/           # ドメイン層（エンティティ、値オブジェクト）
│  ├─ entities/
│  └─ value_objects/
├─ application/      # アプリケーション層（ユースケース）
│  ├─ use_cases/
│  └─ ports/         # ポート（インターフェース定義）
│      ├─ input/     # 入力ポート（ユースケース境界）
│      └─ output/    # 出力ポート（リポジトリ、外部サービス）
├─ adapters/         # アダプター層（実装）
│  ├─ persistence/   # DBアクセス（SQLAlchemy）
│  ├─ external/      # 外部API（Webull, Discord）
│  └─ scheduler/     # ジョブスケジューラ
├─ infrastructure/   # インフラ層
│  ├─ database/      # DB接続設定
│  └─ config/        # 設定管理
└─ jobs/             # エントリーポイント（各ジョブ）
```

### 依存関係の方向

```
jobs → application → domain
         ↓
      adapters → infrastructure
```

- **domain**: 他のどの層にも依存しない
- **application**: domainのみに依存、portsでアダプターを抽象化
- **adapters**: application/portsの実装
- **jobs**: DIコンテナでアダプターを注入し、ユースケースを実行

### ポート・アダプター例

```python
# application/ports/output/position_repository.py（ポート）
class PositionRepository(Protocol):
    def find_open_positions(self) -> list[Position]: ...
    def save(self, position: Position) -> None: ...

# adapters/persistence/sqlalchemy_position_repository.py（アダプター）
class SqlAlchemyPositionRepository:
    def __init__(self, session: Session):
        self.session = session

    def find_open_positions(self) -> list[Position]:
        # SQLAlchemy実装
        ...
```

---

## ディレクトリ構成（全体）

```
/
├── apps/
│   ├── web/                      # Next.js（管理画面）
│   │   ├── src/
│   │   │   ├── app/              # App Router（薄く保つ）
│   │   │   ├── features/         # ドメインごとの機能群
│   │   │   │   ├── dashboard/
│   │   │   │   ├── positions/
│   │   │   │   ├── trades/
│   │   │   │   └── settings/
│   │   │   ├── shared/           # 共通UI・レイアウト
│   │   │   └── external/         # サーバーアダプタ層
│   │   │       ├── dto/
│   │   │       ├── handler/
│   │   │       ├── service/
│   │   │       └── repository/
│   │   ├── eslint-local-rules/   # カスタムESLintルール
│   │   └── package.json
│   │
│   └── jobs/                     # Python（バッチ処理）
│       ├── src/
│       │   ├── domain/           # ドメイン層
│       │   ├── application/      # アプリケーション層
│       │   ├── adapters/         # アダプター層
│       │   ├── infrastructure/   # インフラ層
│       │   └── jobs/             # エントリーポイント
│       ├── migrations/           # Alembic マイグレーション
│       ├── tests/
│       ├── alembic.ini
│       └── pyproject.toml
│
├── docs/                         # ドキュメント
└── README.md
```

---

## Lint 設定

### Biome（基本）

- フォーマット
- 基本的なLintルール

### ESLint（カスタムルールのみ）

```
eslint-local-rules/
├─ restrict-service-imports.js   # service層を直接import禁止（handlerのみ許可）
├─ restrict-action-imports.js    # *.action.ts は client/hooks からのみ利用可能
└─ use-nextjs-helpers.js         # PageProps/LayoutProps の統一
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
