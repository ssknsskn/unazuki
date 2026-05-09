# ユニット・オブ・ワーク定義 — Unazu-kiro

## ユニット一覧

| ユニット | 名称 | 実行環境 | 開発順序 |
|---------|------|---------|---------|
| Unit 1 | PC Agent App | macOS（PC） | 1番目 |
| Unit 3 | Config Web UI | macOS（PC、Unit 1 と同一プロセス） | 2番目 |
| Unit 2 | Device App | Raspberry Pi | 3番目 |

> **開発順序の理由**: Unit 1 が WebSocket サーバー・設定管理・コマンドディスパッチャーの基盤を提供するため最初に実装。Unit 3 は Unit 1 の ConfigManager に依存するため2番目。Unit 2 は Unit 1 の WebSocket API が確定してから実装するため最後。

---

## Unit 1: PC Agent App

### 概要
```
+--------------------------------------------------+
|  Unit 1: PC Agent App                            |
|  実行環境: macOS                                  |
|  起動: python pc_agent/main.py                   |
|                                                  |
|  責務:                                           |
|  - システム音声のリアルタイムキャプチャ            |
|  - AWS Transcribe Streaming で文字起こし          |
|  - キーワード・名前のローカル検知                  |
|  - Raspberry Pi へ WebSocket コマンド送信         |
|  - 無発言タイマー・相槌タイマー管理               |
|  - keywords.yaml の読み書き                      |
|  - Unit 3（Config Web UI）をスレッドで起動        |
+--------------------------------------------------+
```

### 含まれるモジュール

| モジュール | パス | 主なクラス |
|----------|------|----------|
| エントリーポイント | `pc_agent/main.py` | — |
| 音声キャプチャ | `pc_agent/audio/capture.py` | `AudioCapture` |
| 文字起こしクライアント | `pc_agent/transcribe/client.py` | `TranscribeClient` |
| キーワード検知 | `pc_agent/detection/keyword_detector.py` | `KeywordDetector` |
| WebSocket サーバー | `pc_agent/device/ws_server.py` | `WSServer` |
| コマンドディスパッチャー | `pc_agent/device/dispatcher.py` | `CommandDispatcher` |
| タイマーマネージャー | `pc_agent/timer/timer_manager.py` | `TimerManager` |
| 設定マネージャー | `pc_agent/config/config_manager.py` | `ConfigManager` |
| 会議オーケストレーション | `pc_agent/services/meeting_orchestration.py` | `MeetingOrchestrationService` |
| 設定同期サービス | `pc_agent/services/config_sync.py` | `ConfigSyncService` |

### 主要な外部依存

| 依存先 | 用途 | ライブラリ |
|--------|------|----------|
| AWS Transcribe Streaming | リアルタイム音声認識 | `boto3`, `amazon-transcribe` |
| システム音声キャプチャ | 仮想オーディオデバイス | `sounddevice` または `pyaudio` |
| WebSocket サーバー | デバイスとの通信 | `websockets` |
| 設定ファイル | キーワード管理 | `PyYAML` |

### 完了定義
- `python pc_agent/main.py` で起動できる
- システム音声をキャプチャして AWS Transcribe に送信できる
- キーワード検知が動作し、WebSocket でコマンドを送信できる
- 無発言タイマー・相槌タイマーが動作する
- Unit 3（Config Web UI）がスレッドとして起動する

---

## Unit 3: Config Web UI

### 概要
```
+--------------------------------------------------+
|  Unit 3: Config Web UI                           |
|  実行環境: macOS（Unit 1 と同一プロセス内スレッド）|
|  アクセス: http://localhost:8080                  |
|                                                  |
|  責務:                                           |
|  - キーワード・名前・フレーズの CRUD API 提供     |
|  - タイミング設定の変更 API 提供                  |
|  - デバイス接続状態のリアルタイム表示             |
|  - テスト発火ボタン（各機能の手動トリガー）        |
|  - ブラウザ向け HTML/CSS/JS UI の配信            |
+--------------------------------------------------+
```

### 含まれるモジュール

| モジュール | パス | 主なクラス/内容 |
|----------|------|--------------|
| FastAPI アプリ | `pc_agent/web_ui/app.py` | `WebUIApp` |
| 設定 API ルーター | `pc_agent/web_ui/routers/config.py` | `config_router` |
| ステータス API ルーター | `pc_agent/web_ui/routers/status.py` | `status_router` |
| フロントエンド | `pc_agent/web_ui/static/index.html` | HTML/CSS/JS |

### 主要な外部依存

