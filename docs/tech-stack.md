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

参考: [immortal-architecture-clean](https://github.com/YukiOnishi1129/immortal-architecture-clean/blob/main/backend-clean/docs/02_clean_architecture_guide.md)

### 基本ルール

- **依存は「内側に向かってのみ」許可**
- 外側のレイヤーは内側を知ってよい；内側は外側を知らない
- domain層は外部ライブラリimport不可
- ポート経由で依存（具体実装ではなくインターフェース参照）
- トランザクションはusecase層で管理
- Factoryパターンでリクエストごとに新規インスタンス生成

### レイヤー構成

```
apps/jobs/src/
├─ domain/           # ビジネスルール（フレームワーク非依存）
│  ├─ entities/      # エンティティ
│  ├─ value_objects/ # 値オブジェクト
│  └─ errors/        # ドメインエラー
├─ usecase/          # アプリケーションロジック（処理の手順・流れ）
├─ port/             # インターフェース定義（約束事）
│  ├─ input/         # 入力ポート（ユースケース境界）
│  └─ output/        # 出力ポート（リポジトリ、外部サービス）
├─ adapter/          # 外部接続（型・形式の変換）
│  ├─ gateway/       # DBアクセス（SQLAlchemy）
│  └─ client/        # 外部API（Webull, Discord）
└─ driver/           # 初期化・Factory・設定
    ├─ factory/      # DIファクトリ
    ├─ config/       # 設定管理
    └─ jobs/         # エントリーポイント（各ジョブ）
```

### 4つのレイヤーの責務

| レイヤー | 役割 | 例 |
|---------|------|-----|
| **domain** | ビジネスルール定義 | "ポジションは最大10まで" |
| **usecase** | 処理の手順・流れ | "取得→検証→保存→通知" |
| **port** | インターフェース定義 | リポジトリ・外部サービスの抽象 |
| **adapter** | 型・形式の変換、外部接続 | DB行↔ドメインモデル変換 |
| **driver** | 初期化・DI・設定 | Factory、エントリーポイント |

### 依存関係の方向（同心円）

```
        ┌─────────────────────────────────────┐
        │            driver (外側)            │
        │  ┌─────────────────────────────┐   │
        │  │         adapter             │   │
        │  │  ┌─────────────────────┐   │   │
        │  │  │        port         │   │   │
        │  │  │  ┌─────────────┐   │   │   │
        │  │  │  │   usecase   │   │   │   │
        │  │  │  │  ┌───────┐  │   │   │   │
        │  │  │  │  │domain │  │   │   │   │
        │  │  │  │  │(中心) │  │   │   │   │
        │  │  │  │  └───────┘  │   │   │   │
        │  │  │  └─────────────┘   │   │   │
        │  │  └─────────────────────┘   │   │
        │  └─────────────────────────────┘   │
        └─────────────────────────────────────┘
                    依存は内側へのみ →
```

- **domain**: 他のどの層にも依存しない（純粋なビジネスロジック）
- **usecase**: domainのみに依存、portでアダプターを抽象化
- **port**: domain/usecaseが使うインターフェース定義
- **adapter**: portの実装、DB例外→ドメイン例外への変換
- **driver**: DIコンテナでアダプターを注入し、usecaseを実行

### リクエスト処理フロー

```
Job Entry → Factory → UseCase → Port(Repository) → Gateway → DB
              ↓生成      ↓依頼        ↓抽象          ↓実装
           UseCase ← Domain型 ←────── 変換
              ↓結果
           通知・ログ
```

### ポート・アダプター例

```python
# port/output/position_repository.py（ポート：約束事）
from typing import Protocol

class PositionRepository(Protocol):
    """ポジションリポジトリのインターフェース"""
    def find_open_positions(self) -> list[Position]: ...
    def save(self, position: Position) -> None: ...
    def find_by_id(self, position_id: PositionId) -> Position | None: ...

# adapter/gateway/sqlalchemy_position_gateway.py（アダプター：実装）
class SqlAlchemyPositionGateway:
    """SQLAlchemyによるPositionRepositoryの実装"""
    def __init__(self, session: Session):
        self._session = session

    def find_open_positions(self) -> list[Position]:
        rows = self._session.query(PositionModel).filter(
            PositionModel.status == "OPEN"
        ).all()
        return [self._to_domain(row) for row in rows]

    def save(self, position: Position) -> None:
        model = self._to_model(position)
        self._session.merge(model)

    def _to_domain(self, row: PositionModel) -> Position:
        """DB行 → ドメインモデル変換"""
        return Position(
            id=PositionId(row.id),
            symbol_id=SymbolId(row.symbol_id),
            status=PositionStatus(row.status),
            # ...
        )

    def _to_model(self, entity: Position) -> PositionModel:
        """ドメインモデル → DB行変換"""
        return PositionModel(
            id=str(entity.id),
            symbol_id=str(entity.symbol_id),
            status=entity.status.value,
            # ...
        )
```

### Factoryパターン

```python
# driver/factory/usecase_factory.py
class UseCaseFactory:
    """ユースケースのファクトリ（DIコンテナ）"""
    def __init__(self, config: Config, session_factory: SessionFactory):
        self._config = config
        self._session_factory = session_factory

    def create_daily_batch_usecase(self) -> DailyBatchUseCase:
        """日次バッチユースケースを生成"""
        session = self._session_factory.create()
        return DailyBatchUseCase(
            position_repo=SqlAlchemyPositionGateway(session),
            order_repo=SqlAlchemyOrderGateway(session),
            webull_client=WebullClient(self._config.webull),
            discord_client=DiscordClient(self._config.discord),
            settings_repo=SqlAlchemySettingsGateway(session),
        )

# driver/jobs/daily_batch.py（エントリーポイント）
def main():
    config = load_config()
    factory = UseCaseFactory(config, create_session_factory(config.database))

    usecase = factory.create_daily_batch_usecase()
    usecase.execute()
```

### エラーハンドリング

```python
# domain/errors/position_error.py（ドメインエラー）
class PositionNotFoundError(DomainError):
    """ポジションが見つからない"""
    pass

class MaxPositionsExceededError(DomainError):
    """最大ポジション数超過"""
    pass

# adapter/gateway/sqlalchemy_position_gateway.py（エラー変換）
def find_by_id(self, position_id: PositionId) -> Position:
    try:
        row = self._session.query(PositionModel).filter(
            PositionModel.id == str(position_id)
        ).one()
        return self._to_domain(row)
    except NoResultFound:
        raise PositionNotFoundError(f"Position not found: {position_id}")
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
│       │   ├── domain/           # ビジネスルール
│       │   │   ├── entities/
│       │   │   ├── value_objects/
│       │   │   └── errors/
│       │   ├── usecase/          # アプリケーションロジック
│       │   ├── port/             # インターフェース定義
│       │   │   ├── input/
│       │   │   └── output/
│       │   ├── adapter/          # 外部接続
│       │   │   ├── gateway/      # DB（SQLAlchemy）
│       │   │   └── client/       # 外部API
│       │   └── driver/           # 初期化・Factory
│       │       ├── factory/
│       │       ├── config/
│       │       └── jobs/         # エントリーポイント
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
