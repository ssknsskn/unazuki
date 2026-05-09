# 🤖 Unazu-kiro（全自動うなずきマシーン）

> オンライン会議で、中身を聞かずに評価だけを得るための「抜け殻」生成装置

---

## 概要

**Unazu-kiro** は、ソフトとハードを融合した全自動うなずきデバイスです。  
PCに常駐するAIが会議音声をリアルタイム解析し、手元のRaspberry Piが物理的なお面を動かして「納得感のある頷き」を自動演出します。あなたは何もしなくていい。

```
[会議ツール Zoom/Meet]
        |
        | システム音声
        v
[PC常駐Pythonアプリ] ──→ [AWS Transcribe（音声認識）]
        |
        | キーワード検知・コマンド送信（WebSocket）
        v
[Raspberry Pi]
        |
        +──→ サーボモーター → お面が頷く
        +──→ スピーカー    → AWS Polly で相槌を発話
        +──→ LED           → 忖度モード発動を演出
```

---

## 「だめ人間」機能一覧

| 機能 | 説明 |
|------|------|
| 🚀 **忖度ブーストモード** | 「社長」「予算」「重要」などのキーワードを検知すると、デバイスがマッハで頷き始める |
| 😴 **生存確認フェイク** | 60秒以上無発言が続くと「あー、なるほど…」とスピーカーから独り言を流し、寝落ちを隠蔽 |
| 📡 **褒め言葉レーダー** | 自分の名前が呼ばれると超高速でうなずく |
| 🎲 **ランダム相槌ルーレット** | 定期的に「おっしゃる通りです」「確かに…」をランダム発話して存在感を演出 |

---

## システム構成

```
pc_agent/          # Unit 1: PCエージェントアプリ（macOS）
├── main.py        # エントリーポイント（python main.py で起動）
├── audio/         # システム音声キャプチャ
├── transcribe/    # AWS Transcribe Streaming 連携
├── detection/     # キーワード・名前検知エンジン
├── device/        # WebSocket サーバー・コマンド送信
├── timer/         # 無発言タイマー・相槌タイマー
├── config/        # 設定管理（keywords.yaml）
├── services/      # オーケストレーション・設定同期
└── web_ui/        # Unit 3: 設定ダッシュボード（localhost:8080）

device/            # Unit 2: Raspberry Pi デバイスアプリ
├── device.py      # エントリーポイント（python device.py で起動）
├── ws/            # WebSocket クライアント
├── hardware/      # サーボ・LED 制御（GPIO/PWM）
├── audio/         # AWS Polly 音声再生（SORACOM Beam経由）
├── fallback/      # 接続断時の自律フォールバック
└── services/      # コマンドルーティング
```

---

## 技術スタック

| レイヤー | 技術 |
|---------|------|
| 音声認識 | AWS Transcribe Streaming API |
| 音声合成 | AWS Polly |
| PCアプリ | Python 3.11+、sounddevice、boto3、websockets |
| 設定UI | FastAPI + 素のHTML/CSS/JS |
| ハードウェア | Raspberry Pi 4/5、サーボモーター、LED |
| サーボ制御 | RPi.GPIO / pigpio（PWM） |
| デバイス接続 | SORACOM Arc + SORACOM Beam（AWS認証情報をデバイスに置かない） |
| PC↔デバイス通信 | WebSocket（同一LAN） |

---

## セットアップ

### 必要なもの

- macOS PC（会議参加用）
- Raspberry Pi 4 または 5
- サーボモーター × 1（お面の首振り用）
- USB スピーカーまたは 3.5mm スピーカー
- LED × 1（忖度ブースト演出用）
- AWSアカウント（Transcribe・Polly 利用）
- SORACOMアカウント（Arc + Beam 設定）

### PC側のセットアップ

```bash
# 1. 依存パッケージをインストール
pip install -r requirements.txt

# 2. 環境変数を設定
cp .env.example .env
# .env を編集して AWS 認証情報・デバイスIPを記入

# 3. 起動
python pc_agent/main.py
# → http://localhost:8080 で設定ダッシュボードが開く
```

### Raspberry Pi 側のセットアップ

```bash
# 1. 依存パッケージをインストール
pip install -r requirements-device.txt

# 2. SORACOM Arc の設定（別途 SORACOM コンソールで設定）

# 3. 起動
python device/device.py
```

### キーワードのカスタマイズ

`pc_agent/config/keywords.yaml` を編集するか、ブラウザで `http://localhost:8080` を開いて設定ダッシュボードから変更できます。

```yaml
boost_keywords:
  - 社長
  - 重要
  - 予算
  - 決定

name_triggers:
  - あなたの名前

aizuchi_phrases:
  - あー、なるほど…
  - 確かに…
  - おっしゃる通りです

silence_threshold_sec: 60   # 無発言フェイクの閾値（秒）
aizuchi_interval_sec: 90    # 相槌ルーレットの間隔（秒）
```

---

## 設計ドキュメント

AI-DLC（AI駆動開発ライフサイクル）を使って設計しました。詳細は `aidlc-docs/` を参照してください。

- [要件定義書](aidlc-docs/inception/requirements/requirements.md)
- [アプリケーション設計書](aidlc-docs/inception/application-design/application-design.md)
- [ユニット定義](aidlc-docs/inception/application-design/unit-of-work.md)
- [実行計画](aidlc-docs/inception/plans/execution-plan.md)

---

## ライセンス

MIT
