# コンポーネントメソッド定義 — Unazu-kiro

> **注意**: 詳細なビジネスロジックはコンストラクションフェーズの機能設計で定義します。
> ここではメソッドシグネチャと高レベルの目的を定義します。

---

## Component 1: PC Agent App

### AudioCapture

```python
class AudioCapture:
    def start(self) -> None
    # システム音声キャプチャを開始する。仮想オーディオデバイスを選択してストリーミング開始。

    def stop(self) -> None
    # キャプチャを停止してリソースを解放する。

    def get_audio_chunk(self) -> bytes
    # 音声チャンクを取得して返す（ブロッキング）。TranscribeClient が消費する。

    def list_audio_devices(self) -> list[dict]
    # 利用可能なオーディオデバイス一覧を返す（セットアップ時の確認用）。
```

### TranscribeClient

```python
class TranscribeClient:
    def start_streaming(self, audio_source: AudioCapture) -> None
    # AWS Transcribe Streaming セッションを開始し、音声チャンクを送信し続ける。

    def stop_streaming(self) -> None
    # ストリーミングセッションを終了する。

    def on_transcript(self, callback: Callable[[str, bool], None]) -> None
    # 文字起こし結果を受け取るコールバックを登録する。
    # callback(text: str, is_final: bool) の形式で呼び出される。
```

### KeywordDetector

```python
class KeywordDetector:
    def detect(self, text: str) -> DetectionResult
    # テキストからキーワード・名前を検知し、検知種別と一致したキーワードを返す。
    # DetectionResult: { type: "boost"|"name"|"none", keyword: str|None }

    def reload_config(self, config: Config) -> None
    # 設定変更時にキーワードリストをホットリロードする。
```

### CommandDispatcher

```python
class CommandDispatcher:
    async def send_command(self, command: DeviceCommand) -> None
    # デバイスに WebSocket コマンドを送信する。
    # DeviceCommand: { action: "nod"|"boost"|"voice"|"led", params: dict }

    def get_connected_clients(self) -> list[str]
    # 接続中のデバイスクライアント一覧を返す（接続状態確認用）。
```

### TimerManager

```python
class TimerManager:
    def reset_silence_timer(self) -> None
    # 無発言タイマーをリセットする（発言検知時に呼び出す）。

    def on_silence_timeout(self, callback: Callable[[], None]) -> None
    # 無発言タイムアウト時のコールバックを登録する。

    def start_aizuchi_timer(self) -> None
    # ランダム相槌ルーレットの定期タイマーを開始する。

    def stop_all(self) -> None
    # 全タイマーを停止する。
```

### ConfigManager

```python
class ConfigManager:
    def load(self) -> Config
    # keywords.yaml を読み込んで Config オブジェクトを返す。

    def save(self, config: Config) -> None
    # Config オブジェクトを keywords.yaml に書き込む。

    def get(self) -> Config
    # 現在のインメモリ設定を返す（ファイル読み込みなし）。

    def update(self, partial: dict) -> Config
    # 設定の一部を更新してメモリと yaml に反映する。
```

### WSServer

```python
class WSServer:
    async def start(self, host: str, port: int) -> None
    # WebSocket サーバーを起動する。

    async def broadcast(self, message: dict) -> None
    # 接続中の全クライアントにメッセージをブロードキャストする。

    def get_connection_status(self) -> dict
    # 接続中クライアント数と接続状態を返す。
```

---

## Component 2: Device App

### WSClient

```python
class WSClient:
    async def connect(self, uri: str) -> None
    # PC Agent App の WebSocket サーバーに接続する。

    async def listen(self) -> None
    # コマンドを受信し続けるループ。受信したコマンドをハンドラーに渡す。

    def on_command(self, callback: Callable[[DeviceCommand], None]) -> None
    # コマンド受信時のコールバックを登録する。

    def is_connected(self) -> bool
    # 現在の接続状態を返す。

    async def reconnect(self) -> None
    # 切断時に再接続を試みる（指数バックオフ）。
```

### ServoController

```python
class ServoController:
    def nod(self, pattern: NodPattern) -> None
    # 指定パターンでうなずき動作を実行する。
    # NodPattern: { speed: "slow"|"normal"|"fast"|"turbo", duration_sec: float, count: int }

    def stop(self) -> None
    # サーボを中立位置に戻して停止する。

    def set_angle(self, angle: int) -> None
    # サーボを指定角度（0〜180度）に移動する（デバッグ・調整用）。
```

### VoicePlayer

```python
class VoicePlayer:
    async def speak(self, text: str) -> None
    # SORACOM Beam 経由で AWS Polly を呼び出し、音声を生成してスピーカーから再生する。

    def play_audio(self, audio_data: bytes) -> None
    # 音声データをスピーカーから再生する。

    def set_volume(self, volume: float) -> None
    # 音量を設定する（0.0〜1.0）。
```

### LEDController

```python
class LEDController:
    def blink(self, pattern: LEDPattern) -> None
    # 指定パターンで LED を点滅させる。
    # LEDPattern: { color: str, interval_ms: int, duration_sec: float }

    def on(self) -> None
    # LED を点灯する。

    def off(self) -> None
    # LED を消灯する。
```

### FallbackManager

```python
class FallbackManager:
    def start_monitoring(self, ws_client: WSClient) -> None
    # WebSocket 接続状態の監視を開始する。

    def enter_fallback_mode(self) -> None
    # フォールバックモードに移行し、定期うなずきを開始する。

    def exit_fallback_mode(self) -> None
    # フォールバックモードを終了し、通常モードに戻る。

    def is_fallback_active(self) -> bool
    # フォールバックモードが有効かどうかを返す。
```

---

## Component 3: Config Web UI

### WebUIApp（FastAPI）

```python
# REST API エンドポイント一覧

GET  /                          # ダッシュボードHTML を返す
GET  /api/config                # 現在の設定全体を返す
PUT  /api/config                # 設定全体を更新する
POST /api/config/keywords       # キーワードを追加する  { keyword: str }
DELETE /api/config/keywords/{keyword}  # キーワードを削除する
POST /api/config/names          # 名前トリガーを追加する  { name: str }
DELETE /api/config/names/{name} # 名前トリガーを削除する
POST /api/config/phrases        # 相槌フレーズを追加する  { phrase: str }
DELETE /api/config/phrases/{phrase}  # 相槌フレーズを削除する
GET  /api/status                # デバイス接続状態を返す
POST /api/test/{action}         # テスト発火 { action: "nod"|"boost"|"voice" }
```

---

## 共通データ型

```python
# 設定オブジェクト
class Config:
    boost_keywords: list[str]       # 忖度ブーストキーワード
    name_triggers: list[str]        # 褒め言葉レーダー用名前
    aizuchi_phrases: list[str]      # ランダム相槌フレーズ
    silence_threshold_sec: int      # 無発言閾値（秒）デフォルト60
    aizuchi_interval_sec: int       # 相槌間隔（秒）デフォルト90

# デバイスコマンド
class DeviceCommand:
    action: str     # "nod" | "boost" | "voice" | "led" | "fallback"
    params: dict    # アクション固有のパラメータ

# キーワード検知結果
class DetectionResult:
    type: str       # "boost" | "name" | "none"
    keyword: str | None
```
