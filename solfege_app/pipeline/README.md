# solfege_app.pipeline

音声ファイルからソルフェージュを生成するための音声処理パイプラインパッケージ。

各モジュールはパイプラインの1ステップに対応しており、`orchestrator.py` がこれらを組み合わせて全体のワークフローを制御します。

## パイプライン全体の流れ

```
入力音声 (.mp3/.wav)
  │
  ▼
┌─────────────────────┐
│ 1. separation       │  Demucs による音源分離
│    (vocals/bass/    │  → ボーカル・ベース・ドラムス・その他
│     drums/other)    │
└─────────┬───────────┘
          ▼
┌─────────────────────┐
│ 2. transcription    │  チューニング補正・HPSS・basic-pitch 推論
│    (ステップ1-4)     │  → フレーム単位のピッチ/オンセット特徴量
└─────────┬───────────┘
          ▼
┌─────────────────────┐
│ 3. beat_grid        │  テンポ・ビート検出 + グリッド量子化
│                     │  → ビートグリッド上の特徴量
└─────────┬───────────┘
          ▼
┌─────────────────────┐
│ 4. key_estimation   │  クロマ + HMM Viterbi によるキー推定
│                     │  → 時変キー系列 + グローバルキー
└─────────┬───────────┘  → ピッチ/オンセット特徴量の補正
          ▼
┌─────────────────────┐
│ 5. motif            │  再帰行列によるモチーフ検出・補正
│                     │  → ピッチ特徴量の補正
└─────────┬───────────┘
          ▼
┌─────────────────────┐
│ 6. note_assignment  │  特徴量に基づくノート割り当て + MIDI 生成
│                     │  → MIDI ノート列
└─────────┬───────────┘
          ▼
┌─────────────────────┐
│ 7. solfege          │  移動ド変換
│                     │  → 各ノートにソルフェージュラベルを付与
└─────────┴───────────┘
          ▼
出力: MIDI / ミックス WAV / solfege.json
```

## モジュール構成

```
pipeline/
├── __init__.py          # パッケージ初期化 (空)
├── orchestrator.py      # パイプライン全体の制御
├── separation.py        # 音源分離
├── transcription.py     # 採譜パイプライン統合
├── beat_grid.py         # ビートグリッド量子化
├── key_estimation.py    # キー推定
├── motif.py             # モチーフ抽出・補正
├── note_assignment.py   # ノート割り当て・MIDI 生成
├── solfege.py           # ソルフェージュ（移動ド）生成
└── README.md            # 本ファイル
```

## 各モジュールの詳細

---

### `orchestrator.py` — パイプライン オーケストレータ

パイプライン全体の実行順序を制御するエントリーポイント。

- **`run_full_pipeline(job_id, update_job, client_id, job_dir, input_path)`**
  - Demucs 分離 → 採譜パイプラインの順に呼び出す
  - 各ステップの進捗を `update_job` コールバックで通知
  - 結果として推定キー、ノート数、メディアパスなどの辞書を返す

---

### `separation.py` — 音源分離

Demucs および UVR を用いた音源分離処理。

#### 主要クラス・関数

| 名前 | 種別 | 説明 |
|---|---|---|
| `DemucsSeparationResult` | dataclass | htdemucs の4ステム出力パス (vocals, drums, bass, other) |
| `run_separation_by_demucs()` | 関数 | Demucs (htdemucs) を実行し分離結果を返す |
| `AudioSeparationPipeline` | クラス | 音声分離処理と関連パスを一元管理 |
| `SeparationArtifacts` | dataclass | 全分離処理の成果物パスを格納 |

#### `AudioSeparationPipeline` のメソッド

| メソッド | 説明 |
|---|---|
| `run_demucs()` | Demucs による4ステム分離 |
| `run_instrumental_separation()` | UVR (`UVR_MDXNET_KARA`) によるボーカル/インスト分離 |
| `run_echo_and_reverb_removal()` | UVR (`DeEcho-DeReverb`) によるエコー・リバーブ除去 |
| `run_all()` | 上記3ステップを順に実行 |

---

### `transcription.py` — 採譜パイプライン

全採譜ステップを統合する中心モジュール。各サブモジュールを順に呼び出します。

- **`run_transcription(demucs_result, artifacts_dir, original_audio_path)`**

**処理ステップ:**

1. **チューニング補正**: `librosa.estimate_tuning` で検出しリサンプリングで補正
2. **HPSS**: ボーカルのハーモニック成分を抽出
3. **テンポ・ビート検出**: `beat_grid.detect_beats` で BPM とビート位置を取得
4. **basic-pitch 推論**: ニューラルネットワークでフレーム単位の onset/note 特徴量を生成
5. **ビートグリッド量子化**: `beat_grid.quantize_to_grid` でフレーム→グリッドに変換
6. **キー推定**: `key_estimation.estimate_key_sequence` で時変キー系列を推定
7. **スケールバイアス**: `key_estimation.apply_scale_bias` でスケール構成音を優遇
8. **モチーフ補正**: `motif.extract_motifs` + `motif.apply_motif_correction`
9. **ノート割り当て**: `note_assignment.greedy_note_assignment` で MIDI ノート列を生成
10. **MIDI 保存・ミックス**: MIDI ファイル出力とインスト音源とのミックス WAV 生成
11. **ソルフェージュ生成**: `solfege.attach_solfege` で各ノートに移動ドラベルを付与

