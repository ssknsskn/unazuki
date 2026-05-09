# ユニット依存関係 — Unazu-kiro

## 依存関係マトリクス

| 依存元 \ 依存先 | Unit 1 (PC Agent) | Unit 3 (Config UI) | Unit 2 (Device) |
|---------------|:-----------------:|:------------------:|:---------------:|
| **Unit 1** | — | 起動・参照を提供 | WebSocket API を提供 |
| **Unit 3** | ConfigManager・Dispatcher・WSServer に依存 | — | なし |
| **Unit 2** | WebSocket サーバーに依存 | なし | — |

## 依存関係図

```
+------------------+
|   Unit 1         |
|   PC Agent App   |
+------------------+
    |         |
    |         | WebSocket API
    |         | (ws://host:8765)
    |         v
    |   +------------------+
    |   |   Unit 2         |
    |   |   Device App     |
    |   |   (Raspberry Pi) |
    |   +------------------+
    |
    | 同一プロセス内
    | ConfigManager / Dispatcher / WSServer 参照
    v
+------------------+
|   Unit 3         |
|   Config Web UI  |
|   (localhost:8080)|
+------------------+
```

## 開発順序と理由

### 順序: Unit 1 → Unit 3 → Unit 2

```
Week 1 前半: Unit 1 (PC Agent App)
  |
  | 完了条件: WebSocket サーバー起動・キーワード検知動作確認
  v
Week 1 中盤: Unit 3 (Config Web UI)
  |
  | 完了条件: ブラウザから設定変更・即時反映確認
  v
Week 1 後半: Unit 2 (Device App)
  |
  | 完了条件: サーボ動作・音声再生・フォールバック確認
  v
統合テスト: 全ユニット連携動作確認
```

### 各ユニットの開発前提条件

| ユニット | 開発開始の前提 | 理由 |
|---------|-------------|------|
| Unit 1 | なし（最初に開発） | 他ユニットの基盤となるため |
| Unit 3 | Unit 1 の ConfigManager・WSServer が実装済み | 同一プロセス内で直接参照するため |
| Unit 2 | Unit 1 の WebSocket API 仕様が確定済み | コマンド形式・ポート番号が必要なため |

## インターフェース仕様（ユニット間）

### Unit 1 → Unit 2: WebSocket コマンド仕様

**接続先**: `ws://<PC_IP>:8765`

```json
// 通常うなずき
{ "action": "nod", "params": { "speed": "normal" } }

// 忖度ブースト
{ "action": "boost", "params": { "duration_sec": 4 } }

// 音声発話
{ "action": "voice", "params": { "text": "あー、なるほど…" } }

// フォールバック制御
{ "action": "fallback", "params": { "mode": "enter" } }
{ "action": "fallback", "params": { "mode": "exit" } }
```

### Unit 3 → Unit 1: 内部 Python API

```python
# ConfigManager（直接参照）
config_manager.get()           # 現在の設定を取得
config_manager.update(partial) # 設定を更新

# CommandDispatcher（直接参照）
await dispatcher.send_command(DeviceCommand(...))  # テスト発火

# WSServer（直接参照）
ws_server.get_connection_status()  # 接続状態取得
```

## 並行開発の可能性

Unit 1 の WebSocket API 仕様（コマンド形式）が確定した時点で、Unit 2 の開発を Unit 3 と並行して進めることも可能です。ただし、ハッカソンの1人または少人数開発を想定し、順次開発（Unit 1 → Unit 3 → Unit 2）を推奨します。
