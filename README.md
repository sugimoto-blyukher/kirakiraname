# キラキラネームメーカー（Kirakira Name Maker）

ESP32から送られた入力をもとに、Google Geminiで「キラキラネーム」を生成し、命名書のPDFを作成・印刷するシステムです。

P2HACKS Pre-PBLで制作したチーム開発作品です（CyberAgent賞）。**このリポジトリではバックエンド部分を公開しています。**

## システム概要

```text
ESP32（5ビットの入力）
    ↓ HTTP
FastAPI
    ↓
Gemini API（名前・読み仮名を生成）
    ↓
Jinja2 + WeasyPrint（命名書PDF）
    ↓
CUPS / lp（プリンタに送信）
```

印刷要求はジョブとして管理し、進行状況や生成済みPDFをAPIから取得できます。

### 主な機能

- **名前生成**：5個の0/1の入力から、名前の雰囲気・テーマなどを選んでGeminiに生成を依頼
- **PDF生成**：生成された名前と読み仮名をHTMLテンプレートに埋め込み、命名書をPDF化
- **印刷**：CUPSの `lp` コマンドを使用してPDFを印刷
- **ジョブ管理**：受付・生成・印刷の各段階をステータスとしてファイルに記録
- **重複印刷対策**：`device_id` と `idempotency_key` の組み合わせで既存ジョブを再利用

## 技術スタック

| 用途 | 技術 |
| --- | --- |
| APIサーバー | Python / FastAPI / Pydantic |
| 生成AI | Google Gemini API |
| テンプレート | Jinja2 |
| PDF生成 | WeasyPrint |
| 印刷 | CUPS（`lp`） |
| ジョブ・生成結果の保存 | ローカルファイル（JSON / PDF） |
| テスト | pytest / FastAPI TestClient |

## 入力と生成処理

ESP32から渡される `payload` は、**5個の0/1** を表す文字列です。例えば `"[1,1,0,0,1]"` または `"11001"` を受け取れます。

現行実装では、先頭4ビットを以下のように解釈します。

| ビット | 値と意味 |
| --- | --- |
| 1番目 | `0`：女性寄り / `1`：男性寄り |
| 2番目 | `0`：現代風 / `1`：和風 |
| 3・4番目 | `00`：ファンタジー / `01`：外国語由来 / `10`：事象 / `11`：アニメ |
| 5番目 | 生成プロンプトに追加する条件の切り替え（現状は調整中） |

Geminiの返答から `name`（漢字の名前）と `ruby`（ひらがなの読み仮名）を検証し、PDF生成に利用します。

## セットアップ

### 動作環境

- Python 3.13系を想定（CIで指定されているバージョン）
- Gemini APIキー
- 実際に印刷する場合は、CUPSおよび利用可能なプリンタ
- WeasyPrintが必要とするシステムライブラリ（OSによって異なります）

```bash
git clone https://github.com/sugimoto-blyukher/kirakiraname.git
cd kirakiraname/backend

python3 -m venv .venv
source .venv/bin/activate

python -m pip install -r requirements.txt
# 現行requirements.txtに不足しているSDKを追加
python -m pip install google-genai
```

`backend/.env` を作成します。

```dotenv
GEMINI_API_KEY=your_gemini_api_key
GEMINI_MODEL=gemini-2.5-flash
# 利用するプリンタを指定する場合のみ設定
# PRINTER_NAME=your_printer_name
```

```bash
bash scripts/dev.sh
```

APIはデフォルトで `http://localhost:8000` に起動します。FastAPIのAPIドキュメントは `http://localhost:8000/docs` から確認できます。

**注意：** 印刷処理は `lp` を実行します。CUPSやプリンタが未設定の場合、PDFが生成できてもジョブは `PRINT_FAILED` になることがあります。

## API

| Method | Endpoint | 内容 |
| --- | --- | --- |
| `GET` | `/health` | ヘルスチェック |
| `POST` | `/v1/print` | 名前生成・PDF生成・印刷ジョブを登録 |
| `GET` | `/v1/jobs/{job_id}` | ジョブの進行状況を取得 |
| `GET` | `/v1/download/{job_id}` | 生成済みPDFをダウンロード |

### 印刷ジョブの登録

```bash
curl -X POST http://localhost:8000/v1/print \
  -H "Content-Type: application/json" \
  -d '{
    "device_id": "esp32-01",
    "idempotency_key": "print-request-0001",
    "payload": "[1,1,0,0,1]",
    "template_id": "default",
    "copies": 1
  }'
```

レスポンス例：

```json
{
  "job_id": "<発行されたUUID>",
  "status": "QUEUED"
}
```

`idempotency_key` は8〜200文字、`copies` は1〜20の範囲です。同一の `device_id` と `idempotency_key` で再送した場合は、新規ジョブを作らず既存のジョブIDと状態を返します。

### 状態確認とPDF取得

```bash
curl http://localhost:8000/v1/jobs/<job_id>
curl -o result.pdf http://localhost:8000/v1/download/<job_id>
```

主な状態遷移は次のとおりです。

```text
RECEIVED → QUEUED → LLM_PROCESSING → RENDERING → PRINTING → PRINTED
                          ↓              ↓            ↓
                      LLM_FAILED     RENDER_FAILED  PRINT_FAILED
```

失敗時にはジョブ情報に `error` が記録されます。

## ディレクトリ構成

```text
backend/
├── app/
│   ├── main.py           # APIとジョブ処理
│   ├── gemini_client.py  # 入力の解釈・Gemini API呼び出し
│   ├── render.py         # PDF生成
│   ├── print_service.py  # 印刷スクリプトの実行
│   ├── store.py          # ジョブと冪等性キーの保存
│   ├── models.py         # リクエスト・レスポンスモデル
│   └── config.py         # 環境変数
├── templates/            # 命名書のHTMLテンプレート
├── scripts/              # 開発・テスト・印刷用スクリプト
├── test/                 # pytest
├── data/                 # 実行時データ（ジョブ・JSON・PDF）
└── requirements.txt
```

## テスト

`backend/` ディレクトリで実行します。

```bash
bash scripts/test.sh
# または
pytest -q
```

ヘルスチェック、APIの重複リクエスト処理、PDFダウンロード、印刷コマンド呼び出し、ファイル保存などのテストコードを含みます。外部サービスや印刷処理は一部のテストでモック化しています。

## 実装上の注意点・今後の課題

- **依存関係**：コードは `from google import genai` を使用していますが、現行 `requirements.txt` は `google.generativeai` を指定しています。起動前に `google-genai` の追加インストールが必要です。
- **テストの追従**：一部のPDFテストでは旧モデルのフィールド名や旧テンプレート名を参照しており、現行実装に合わせた修正が必要です。テスト全件の成功は保証していません。
- **テンプレートのフォント参照**：HTML内のフォントパスとリポジトリ上の実ファイル配置は、環境に応じた確認・調整が必要です。
- **ジョブ実行基盤**：処理にはFastAPIの `BackgroundTasks` とローカルファイル保存を使用しています。永続キューや分散ワーカーを備えた構成ではありません。

## 開発について

チーム制作のうち、本リポジトリの**バックエンド実装を担当**しました。API、Gemini連携、PDF生成・印刷処理を統合しています。一部の実装にはAIコーディング支援ツールを利用しています。
