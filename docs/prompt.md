# 開発用 LLM プロンプト

このファイルは、LLM を利用して本プロジェクトのコードを生成・更新するための指示書です。  
**docs/requirements.md を仕様のソース・オブ・トゥルース** として扱い、そこに書かれている要件に厳密に従って実装してください。

---

## 1. プロジェクト概要（LLM 向けの要約）

- プロジェクト名：空港混雑度推測・Teams自動投稿チャットボット (Weather-Driven Lite)
- 対象：新千歳空港の **翌日（06:00〜18:00）** の混雑リスク
- 目的：  
  気象庁の公開 JSON とカレンダー情報（曜日・祝日・イベント）のみを使って、0〜100 の混雑度スコアと「低・中・高」のレベルを推定し、Microsoft Teams に Adaptive Card として自動投稿する。

---

## 2. 最重要制約：外部アクセスの制限

**必ず守ること：**

- ✅ 利用してよい外部アクセス
  - 気象庁 (JMA) の公開 JSON エンドポイント（`https://www.jma.go.jp/bosai/...`）
    - 例：`/bosai/forecast/data/forecast/{area_code}.json`
    - 例：`/bosai/warning/data/warning/{pref_code}.json`
  - Microsoft Teams Incoming Webhook
  - Python パッケージインデックス（`pip` 経由でのライブラリ取得）

- ❌ 禁止すること
  - Selenium / Playwright 等のブラウザ操作
  - ログインが必要なサイトへのアクセス
  - API キー取得が有料、または煩雑なサービスの利用
  - スクレイピングが利用規約的にグレーなサイトへのアクセス

---

## 3. 実装要件（LLM が守るべき具体ルール）

### 3.1 ファイル構成

次の 4 つの Python ファイルを `src/` 配下に実装すること：

- `src/weather.py`
  - 気象庁の JSON を取得し、必要な情報（翌日の天気・警報など）をパースする
- `src/logic.py`
  - カレンダー情報（曜日・祝日・events.json）と天気情報を元に、スコアとリスクレベルを計算する
- `src/teams.py`
  - Adaptive Card の JSON を生成し、Teams Incoming Webhook に投稿する
- `src/main.py`
  - 全体のエントリーポイント
  - 「翌日」の日付を決定（JST 基準）
  - weather / logic / teams を順に呼び出す

※ 旧名 `fetcher.py` は使用せず、**必ず `weather.py`** に統一する。

### 3.2 依存ライブラリ

- 使用を前提とするライブラリ：
  - `requests`（HTTP クライアント）
  - `jpholidays`（日本の祝日判定）
  - `PyYAML`（`config/settings.yaml` の読み込み）
- Python 標準ライブラリ：
  - `datetime`, `zoneinfo`（タイムゾーン）、`json`, `logging`, `typing` など

> `holidays` ライブラリは使用せず、**祝日判定は `jpholidays` に統一** すること。

---

## 4. ロジック仕様（docs/requirements.md との整合）

### 4.1 日付・タイムゾーン

- 実行日は「現在日時」を JST（`Asia/Tokyo`）に変換し、その翌日を対象日とする。
- 対象時間帯は **翌日 06:00〜18:00**。
- GitHub Actions では、JST 17:00 実行を想定する（`cron: "0 8 * * *"` は UTC 08:00）。

### 4.2 気象データ取得 (`weather.py`)

- `requests.get` を用いて、以下を取得することを想定：
  - 天気・予報：`/bosai/forecast/data/forecast/{area_code}.json`
  - 警報・注意報：`/bosai/warning/data/warning/{pref_code}.json`
- `config/settings.yaml` に記載されたコードを使用する：
  - `jma.forecast_area_code`
  - `jma.city_code`
  - `jma.warning_pref_code`
- 取得・パース結果として、ロジック側に渡す構造体（例：`WeatherSummary` dataclass）を定義してよい。

#### 4.2.1 天気カテゴリの判定

天気テキスト（日本語）に対して、以下のように分類する：

- テキストに「暴風雪」または「大雪」を含む → カテゴリ `"storm_snow"`
- 上記以外で「雪」を含む → カテゴリ `"snow"`
- それ以外 → カテゴリ `"normal"`

警報・注意報については、warning JSON の内容から以下を抽出：

- 大雪警報 / 暴風雪警報 / 暴風警報 → `major_warning = True`
- 大雪注意報 / 風雪注意報 / 強風注意報 → `minor_warning = True`

### 4.3 カレンダー判定 (`logic.py`)

- `jpholidays.is_holiday(date)` により、対象日が日本の祝日かどうか判定する。
- 曜日（`weekday()`）を見て、平日 / 金 / 土 / 日 / 月を区別する。
- 連休中日の判定はシンプルでよい（例）：
  - 「祝日 or 日曜」に挟まれた平日を連休中日と見なす。

- `config/events.json` を読み込み、対象日がいずれかのイベント期間に含まれていれば、その `score_bonus` 分をイベント加点とする。
  - ファイルが存在しない、または空配列の場合は、イベント加点 0 とする。

