# アプリケーション設計書 — Unazu-kiro（全自動うなずきマシーン）

## 設計方針

| 決定事項 | 選択 | 理由 |
|---------|------|------|
| PC↔デバイス通信 | WebSocket（PCがサーバー） | 低レイテンシ・双方向・同一LAN前提でシンプル |
| キーワード検知 | PCアプリ内ローカル処理 | レイテンシ1秒以内の要件を満たすため |
| 設定UI共有方式 | 同一プロセス内スレッド | メモリ共有で即時反映・シンプルな構成 |
| Polly音声再生 | ラズパイが直接呼び出し（SORACOM Beam経由） | デバイスにAWS認証情報を置かないセキュアな設計 |
| フォールバック | ラズパイ側で自律実装 | PCとの接続断でも独立動作を保証 |

---

## システムアーキテクチャ概要

```
+------------------------------------------------------------------+
|                    Unazu-kiro システム                            |
|                                                                  |
|  +---------------------------+   WebSocket    +---------------+  |
|  |   PC Agent App            |<------------->|  Device App   |  |
|  |   (Unit 1)                |   (LAN)       |  (Unit 2)     |  |
|  |                           |               |  Raspberry Pi |  |
|  |  - 音声キャプチャ          |               |               |  |
|  |  - AWS Transcribe連携     |               |  - サーボ制御  |  |
|  |  - キーワード検知          |               |  - LED制御    |  |
|  |  - コマンド送信            |               |  - 音声再生   |  |
|  |                           |               |  - フォールバック|  |
|  +---------------------------+               +---------------+  |
|            |                                        |           |
|  +----------+----------+                    SORACOM Arc/Beam    |
|  |  Config Web UI      |                            |           |
|  |  (Unit 3)           |                     AWS Polly (TTS)   |
|  |  localhost:8080     |                                        |
|  |  - キーワード管理    |                                        |
|  |  - タイミング設定    |    AWS Transcribe Streaming           |
|  |  - 接続状態表示      |<-----------------------------------+  |
|  |  - テスト発火        |                                    |  |
|  +---------------------+                                    |  |
|                                                             |  |
+-------------------------------------------------------------+--+
                                                             |
                                              [会議ツール Zoom/Meet]
                                              システム音声キャプチャ
```

---

## コンポーネント一覧

### Unit 1: PC Agent App

| サブコンポーネント | クラス | 主な責務 |
|-----------------|--------|---------|
| 音声キャプチャ | `AudioCapture` | システム音声のリアルタイムキャプチャ |
| 文字起こしクライアント | `TranscribeClient` | AWS Transcribe Streaming 連携 |
| キーワード検知エンジン | `KeywordDetector` | テキストからキーワード・名前を検知 |
| コマンドディスパッチャー | `CommandDispatcher` | WebSocket でデバイスにコマンド送信 |
| タイマーマネージャー | `TimerManager` | 無発言・相槌タイマー管理 |
| 設定マネージャー | `ConfigManager` | keywords.yaml の読み書き・ホットリロード |
| WebSocket サーバー | `WSServer` | ラズパイからの接続受付 |
| 会議オーケストレーション | `MeetingOrchestrationService` | 全体パイプラインの制御 |
| 設定同期 | `ConfigSyncService` | 設定変更の即時反映 |

### Unit 2: Device App（Raspberry Pi）

| サブコンポーネント | クラス | 主な責務 |
|-----------------|--------|---------|
| WebSocket クライアント | `WSClient` | PCへの接続・コマンド受信・再接続 |
| サーボコントローラー | `ServoController` | GPIO/PWM でサーボモーター制御 |
| 音声プレイヤー | `VoicePlayer` | SORACOM Beam 経由で Polly 呼び出し・再生 |
| LED コントローラー | `LEDController` | GPIO で LED 制御 |
| フォールバックマネージャー | `FallbackManager` | 接続断検知・定期うなずきモード |
| デバイスコマンドサービス | `DeviceCommandService` | コマンドルーティング・優先度管理 |

### Unit 3: Config Web UI

| サブコンポーネント | 種別 | 主な責務 |
|-----------------|------|---------|
| FastAPI アプリ | `WebUIApp` | REST API エンドポイント提供 |
| 設定 API ルーター | `config_router` | キーワード等のCRUD |
| ステータス API ルーター | `status_router` | 接続状態・テスト発火 |
| フロントエンド | `index.html` | ブラウザUI（素のHTML/JS） |

---

## デバイスコマンド仕様

| コマンド | action 値 | params | 動作 |
|---------|-----------|--------|------|
| 通常うなずき | `"nod"` | `{ speed: "normal" }` | ゆっくり1〜2回うなずく |
| 忖度ブースト | `"boost"` | `{ duration_sec: 4 }` | 超高速うなずき + LED点滅 |
| 音声発話 | `"voice"` | `{ text: "あー、なるほど…" }` | Polly で発話 + ゆっくりうなずく |
| フォールバック開始 | `"fallback"` | `{ mode: "enter" }` | 定期うなずきモード開始 |
| フォールバック解除 | `"fallback"` | `{ mode: "exit" }` | 通常モードに復帰 |

---

## 設定ファイル仕様（keywords.yaml）

```yaml
# config/keywords.yaml
boost_keywords:
  - 社長
  - 重要
  - 予算
  - 決定
  - 緊急

name_triggers:
  - 田中
  - Tanaka

aizuchi_phrases:
  - あー、なるほど…
  - 確かに…
  - そうですね…
  - うーん、なるほど
  - おっしゃる通りです

silence_threshold_sec: 60
aizuchi_interval_sec: 90
```

---

## 詳細設計ドキュメント

- [コンポーネント定義](./components.md)
- [コンポーネントメソッド定義](./component-methods.md)
- [サービス定義](./services.md)
- [コンポーネント依存関係](./component-dependency.md)