| 依存先 | 用途 | ライブラリ |
|--------|------|----------|
| Unit 1 の ConfigManager | 設定の読み書き | 同一プロセス内参照 |
| Unit 1 の CommandDispatcher | テスト発火コマンド送信 | 同一プロセス内参照 |
| Unit 1 の WSServer | 接続状態取得 | 同一プロセス内参照 |
| FastAPI | REST API フレームワーク | `fastapi`, `uvicorn` |

### 完了定義
- `http://localhost:8080` でダッシュボードが表示される
- キーワードの追加・削除が動作し、即座に keywords.yaml に反映される
- デバイス接続状態がリアルタイムで表示される
- テスト発火ボタンで各機能が動作する

---

## Unit 2: Device App

### 概要
```
+--------------------------------------------------+
|  Unit 2: Device App                              |
|  実行環境: Raspberry Pi                           |
|  起動: python device/device.py                   |
|                                                  |
|  責務:                                           |
|  - Unit 1 の WebSocket サーバーに接続            |
|  - コマンド受信 → サーボモーター制御              |
|  - コマンド受信 → LED 制御                       |
|  - SORACOM Arc + Beam 経由で AWS Polly 呼び出し  |
|  - 音声データをスピーカーから再生                 |
|  - 接続断検知 → 自律フォールバックモード移行      |
+--------------------------------------------------+
```

### 含まれるモジュール

| モジュール | パス | 主なクラス |
|----------|------|----------|
| エントリーポイント | `device/device.py` | — |
| WebSocket クライアント | `device/ws/client.py` | `WSClient` |
| サーボコントローラー | `device/hardware/servo.py` | `ServoController` |
| LED コントローラー | `device/hardware/led.py` | `LEDController` |
| 音声プレイヤー | `device/audio/voice_player.py` | `VoicePlayer` |
| フォールバックマネージャー | `device/fallback/fallback_manager.py` | `FallbackManager` |
| デバイスコマンドサービス | `device/services/device_command.py` | `DeviceCommandService` |

### 主要な外部依存

| 依存先 | 用途 | ライブラリ/方式 |
|--------|------|--------------|
| Unit 1 の WebSocket サーバー | コマンド受信 | `websockets` |
| AWS Polly（SORACOM Beam経由） | TTS 音声生成 | `requests`（HTTPS直接呼び出し） |
| SORACOM Arc | セキュアなクラウド接続 | WireGuard VPN |
| GPIO（サーボ・LED） | ハードウェア制御 | `RPi.GPIO` または `pigpio` |
| スピーカー | 音声再生 | `pygame` または `playsound` |

### 完了定義
- `python device/device.py` で起動し、Unit 1 に接続できる
- `nod` コマンドでサーボモーターが動作する
- `boost` コマンドで超高速うなずき + LED 点滅が動作する
- `voice` コマンドで Polly 音声がスピーカーから再生される
- 接続断後60秒で自律フォールバックモードに移行する

---

## プロジェクト全体のディレクトリ構成

```
unazuki/                              # リポジトリルート
|
+-- pc_agent/                         # Unit 1 + Unit 3
|   +-- main.py                       # エントリーポイント
|   +-- audio/
|   |   +-- capture.py
|   +-- transcribe/
|   |   +-- client.py
|   +-- detection/
|   |   +-- keyword_detector.py
|   +-- device/
|   |   +-- ws_server.py
|   |   +-- dispatcher.py
|   +-- timer/
|   |   +-- timer_manager.py
|   +-- config/
|   |   +-- config_manager.py
|   |   +-- keywords.yaml             # 設定ファイル（デフォルト値入り）
|   +-- services/
|   |   +-- meeting_orchestration.py
|   |   +-- config_sync.py
|   +-- web_ui/                       # Unit 3
|       +-- app.py
|       +-- routers/
|       |   +-- config.py
|       |   +-- status.py
|       +-- static/
|           +-- index.html
|
+-- device/                           # Unit 2
|   +-- device.py                     # エントリーポイント
|   +-- ws/
|   |   +-- client.py
|   +-- hardware/
|   |   +-- servo.py
|   |   +-- led.py
|   +-- audio/
|   |   +-- voice_player.py
|   +-- fallback/
|   |   +-- fallback_manager.py
|   +-- services/
|       +-- device_command.py
|
+-- requirements.txt                  # PC側依存パッケージ
+-- requirements-device.txt           # Raspberry Pi側依存パッケージ
+-- .env.example                      # 環境変数テンプレート
+-- README.md
+-- .gitignore
```
