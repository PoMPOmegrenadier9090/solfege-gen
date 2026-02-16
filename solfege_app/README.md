# solfege_app

ソルフェージュ自動生成 Flask アプリケーションパッケージ。

音声ファイルをアップロードすると、音源分離 → 採譜 → キー推定 → ソルフェージュ生成の一連のパイプラインを実行し、MIDI・ミックス音声・ソルフェージュ JSON を生成して返します。

## パッケージ構成

```
solfege_app/
├── __init__.py       # Flask アプリケーションファクトリ (create_app)
├── config.py         # 全体設定・定数定義
├── jobs.py           # 非同期ジョブ管理
├── pipeline/         # 音声処理パイプライン (詳細は pipeline/README.md)
└── README.md         # 本ファイル
```

## モジュール詳細

### `__init__.py` — アプリケーションファクトリ

`create_app()` 関数で Flask アプリを生成します。以下のエンドポイントを提供します:

| エンドポイント | メソッド | 説明 |
|---|---|---|
| `/` | GET | Web UI (index.html) を返す |
| `/api/jobs` | POST | 音声ファイルを受け取り、フルパイプラインを**同期実行**して結果を返す |
| `/api/media/<client_id>/<job_id>/<filename>` | GET | 生成された成果物 (MIDI, WAV, JSON) をダウンロードする |

**リクエスト仕様 (`POST /api/jobs`)**:
- `file`: アップロードする音声ファイル (mp3, wav, flac, ogg, m4a, mp4)
- `client_id` (任意): クライアント識別子。省略時は `default_client`

### `config.py` — 設定・定数

アプリケーション全体で使用する設定値と定数を定義します。

| カテゴリ | 主な定数 | 説明 |
|---|---|---|
| パス設定 | `BASE_DIR`, `RUNTIME_DIR`, `CLIENTS_DIR` | プロジェクトルートとランタイムディレクトリ |
| アップロード設定 | `MAX_CONTENT_LENGTH`, `ALLOWED_EXTENSIONS` | ファイルサイズ上限 (200MB)、許可拡張子 |
| 音声パラメータ | `HOP_LENGTH`, `SAMPLE_RATE`, `SPF_THEORETICAL` | basic-pitch のフレーム関連定数 |
| キープロファイル | `MAJOR_PROFILE`, `MINOR_PROFILE`, `KEY_NAMES` | Krumhansl-Schmuckler キープロファイル |
| キー推定 HMM | `KEY_ESTIMATION_HMM_PARAMS` | キー遷移確率のパラメータ |
| 歌唱音域 | `LOWEST_PITCH`, `HIGHEST_PITCH` | basic-pitch インデックス範囲 (C2〜E6) |
| ノート推定 HMM | `PITCH_STAY_PROB`, `PITCH_TO_REST_PROB` 等 | ピッチ遷移確率 |
| モチーフ検出 | `REC_WIDTH`, `REC_MIN_LEN`, `REC_QUANTILE` | 再帰行列パラメータ |

### `jobs.py` — ジョブ管理

非同期でパイプラインを実行するためのジョブ管理機能を提供します（現在の Web エンドポイントは同期実行）。

- **`JobRecord`** — ジョブの状態を保持するデータクラス（ID、ステータス、進捗、結果など）
- **`JobManager`** — `ThreadPoolExecutor` を使用したジョブの作成・実行・状態管理クラス
  - `create_job()`: 新規ジョブを生成
  - `submit()`: ジョブをワーカースレッドに投入
  - `update()`: ジョブの進捗を更新
  - `get()`: ジョブの現在の状態を取得

## 依存関係

主要な外部ライブラリ:

- **Flask** — Web フレームワーク
- **Demucs** — 音源分離 (Meta Research)
- **basic-pitch** — ニューラルネットワークベースの音高推定 (Spotify)
- **librosa** — 音声解析・特徴量抽出
- **pretty_midi** — MIDI ファイルの生成・操作
- **audio-separator** — UVR ベースの音声分離 (オプション)
