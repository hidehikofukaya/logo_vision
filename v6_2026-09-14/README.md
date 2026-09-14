# logo_vision v6 アーカイブ（2026-09-14）

刻印ブロックの Bbox 切り出しシステムの**ベースライン（v6）**一式。他の PC で試すための持ち出し用。

## 中身

| フォルダ | 内容 |
|---|---|
| `01_設計書/` | **`08_システム設計書_v6.md`（まずこれ）**: プロジェクト概要・データ生成・網の構造・学習・評価・再現手順。`07_学習によるBbox生成.md` は経緯の記録、`CLAUDE.md` はプロジェクトの前提 |
| `02_学習済みモデル/` | **`v6H.pt`**（推論はこれ 1 つで動く）、`log_v6H.txt`（学習ログ）、`vote_P0_bb.pt` / `vote_P0.pt`（**学習し直すとき**の初期値） |
| `03_画像データ/` | **`ds_v4.tar`**（データセット 1024×768、228 シーン / 8,773 枚、EXR + index.json + シーン npz）、`評価画像/`（材質別の結果画像・失敗分析・確認用シート・数値）。**GitHub には含めない（別途受け渡し）** |
| `04_コードと素材/` | `code_tools_blender.tar.gz`（コード）、`assets_hdri_props.tar.gz`（HDRI・小道具・ロゴ。データ生成に必要）、`step_files.tar.gz`（実部品の STEP 140 個） |

## 結果（v6、合格 = 教師 Bbox を 100% 覆い面積 2.0 倍未満）

| | ガラス | 布タグ | 実部品 | 吸音材 | 全体 |
|---|---|---|---|---|---|
| gen | 98.2% | 99.7% | 97.6% | 94.9% | **97.8%** |
| val | 97.4% | 99.2% | 96.7% | 91.3% | **96.8%** |

## 他の PC での始め方

```
# 1. 展開（プロジェクトの直下で）
tar xzf 04_コードと素材/code_tools_blender.tar.gz
tar xzf 04_コードと素材/assets_hdri_props.tar.gz
mkdir -p out && tar xf 03_画像データ/ds_v4.tar          # out/ds_v4 ができる
cp 02_学習済みモデル/*.pt out/

# 2. Python: numpy scipy pillow torch opencv-python-headless==4.10.0.84（EXR のため版を固定）
set OPENCV_IO_ENABLE_OPENEXR=1
set KMP_DUPLICATE_LIB_OK=TRUE

# 3. 推論の自己検証（データの 1 コマで合格するか）
python tools/infer_block.py --model out/v6H.pt --selftest out/ds_v4

# 4. 任意の画像で推論
python tools/infer_block.py --model out/v6H.pt --image photo.jpg --vis result.png

# 5. 評価（全コマ、材質別の合格率と画像）
python tools/viz_block2.py --root out/ds_v4 --ckpt out/v6H.pt --split gen --scan 2000 --out out/v6_eval
```

学習し直す・データを作り直す手順は設計書の §8。Intel GPU の PC では、レンダは Blender の oneAPI、学習は PyTorch の XPU 版が必要（コードの GPU 選択に追記が要る）。
