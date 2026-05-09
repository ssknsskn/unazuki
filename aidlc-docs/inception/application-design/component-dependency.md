# コンポーネント依存関係 — Unazu-kiro

## 依存関係マトリクス

| 依存元 | 依存先 | 種別 | 通信方式 |
|--------|--------|------|---------|
| PC Agent App | AWS Transcribe Streaming | 外部API | HTTPS/WebSocket |
| PC Agent App | Device App | 内部通信 | WebSocket（LAN） |
| PC Agent App | Config Web UI | 同一プロセス | スレッド間メモリ共有 |
| Device App | PC Agent App | 内部通信 | WebSocket（LAN） |
| Device App | AWS Polly | 外部API | HTTPS（SORACOM Beam経由） |
| Device App | SORACOM Arc/Beam | 外部サービス | WireGuard VPN |
| Config Web UI | PC Agent App | 同一プロセス | ConfigManager 直接参照 |

---

## システム全体のデータフロー図

```
[会議ツール Zoom/Meet]
         |
         | システム音声（仮想オーディオ）
         v
+--------+------------------------------------------+
|        PC Agent App（macOS）                      |
|                                                   |
|  AudioCapture                                     |
|       |                                           |
|       | 音声チャンク（PCM）                        |
|       v                                           |
|  TranscribeClient -----> AWS Transcribe Streaming |
|       |                  （HTTPS/WebSocket）       |
|       | テキスト（最終結果）                       |
|       v                                           |
|  KeywordDetector                                  |
|       |                                           |
|       | DetectionResult                           |
|       v                                           |
|  MeetingOrchestrationService                      |
|       |                                           |
|       | DeviceCommand（JSON）                     |
|       v                                           |
|  CommandDispatcher / WSServer                     |
|       |                                           |
+-------+-------------------------------------------+
        |
        | WebSocket（同一LAN）
        | ws://192.168.x.x:8765
        v
+-------+-------------------------------------------+
|        Device App（Raspberry Pi）                 |
|                                                   |
|  WSClient                                         |
|       |                                           |
|       | DeviceCommand                             |
|       v                                           |
|  DeviceCommandService                             |
|       |                                           |
|       +---> ServoController --> サーボモーター     |
|       |     （GPIO/PWM）        （お面が頷く）     |
|       |                                           |
|       +---> LEDController   --> LED               |
|       |     （GPIO）            （点滅演出）       |
|       |                                           |
|       +---> VoicePlayer                           |
|             |                                     |
+-------------+-------------------------------------+
              |
              | HTTPS（SORACOM Beam経由）
              v
         SORACOM Beam
              |
              | AWS SigV4 署名付与
              v
         AWS Polly
              |
              | 音声データ（MP3）
              v
         VoicePlayer.play_audio()
              |
              v
         スピーカー（3.5mm/USB）

+-------------------------------------------+
|  Config Web UI（同一プロセス内スレッド）   |
|  http://localhost:8080                    |
|                                           |
|  FastAPI ←→ ConfigManager ←→ keywords.yaml|
|                |                          |
|                | ConfigSyncService        |
|                v                          |
|  MeetingOrchestrationService              |
|  （KeywordDetector ホットリロード）        |
+-------------------------------------------+
```

---

## SORACOM 構成詳細

```
+------------------+        +------------------+        +------------------+
|  Raspberry Pi    |        |  SORACOM          |        |  AWS             |
|                  |        |                  |        |                  |
|  VoicePlayer     |        |  Arc（VPN）       |        |  Polly           |
|  HTTPS リクエスト|------->|  Beam（プロキシ） |------->|  （TTS API）     |
|  （認証情報なし）|        |  SigV4 署名付与  |        |                  |
+------------------+        +------------------+        +------------------+
```

- **SORACOM Arc**: Raspberry Pi からSORACOMネットワークへのWireGuard VPN接続
- **SORACOM Beam**: HTTPSリクエストをインターセプトし、AWS SigV4署名を付与してAWS APIに転送
- **メリット**: Raspberry Pi 側にAWSアクセスキー・シークレットキーを保存不要

---

## フォールバック動作フロー

```
通常モード:
PC Agent App --[WebSocket]--> Device App
                              ServoController（コマンド駆動）

接続断検知（タイムアウト60秒）:
Device App
  FallbackManager.enter_fallback_mode()
       |
       v
  定期うなずきループ（10秒ごとに1回うなずき）
  ※ PCとの接続が復帰したら自動でフォールバック解除
```

---

## パッケージ構成（予定）

```
unazuki/
|
+-- pc_agent/                    # Unit 1: PCエージェントアプリ
|   +-- main.py                  # エントリーポイント
|   +-- audio/
|   |   +-- capture.py           # AudioCapture
|   +-- transcribe/
|   |   +-- client.py            # TranscribeClient
|   +-- detection/
|   |   +-- keyword_detector.py  # KeywordDetector
|   +-- device/
|   |   +-- ws_server.py         # WSServer
|   |   +-- dispatcher.py        # CommandDispatcher
|   +-- timer/
|   |   +-- timer_manager.py     # TimerManager
|   +-- config/
|   |   +-- config_manager.py    # ConfigManager
|   |   +-- keywords.yaml        # 設定ファイル
|   +-- services/
|   |   +-- meeting_orchestration.py
|   |   +-- config_sync.py
|   +-- web_ui/                  # Unit 3: 設定Web UI
|       +-- app.py               # FastAPI アプリ
|       +-- routers/
|       |   +-- config.py
|       |   +-- status.py
|       +-- static/
|           +-- index.html
|
+-- device/                      # Unit 2: Raspberry Pi デバイスアプリ
|   +-- device.py                # エントリーポイント
|   +-- ws/
|   |   +-- client.py            # WSClient
|   +-- hardware/
|   |   +-- servo.py             # ServoController
|   |   +-- led.py               # LEDController
|   +-- audio/
|   |   +-- voice_player.py      # VoicePlayer
|   +-- fallback/
|   |   +-- fallback_manager.py  # FallbackManager
|   +-- services/
|       +-- device_command.py    # DeviceCommandService
|
+-- requirements.txt             # PC側依存パッケージ
+-- requirements-device.txt      # Raspberry Pi側依存パッケージ
+-- README.md
```
