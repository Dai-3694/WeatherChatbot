# WeatherChatbot
# README：空港混雑度推測・Teams自動投稿チャットボット (Weather-Driven Lite)

## 📌 プロジェクト概要

新千歳空港の **翌日（06:00〜18:00）混雑度** を、**気象予報とカレンダー情報のみ** を用いて推測し、Microsoft Teams へ自動投稿するボットです。

- 気象庁（JMA）の **公開 JSON** のみを利用し、API キーやログインは一切不要
- 航空会社の便数・予約数、リアルタイム交通情報などは **一切利用しないシンプルなモデル**
- GitHub Actions 上で **無料・ほぼメンテナンスフリー** で運用できる Lite 版です

詳細な要件は `docs/requirements.md` を参照してください。

---

## ✅ 特徴

- **API キー・ログイン不要**
  - 利用する外部サービスは
    - 気象庁の公開 JSON（`www.jma.go.jp/bosai/...`）
    - Microsoft Teams Incoming Webhook
  - のみです。

- **シンプルなロジック**
  - 「曜日・祝日・イベント」による **基礎需要**
  - 「雪・暴風雪・大雪警報」などによる **気象リスク**
  - を組み合わせて 0〜100 のスコアを算出し、
    - 低 / 中 / 高 の 3 段階で混雑レベルを出します。

- **低コスト運用**
  - GitHub Actions のスケジュール実行を利用し、**毎日 JST 17:00** に自動判定・投稿
  - 実行コストは GitHub の無料枠の範囲を想定

---

## 🛠 システム構成

```text
[気象庁データ (JMA)] + [カレンダー情報 (jpholidays + config/events.json)]
                    ⬇
             [推測エンジン (Python)]
        （基礎需要 × 気象係数 = スコア）
                    ⬇
         [Teams Webhook (Adaptive Card)]
```

---

## 📂 ディレクトリ構成

```text
.
├── src
│   ├── weather.py      # 気象庁データの取得とパース
│   ├── logic.py        # 曜日・イベント・天気に基づくスコア計算
│   ├── teams.py        # Adaptive Card 生成と Teams 投稿
│   └── main.py         # エントリーポイント
├── config
│   ├── events.json     # 「雪まつり」などの特定イベント期間定義（任意・空でも可）
│   └── settings.yaml   # 地域コード・スコア閾値・リンクURL 等の設定
├── docs
│   └── requirements.md # 要件定義書（本プロジェクトの仕様のソース）
├── requirements.txt
└── .github
    └── workflows
        └── run.yml     # GitHub Actions ワークフロー（毎日 JST 17:00 実行）
```

> 📌 実装コードは `src/*.py` にまとめ、構成は README / requirements / prompt 間で統一します。  
> 旧記載の `fetcher.py` は使用せず、`weather.py` に統一しています。

---

## ⚙️ セットアップ

### 1. 必要環境

- Python 3.11 以上（目安）
- インターネット接続（気象庁と Teams Webhook にアクセスできること）

### 2. 依存ライブラリのインストール

```bash
pip install -r requirements.txt
```

`requirements.txt` の想定内容（例）：

```text
requests
jpholidays
PyYAML
```

### 3. 設定ファイルの準備

#### 3-1. `config/settings.yaml`

地域コード・スコア閾値・リンク URL などの設定を記述します。

```yaml
jma:
  forecast_area_code: "016000"   # 例：石狩・空知・後志地方（実コードは適宜確認）
  city_code: "0121800"           # 例：千歳市のコード（実コードは適宜確認）
  warning_pref_code: "01"        # 北海道

scoring:
  low_threshold: 0     # 0〜39 → 低
  medium_threshold: 40 # 40〜69 → 中
  high_threshold: 70   # 70〜100 → 高

links:
  jma_forecast: "https://www.jma.go.jp/bosai/forecast/"
  jma_warning:  "https://www.jma.go.jp/bosai/warning/"
  airport_official: "https://www.new-chitose-airport.jp/"
  # 追加の参考リンクがあればここに追記
```

※ 実際の JMA コードは、開発時に公式ドキュメント等で確認してください。

#### 3-2. `config/events.json`

特定イベント（さっぽろ雪まつり、年末年始、春節など）の期間を記述します。

```json
[
  {
    "name": "Sapporo Snow Festival",
    "start": "2025-02-01",
    "end": "2025-02-11",
    "score_bonus": 30
  }
]
```

- `start` / `end` は ISO8601 形式の日付（YYYY-MM-DD）
- `score_bonus` はその期間中に加算するイベント加点
- **このファイルが空 or 存在しない場合は、イベント加点は 0 として動作** します（= メンテなしでも動く設計）。

---

### 4. Teams Webhook 設定

1. Teams で投稿先チャンネルを開く
2. 「コネクタ」から「Incoming Webhook」を追加
3. Webhook 名とアイコンを設定し、生成された URL をコピー
4. GitHub Actions の Repository Secrets などに保存し、環境変数として利用

ローカルで試す場合は、シェルから:

```bash
export TEAMS_WEBHOOK_URL="https://outlook.office.com/webhook/..."
```

のように設定します。

---

## ▶️ 実行方法

### ローカルでの実行

```bash
python -m src.main
```

- `main.py` 内で
  - 翌日の日付（JST 基準）
  - 気象庁 JSON の取得
  - スコア算出
  - Teams Webhook への投稿
  をまとめて実行します。

### GitHub Actions でのスケジュール実行

`.github/workflows/run.yml` の例：

```yaml
name: Run Airport Congestion Bot

on:
  schedule:
    # 毎日 JST 17:00 = UTC 08:00
    - cron: "0 8 * * *"
  workflow_dispatch: {}

jobs:
  run-bot:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: "3.11"

      - name: Install dependencies
        run: |
          pip install -r requirements.txt

      - name: Run bot
        env:
          TEAMS_WEBHOOK_URL: ${{ secrets.TEAMS_WEBHOOK_URL }}
        run: |
          python -m src.main
```

---

## 🔢 スコアリング概要（ざっくり）

詳細は `docs/requirements.md` に記載されていますが、概要は以下の通りです。

- **混雑度スコア (0–100)**

  ```text
  raw_score = (基礎需要点 + イベント加点) × 気象係数
  score = min(100, max(0, raw_score))
  ```

- **基礎需要点（例）**
  - 平日（月〜木）: 30
  - 金・日・月: 50
  - 土: 40

- **イベント加点（例）**
  - 連休中日: +20
  - 雪まつり期間: +30

- **気象係数（例）**
  - 晴れ/曇り/雨: ×1.0
  - 雪（テキストに「雪」）: ×1.2
  - 暴風雪/大雪警報あり: ×1.8

- **レベル判定**
  - 0〜39: 低
  - 40〜69: 中
  - 70〜100: 高

---

## ⚠️ 制約事項

- 実際の便数・予約数は考慮しません（曜日・イベントによる重み付けのみ）
- リアルタイムの JR / 高速道路の運行情報は取得しません
  - 「暴風雪なら交通麻痺リスク大」としてスコアを上乗せします
- 気象庁の JSON 仕様が大きく変わった場合は、パースロジックの修正が必要になる可能性があります

---

## 🧩 開発者向け

- 実装時は `docs/requirements.md` を **仕様のソース・オブ・トゥルース** とし、README は概要として扱ってください。
- コード自動生成に LLM を使う場合は、`docs/prompt.md` の手順・制約に従ってください。
