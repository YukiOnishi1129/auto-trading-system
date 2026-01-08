# ドメイン設計

## 概要

要件定義（v1）とユースケースに基づき、自動売買システムのドメインモデルを定義する。

---

## 1. ドメインモデル一覧

| ドメイン | 説明 | 集約ルート |
|---------|------|-----------|
| Position | ポジション（保有中/終了） | ✓ |
| Order | 注文（買い/売りTP/売りSL） | ✓ |
| Trade | 確定損益記録 | ✓ |
| Symbol | 監視対象銘柄 | ✓ |
| DailyPnL | 日次損益集計 | ✓ |
| TradingSettings | 取引設定・フラグ | ✓ |
| SecretMetadata | APIキー期限管理 | ✓ |

---

## 2. エンティティ詳細

### 2.1 Position（ポジション）

ポジションは買い注文が約定してから、売り注文（TP/SL）が約定するまでのライフサイクルを持つ。

```
Position
├── id: PositionId (UUID)
├── symbol_id: SymbolId
├── status: PositionStatus (OPEN | CLOSED)
├── side: PositionSide (LONG)  // v1はロングのみ
├── entry_price: Price
├── entry_quantity: Quantity
├── entry_amount: Amount  // 約定金額（円）
├── exit_price: Price?  // 決済時に設定
├── exit_amount: Amount?
├── realized_pnl: Amount?  // 確定損益
├── take_profit_price: Price  // entry_price × 1.10
├── stop_loss_price: Price  // entry_price × 0.95
├── opened_at: DateTime
├── closed_at: DateTime?
├── created_at: DateTime
└── updated_at: DateTime
```

**ステータス遷移:**
```
OPEN ──(TP/SL約定)──▶ CLOSED
```

**ビジネスルール:**
- 最大保有数は10
- 1ポジションの投入額は約1000円相当
- クローズ時に枠が1つ回復

---

### 2.2 Order（注文）

証券会社に発注する注文を管理する。

```
Order
├── id: OrderId (UUID)
├── position_id: PositionId?  // 売り注文はPositionに紐づく
├── symbol_id: SymbolId
├── external_order_id: String?  // 証券会社の注文ID
├── order_type: OrderType (MARKET | LIMIT | STOP | STOP_LIMIT)
├── side: OrderSide (BUY | SELL)
├── purpose: OrderPurpose (ENTRY | TAKE_PROFIT | STOP_LOSS)
├── status: OrderStatus
├── quantity: Quantity
├── limit_price: Price?
├── stop_price: Price?
├── filled_quantity: Quantity?
├── filled_price: Price?
├── filled_amount: Amount?
├── ordered_at: DateTime
├── filled_at: DateTime?
├── canceled_at: DateTime?
├── created_at: DateTime
└── updated_at: DateTime
```

**OrderStatus:**
```
PENDING ──(発注成功)──▶ OPEN ──(約定)──▶ FILLED
                         │
                         └──(キャンセル)──▶ CANCELED
                         │
                         └──(失敗)──▶ FAILED
```

**OrderPurpose:**
| 値 | 説明 |
|---|------|
| ENTRY | 新規買い注文 |
| TAKE_PROFIT | 利確売り注文（+10%） |
| STOP_LOSS | 損切り売り注文（-5%） |

---

### 2.3 Trade（取引記録）

約定した取引の損益を記録する。集計の源泉データ。

```
Trade
├── id: TradeId (UUID)
├── position_id: PositionId
├── order_id: OrderId
├── symbol_id: SymbolId
├── trade_type: TradeType (ENTRY | EXIT)
├── side: TradeSide (BUY | SELL)
├── quantity: Quantity
├── price: Price
├── amount: Amount
├── realized_pnl: Amount?  // EXITの場合のみ
├── traded_at: DateTime
├── created_at: DateTime
└── updated_at: DateTime
```

---

### 2.4 Symbol（銘柄）

監視対象の銘柄を管理する。

```
Symbol
├── id: SymbolId (UUID)
├── ticker: String  // 例: "7203", "AAPL"
├── name: String  // 例: "トヨタ自動車"
├── market: Market (JP | US)
├── instrument_id: String?  // 証券会社の銘柄ID
├── is_active: Boolean  // 監視対象かどうか
├── created_at: DateTime
└── updated_at: DateTime
```

**Market:**
| 値 | 説明 |
|---|------|
| JP | 日本株 |
| US | 米国株 |

---

### 2.5 DailyPnL（日次損益）

日次の損益集計を保存する。グラフ表示や履歴確認用。

```
DailyPnL
├── id: DailyPnLId (UUID)
├── date: Date
├── realized_pnl: Amount  // その日の確定損益
├── cumulative_pnl: Amount  // 累積確定損益
├── position_count: Integer  // その日の終了時点の保有数
├── new_position_count: Integer  // その日の新規買い件数
├── closed_position_count: Integer  // その日の決済件数
├── created_at: DateTime
└── updated_at: DateTime
```

