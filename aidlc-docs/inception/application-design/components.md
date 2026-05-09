# コンポーネント定義 — Unazu-kiro

## コンポーネント一覧

```
+------------------------------------------------------------------+
|                    Unazu-kiro システム全体                        |
|                                                                  |
|  +----------------------+      +----------------------------+   |
|  |   PC Agent App       |      |   Device App               |   |
|  |   (Unit 1)           |      |   (Unit 2)                 |   |
|  +----------------------+      +----------------------------+   |
|                                                                  |
|  +----------------------+      +----------------------------+   |
|  |   Config Web UI      |      |   AWS Cloud Services       |   |
|  |   (Unit 3)           |      |   (外部)                   |   |
|  +----------------------+      +----------------------------+   |
+------------------------------------------------------------------+
```

---

## Component 1: PC Agent App（PCエージェントアプリ）

**パッケージ**: `pc_agent/`  
**起動コマンド**: `python main.py`  
**役割**: 会議音声のキャプチャ・文字起こし・キーワード検知・デバイスへのコマンド送信・設定Web UIのホスト

### 責務
- システム音声（仮想オーディオデバイス）をリアルタイムキャプチャ
- AWS Transcribe Streaming API へ音声ストリームを送信
- 文字起こし結果からキーワードを検知（ローカル処理）
- 検知結果に応じたコマンドを Raspberry Pi へ WebSocket で送信
- 無発言タイマーを管理し、生存確認フェイクをトリガー
- ランダム相槌ルーレットのタイマーを管理
- FastAPI サーバーをバックグラウンドスレッドで起動（設定Web UI）
- `config/keywords.yaml` の読み書き

### サブコンポーネント

| サブコンポーネント | クラス名 | 責務 |
|-----------------|---------|------|
| 音声キャプチャ | `AudioCapture` | システム音声のリアルタイムキャプチャ |
| 文字起こしクライアント | `TranscribeClient` | AWS Transcribe Streaming との通信 |
| キーワード検知エンジン | `KeywordDetector` | テキストからキーワード・名前を検知 |
| コマンドディスパッチャー | `CommandDispatcher` | WebSocket でデバイスにコマンド送信 |
| タイマーマネージャー | `TimerManager` | 無発言タイマー・相槌タイマーの管理 |
| 設定マネージャー | `ConfigManager` | keywords.yaml の読み書き・ホットリロード |
| WebSocket サーバー | `WSServer` | ラズパイからの接続を受け付けるサーバー |

---

## Component 2: Device App（デバイスアプリ）

**パッケージ**: `device/`  
**起動コマンド**: `python device.py`（Raspberry Pi 上で実行）  
**役割**: PCからのコマンド受信・サーボ制御・音声再生・LED制御・自律フォールバック

### 責務
- PC Agent App の WebSocket サーバーに接続・コマンド受信
- コマンドに応じてサーボモーターを制御（うなずきパターン実行）
- SORACOM Arc + SORACOM Beam 経由で AWS Polly を呼び出し音声再生
- LED を制御して忖度ブースト発動を視覚演出
- WebSocket 接続タイムアウト検知 → 自律フォールバック（定期うなずき）モードへ移行

### サブコンポーネント

| サブコンポーネント | クラス名 | 責務 |
|-----------------|---------|------|
| WebSocket クライアント | `WSClient` | PCへの接続・コマンド受信・再接続処理 |
| サーボコントローラー | `ServoController` | GPIO/PWM でサーボモーターを制御 |
| 音声プレイヤー | `VoicePlayer` | SORACOM Beam 経由で Polly 呼び出し・再生 |
| LED コントローラー | `LEDController` | GPIO で LED を制御 |
| フォールバックマネージャー | `FallbackManager` | 接続断検知・定期うなずきモード管理 |

---

## Component 3: Config Web UI（設定Webダッシュボード）

**パッケージ**: `pc_agent/web_ui/`  
**アクセス**: `http://localhost:8080`（PC Agent App 起動時に自動起動）  
**役割**: ブラウザから設定を変更するローカルWebダッシュボード

### 責務
- キーワード・名前・フレーズの一覧表示・追加・削除
- タイミング設定（無発言閾値・相槌間隔）の変更
- Raspberry Pi 接続状態のリアルタイム表示
- 各機能のテスト発火ボタン
- 設定変更を ConfigManager 経由で即時反映

### サブコンポーネント

| サブコンポーネント | 種別 | 責務 |
|-----------------|------|------|
| FastAPI アプリ | `WebUIApp` | REST API エンドポイントの提供 |
| 設定 API ルーター | `config_router` | CRUD エンドポイント（/api/config/*） |
| ステータス API ルーター | `status_router` | デバイス接続状態・テスト発火エンドポイント |
| フロントエンド | `static/index.html` | 素のHTML/CSS/JS によるUI |

---

## 外部サービス（コンポーネントではないが設計上重要）

| サービス | 用途 | 利用コンポーネント |
|---------|------|-----------------|
| AWS Transcribe Streaming | リアルタイム音声認識 | PC Agent App |
| AWS Polly | テキスト読み上げ（TTS） | Device App（SORACOM Beam経由） |
| SORACOM Arc | デバイスからのセキュアなクラウド接続 | Device App |
| SORACOM Beam | AWS認証情報をデバイスに持たせずにAWS APIを呼び出す | Device App |
