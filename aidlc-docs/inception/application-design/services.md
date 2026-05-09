# サービス定義 — Unazu-kiro

## サービス概要

Unazu-kiro のサービス層は、コンポーネント間のオーケストレーションを担います。
各サービスは複数のサブコンポーネントを協調させて、ユーザー向けの機能を実現します。

---

## Service 1: MeetingOrchestrationService（会議オーケストレーションサービス）

**場所**: `pc_agent/services/meeting_orchestration.py`  
**役割**: 会議中の音声キャプチャ〜キーワード検知〜デバイスコマンド送信の全体フローを制御するメインサービス

### 責務
- AudioCapture → TranscribeClient → KeywordDetector → CommandDispatcher のパイプラインを管理
- TimerManager と連携して生存確認フェイク・ランダム相槌をトリガー
- 会議セッションの開始・停止を制御

### オーケストレーションフロー

```
start_session()
      |
      v
AudioCapture.start()
      |
      v
TranscribeClient.start_streaming()
      |
      | on_transcript コールバック
      v
KeywordDetector.detect(text)
      |
      +-- type == "boost" --> CommandDispatcher.send_command(boost)
      |                       TimerManager.reset_silence_timer()
      +-- type == "name"  --> CommandDispatcher.send_command(boost)
      |                       TimerManager.reset_silence_timer()
      +-- type == "none"  --> CommandDispatcher.send_command(nod)
                              TimerManager.reset_silence_timer()

TimerManager.on_silence_timeout()
      |
      v
CommandDispatcher.send_command(voice) -- 生存確認フェイク

TimerManager.aizuchi_timer_tick()
      |
      v
CommandDispatcher.send_command(voice) -- ランダム相槌ルーレット
```

### メソッド

```python
class MeetingOrchestrationService:
    async def start_session(self) -> None
    # 会議セッションを開始する。全コンポーネントを起動してパイプラインを構築。

    async def stop_session(self) -> None
    # 会議セッションを停止する。全コンポーネントをクリーンアップ。

    def get_session_status(self) -> dict
    # 現在のセッション状態（実行中/停止中、接続デバイス数等）を返す。
```

---

## Service 2: DeviceCommandService（デバイスコマンドサービス）

**場所**: `device/services/device_command.py`  
**役割**: 受信したコマンドを解釈し、適切なハードウェア制御コンポーネントに振り分けるサービス

### 責務
- WSClient から受信したコマンドをルーティング
- コマンドの優先度管理（忖度ブーストは通常うなずきを中断して優先実行）
- フォールバックモードとの切り替え制御

### コマンドルーティング

```
receive_command(cmd)
      |
      +-- action == "nod"      --> ServoController.nod(NORMAL)
      +-- action == "boost"    --> ServoController.nod(TURBO)
      |                            LEDController.blink(BOOST_PATTERN)
      +-- action == "voice"    --> VoicePlayer.speak(text)
      |                            ServoController.nod(SLOW)  ← 相槌中もうなずく
      +-- action == "fallback" --> FallbackManager.enter_fallback_mode()
```

### メソッド

```python
class DeviceCommandService:
    async def handle_command(self, command: DeviceCommand) -> None
    # コマンドを受け取り、適切なコントローラーに振り分ける。

    def set_priority_mode(self, mode: str) -> None
    # 優先モードを設定する（"normal" | "boost" | "fallback"）。
```

---

## Service 3: ConfigSyncService（設定同期サービス）

**場所**: `pc_agent/services/config_sync.py`  
**役割**: Web UI からの設定変更を受け取り、実行中のサービスに即時反映するサービス

### 責務
- ConfigManager の変更を MeetingOrchestrationService に通知
- KeywordDetector のホットリロードをトリガー
- TimerManager のタイミング設定を更新

### メソッド

```python
class ConfigSyncService:
    def on_config_updated(self, new_config: Config) -> None
    # 設定変更時に呼び出される。関連コンポーネントにホットリロードを通知。

    def register_listener(self, listener: Callable[[Config], None]) -> None
    # 設定変更リスナーを登録する。
```

---

## サービス間の関係

```
+---------------------------+
|  Web UI (FastAPI)         |
|  POST /api/config/*       |
+---------------------------+
             |
             | ConfigManager.update()
             v
+---------------------------+
|  ConfigSyncService        |
+---------------------------+
             |
             | on_config_updated()
             v
+---------------------------+
|  MeetingOrchestrationSvc  |
|  - KeywordDetector.reload |
|  - TimerManager.update    |
+---------------------------+
             |
             | send_command()
             v
+---------------------------+
|  DeviceCommandService     |
|  (Raspberry Pi 側)        |
+---------------------------+
```