---

### 2.6 TradingSettings（取引設定）

システム全体の取引設定を管理する。

```
TradingSettings
├── id: TradingSettingsId (UUID)
├── is_buy_enabled: Boolean  // 新規買い可能か
├── max_positions: Integer  // 最大保有数（デフォルト: 10）
├── position_size_amount: Amount  // 1ポジション投入額（デフォルト: 1000）
├── take_profit_percentage: Decimal  // 利確%（デフォルト: 0.10）
├── stop_loss_percentage: Decimal  // 損切%（デフォルト: 0.05）
├── stop_loss_threshold: Amount  // 停止ライン（デフォルト: -1000）
├── created_at: DateTime
└── updated_at: DateTime
```

**ビジネスルール:**
- `cumulative_pnl <= stop_loss_threshold` の場合、`is_buy_enabled = false` に自動更新
- 手動で `is_buy_enabled` を解除可能

---

### 2.7 SecretMetadata（シークレットメタ）

APIキーの期限管理用メタデータ。

```
SecretMetadata
├── id: SecretMetadataId (UUID)
├── secret_name: String  // 例: "WEBULL_APP_SECRET"
├── expires_at: DateTime  // キーの有効期限
├── last_rotated_at: DateTime  // 最終更新日時
├── created_at: DateTime
└── updated_at: DateTime
```

**アラートルール:**
- 残り7日以下: 通常通知
- 残り2日以下: 強通知

---

## 3. 値オブジェクト

| 名前 | 型 | 説明 |
|-----|---|------|
| PositionId | UUID | ポジションID |
| OrderId | UUID | 注文ID |
| TradeId | UUID | 取引ID |
| SymbolId | UUID | 銘柄ID |
| Price | Decimal | 価格 |
| Quantity | Decimal | 数量（端株対応） |
| Amount | Decimal | 金額（円） |
| Percentage | Decimal | パーセンテージ（0.10 = 10%） |

---

## 4. ドメインサービス

### 4.1 PositionService

```
PositionService
├── canOpenNewPosition(): Boolean
│   └── 現在のOPENポジション数 < max_positions かつ is_buy_enabled
├── calculateTakeProfitPrice(entry_price): Price
│   └── entry_price × (1 + take_profit_percentage)
├── calculateStopLossPrice(entry_price): Price
│   └── entry_price × (1 - stop_loss_percentage)
└── getAvailableSlots(): Integer
    └── max_positions - 現在のOPENポジション数
```

### 4.2 PnLService

```
PnLService
├── calculateRealizedPnL(entry_amount, exit_amount): Amount
├── getCumulativePnL(): Amount
├── getTodayPnL(): Amount
└── shouldStopBuying(): Boolean
    └── cumulative_pnl <= stop_loss_threshold
```

### 4.3 NotificationService

```
NotificationService
├── notifyTakeProfit(position): void
├── notifyStopLoss(position): void
├── notifyBuyingStopped(cumulative_pnl): void
├── notifyDailyReport(daily_pnl): void
├── notifySecretExpiring(days_remaining): void
└── notifyError(error): void
```

---

## 5. 集約とリポジトリ

| 集約ルート | リポジトリ |
|-----------|-----------|
| Position | PositionRepository |
| Order | OrderRepository |
| Trade | TradeRepository |
| Symbol | SymbolRepository |
| DailyPnL | DailyPnLRepository |
| TradingSettings | TradingSettingsRepository |
| SecretMetadata | SecretMetadataRepository |

---

## 6. ドメインイベント

| イベント | トリガー | 後続処理 |
|---------|---------|---------|
| PositionOpened | 買い注文約定時 | OCO注文発注 |
| PositionClosed | TP/SL約定時 | Trade記録、通知 |
| TakeProfitFilled | TP約定時 | Discord通知 |
| StopLossFilled | SL約定時 | Discord通知 |
| BuyingDisabled | 停止条件到達時 | Discord通知 |
| SecretExpiring | 期限チェック時 | Discord通知 |

---

## 7. 境界づけられたコンテキスト

```
┌─────────────────────────────────────────────────────────────┐
│                    Trading Context                          │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐   │
│  │ Position │  │  Order   │  │  Trade   │  │  Symbol  │   │
│  └──────────┘  └──────────┘  └──────────┘  └──────────┘   │
│  ┌──────────────────┐  ┌──────────────────┐               │
│  │ TradingSettings  │  │    DailyPnL      │               │
│  └──────────────────┘  └──────────────────┘               │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                  Infrastructure Context                      │
│  ┌──────────────────┐  ┌──────────────────┐                │
│  │  SecretMetadata  │  │  Notification    │                │
│  └──────────────────┘  └──────────────────┘                │
└─────────────────────────────────────────────────────────────┘
```

---

## 次のステップ

1. Webull APIドキュメント確認後、APIマッピングを追加
2. データベーススキーマ設計（DDL）
3. アプリケーション層の設計（ユースケース実装）