**出力成果物:**
- `vocals_transcribed.mid` — 採譜結果の MIDI ファイル
- `vocals_transcribed_with_inst.wav` — 採譜音 + インスト音源のミックス
- `solfege.json` — ノート・キー・ソルフェージュの JSON データ

---

### `beat_grid.py` — ビートグリッド量子化

basic-pitch のフレーム単位の出力をビートグリッドに量子化するモジュール。

| 関数 | 説明 |
|---|---|
| `center_weighted_agg_func(array, axis)` | ハミング窓による中心加重集約関数 |
| `detect_beats(y, sr, hop_length)` | librosa を使用したテンポ・ビート位置の検出 |
| `quantize_to_grid(data, beat_times_sec, spf, subdivisions, agg_func)` | フレームデータをビートグリッドに量子化。各ビートを `subdivisions` (デフォルト4) 分割 |

**量子化の仕組み:**
- ビート間を等分割（デフォルト: 16分音符単位）
- 各グリッド区間内のフレームデータを集約関数で集約
- 実時間ベースの処理により正確なタイミングを保持

---

### `key_estimation.py` — キー推定

クロマベクトルと HMM Viterbi デコーディングにより、時間変化するキー系列を推定するモジュール。

| 関数/クラス | 説明 |
|---|---|
| `KeyEstimate` | 推定キー情報のデータクラス (root_pc, mode, label, key_index) |
| `estimate_key_sequence(y_original, sr, spf, beat_times_sec, ...)` | 時変キー系列とグローバルキーを推定 |
| `get_scale_mask(key_idx, n_pitches, midi_offset)` | スケール構成音に基づく重みマスクを生成 |
| `apply_scale_bias(grid_notes, grid_onsets, key_sequence)` | キー系列に基づきノート確率にバイアスを適用 |

**推定アルゴリズム:**
1. CQT クロマ特徴量を抽出
2. ビートグリッドに量子化 → ガウシアン平滑化
3. ブロック化して Krumhansl-Schmuckler プロファイルとのピアソン相関を計算
4. Viterbi アルゴリズムで最適キー系列をデコード
5. 24キー (12 Major + 12 Minor) の遷移行列は属調・下属調・平行調・同主調の関係を考慮

---

### `motif.py` — モチーフ抽出・補正

再帰行列 (Recurrence Matrix) からモチーフ区間を検出し、ノート確率を相互補正するモジュール。

| 関数 | 説明 |
|---|---|
| `extract_motifs(biased_notes)` | ノート確率から再帰行列を構築しモチーフ区間を抽出 |
| `apply_motif_correction(biased_notes, motif_segments, w_self, w_motif)` | モチーフ対応に基づくノート確率の補正 |

**処理の流れ:**
1. 歌唱音域に限定 → ガウシアン平滑化 → L2 正規化
2. `librosa.segment.recurrence_matrix` でコサイン類似度ベースの再帰行列を構築
3. 対角線上の連続区間（モチーフペア）を検出
4. 重複しないモチーフ対を選択
5. 対応するモチーフ間でノート確率を重み付き平均で補正 (デフォルト: self=0.6, motif=0.4)

---

### `note_assignment.py` — ノート割り当て・MIDI 生成

グリッドレベルの特徴量から最終的な MIDI ノート列を生成するモジュール。

| 関数 | 説明 |
|---|---|
| `greedy_note_assignment(grid_onsets, grid_notes, grid_times, ...)` | グリーディなノート割り当て。オンセット/ピッチの閾値判定でノートの開始・終了を決定 |
| `note_list_to_midi(midi_notes)` | `pretty_midi.Note` のリストから PrettyMIDI オブジェクトを生成 |

**割り当てロジック:**
- オンセット検出 → 新規ノート開始
- ピッチ変化 or ピッチ消失 → 現在のノート終了
- 連続する同一ピッチは1つのノートとして結合

---

### `solfege.py` — ソルフェージュ生成

推定キーに基づく移動ド（ドレミ）変換モジュール。

| 関数 | 説明 |
|---|---|
| `key_index_to_root_pc(key_index)` | キーインデックス → ルートピッチクラス (0–11) |
| `key_index_is_minor(key_index)` | マイナーキー判定 |
| `key_index_to_solfege_root(key_index)` | ソルフェージュ用ルートを返す（マイナーは平行長調基準） |
| `key_index_to_label(key_index)` | キーラベル文字列 (例: "C Major") |
| `pitch_to_movable_do(pitch, key_root_pc)` | MIDI ピッチ → 移動ドラベル (ド/レ/ミ...) |
| `attach_solfege(note_events, key_sequence, grid_times)` | ノートイベントにソルフェージュラベルとキー情報を付与 |

**移動ドの方針:**
- **メジャーキー**: トニックが「ド」
- **マイナーキー**: 平行長調のトニックを「ド」とする（例: A minor → C が「ド」）
- 日本語表記: ド / ド# / レ / レ# / ミ / ファ / ファ# / ソ / ソ# / ラ / ラ# / シ