### 4.4 スコアリング

**docs/requirements.md に記載のスコア式を厳守すること。**

- 混雑度スコア (0〜100)：

  ```text
  raw_score = (基礎需要点 + イベント加点) × 気象係数
  score = min(100, max(0, raw_score))
  ```

- 基礎需要点（Base）の例：
  - 平日（月〜木）：30
  - 金曜・日曜・月曜：50
  - 土曜：40

- イベント加点（Event）の例：
  - 連休中日：+20
  - 雪まつりなど大型イベント：+30（events.json の `score_bonus` を使用）

- 気象係数（Weather Multiplier）：

  1. 天気テキストベース
     - `"normal"`：×1.0
     - `"snow"`：×1.2
     - `"storm_snow"`：×1.8
  2. 警報ベース
     - major_warning（大雪・暴風雪・暴風の警報）：最低でも ×1.8
     - minor_warning（大雪注意報・風雪注意報・強風注意報）：最低でも ×1.2

  → 天気カテゴリによる係数と、警報による係数の **大きい方** を採用すること。

- レベル判定：

  ```text
  0〜39  → "低"
  40〜69 → "中"
  70〜100 → "高"
  ```

スコアおよびレベル、また「どの要因が効いたか（天気・警報・連休・イベントなど）」を示すテキスト or リストを戻り値に含めること。

---

## 5. Teams 投稿仕様 (`teams.py`)

### 5.1 投稿方式

- Microsoft Teams Incoming Webhook を使用する。
- Webhook URL は `TEAMS_WEBHOOK_URL` 環境変数から取得する。
- 変数が未設定の場合は、例外を投げるか、標準出力に警告を出して終了する。

### 5.2 Adaptive Card

- 基本構造：

  ```json
  {
    "$schema": "http://adaptivecards.io/schemas/adaptive-card.json",
    "type": "AdaptiveCard",
    "version": "1.4",
    "body": [...],
    "actions": [...]
  }
  ```

- 最低限含める要素：
  - タイトル：「新千歳空港 混雑リスク予測（翌日）」など
  - 対象日（YYYY-MM-DD）
  - 混雑レベル（低・中・高）
  - スコア（数値）
  - 予想天気（概要テキスト）
  - 主なリスク要因（箇条書き）
    - 例：「大雪警報」「連休中日」「雪まつり期間」など
  - 説明コメント（例）：
    - 「明日は大雪警報の可能性があるため、JR運休リスクを含め混雑レベル『高』と予測されます。」
  - 参考リンクボタン：
    - 気象庁天気予報
    - 気象庁警報ページ
    - 新千歳空港公式サイト

- エラー時：
  - 気象庁からのデータ取得や解析に失敗した場合は、
    - 「データ取得エラー」などのタイトル
    - 原因の概要（例：「気象庁予報 JSON の取得に失敗」）
    - 参考リンク（手動で天気を確認できる URL）
    を記載した簡易カードを送信すること。

---

## 6. main.py の役割

`src/main.py` は、次のような流れを実装する：

1. ログ設定の初期化
2. タイムゾーンを JST に設定し、「翌日」の日付を求める
3. `settings.yaml` と `events.json` を読み込む
4. `weather.py` から翌日の天気・警報情報を取得
5. `logic.py` でスコアとレベル、およびリスク要因リストを算出
6. `teams.py` で Adaptive Card を生成し、Teams Webhook に投稿
7. エラーが発生した場合は、可能であればエラー用のカードを Teams に通知

---

## 7. 成果物（LLM が生成すべきファイル）

LLM にコード生成を依頼する際は、次のファイルを対象とする：

- Python コード一式
  - `src/weather.py`
  - `src/logic.py`
  - `src/teams.py`
  - `src/main.py`
- 依存ライブラリ定義
  - `requirements.txt`
- GitHub Actions ワークフローファイル
  - `.github/workflows/run.yml`

> これらを生成・更新する際は、**常に docs/requirements.md の最新版を参照し、一貫性を保つこと**。  
> README.md やこの prompt.md との矛盾がある場合は、requirements.md を優先する。

---

## 8. コーディングスタイル

- Python 3.11 想定・型ヒント（type hints）は可能な範囲で付与すること
- Docstring は最低限でよいが、外部公開 API（関数・クラス）には簡潔な説明を書くこと
- ログ出力には `logging` モジュールを使用し、print は極力避けること

---

## 9. LLM への追加指示の例

実際に LLM に依頼する際は、例えば次のようなプロンプトを追加で与えるとよい：

> - docs/requirements.md と docs/prompt.md に従って、まずは `src/weather.py` のみ実装してください。  
> - 実際の JMA JSON の詳細構造が分からない部分は、コメントで「ここは JMA 公開仕様に合わせて後で調整」と書きつつ、仮のパースロジックを書いてください。  
> - 例外処理とログ出力も最低限含めてください。

このようにして、段階的に `weather.py` → `logic.py` → `teams.py` → `main.py` とファイルを増やしていきます。
