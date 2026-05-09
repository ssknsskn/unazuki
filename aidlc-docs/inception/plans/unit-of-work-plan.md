# ユニット・オブ・ワーク計画 — Unazu-kiro

## 前提コンテキスト

アプリケーション設計で3コンポーネントが確定済み：
- **PC Agent App** — 音声キャプチャ・文字起こし・キーワード検知・コマンド送信・設定Web UIホスト
- **Device App** — Raspberry Pi 上のサーボ制御・音声再生・LED制御・フォールバック
- **Config Web UI** — FastAPI + HTML/JS の設定ダッシュボード（PC Agent App と同一プロセス）

## 生成チェックリスト

- [x] unit-of-work.md — ユニット定義・責務・コード構成
- [x] unit-of-work-dependency.md — 依存関係マトリクス・開発順序
- [x] unit-of-work-story-map.md — 機能要件とユニットのマッピング

---

## 確認質問

アプリケーション設計と実行計画から多くが確定していますが、以下の2点だけ確認します。

---

### Q1: Unit 3（Config Web UI）の独立性

Config Web UI は PC Agent App と同一プロセスで動作しますが、開発・テストの単位としてはどう扱いますか？

A) 独立したユニット（Unit 3）として扱う — 設計・コード生成を別ステップで実施
B) Unit 1（PC Agent App）の一部として扱う — まとめて設計・実装
X) その他（[Answer]: タグの後に記述してください）

[Answer]: A

---

### Q2: ユニットの開発順序

3ユニットの開発順序はどうしますか？

A) Unit 1 → Unit 2 → Unit 3（PCアプリ基盤を先に作り、デバイス、UIの順）
B) Unit 1 → Unit 3 → Unit 2（PCアプリ + 設定UIを先に完成させ、デバイスは後）
C) Unit 2 → Unit 1 → Unit 3（ハードウェア制御を先に確認してからソフトを作る）
X) その他（[Answer]: タグの後に記述してください）

[Answer]: B

---

回答が完了したら「完了しました」とお知らせください。
