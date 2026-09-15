# logo_vision v7 アーカイブ（2026-09-15）

刻印ブロックの Bbox 切り出しシステムの**ベースライン（v7 = Kp2）**一式。
v6 との違いは、実車写真の評価（v6 は 80 枚中 27 枚合格、離れた複数の群に分かれる刻印で全滅）に合わせて**データを 1 から作り直した**こと。

- 撮影画像 1400×2000 の縦長。網の入力は短辺 1024（1024×1472）
- 刻印は 1〜3 群、群の隙間は最大 50mm
- 材質: ガラス・布タグ・樹脂（ドアミラー想定）・レンズ（テールライト想定）・吸音材・実部品

## 中身

| フォルダ | 内容 |
|---|---|
| `01_設計書/` | **`10_v7データセット_実写評価から分かる条件.md`（まずこれ）**<br>v7 の条件・試作・速度の検討・学習と評価の記録・決定の経緯（§10.1〜10.16）。<br>`08_システム設計書_v6.md` はデータ生成・網の構造の土台（v6 時点）、`CLAUDE.md` はプロジェクトの前提と現在地、`00_索引.md` は文書の案内 |
| `02_学習済みモデル/` | **`v7_Kp2.pt`**（ベースライン。構成 `KLFN96` が重みに入っていて、このファイルだけで網が組める）<br>`conf_Kp2.pkl`（信頼度の学習器）、`conf_feats_Kp2.npz`（その学習に使った特徴）、`conf_Kp2_gen.json`（gen のコマごとの信頼度）<br>`v7_KLFN96_init.pt`（Kp2 の学習の初期値）、`log_1_Kp.txt` / `log_2_Kp2.txt` / `log_conf_Kp2.txt`（学習ログ） |
| `03_画像データ/` | **`ds_v7.tar`**（データセット 367 シーン / 7,267 枚: train 283 / val 12 / gen 24 / conf 48 シーン、EXR + index.json + シーン npz）。**GitHub には含めない（別途受け渡し）**<br>`評価画像/`: Kp2 のギャラリーと段階の図、材質別などの表（`eval_v7_*.txt`、比較用に基準・Kp・S3p・S3p2 も）、コマごとの結果（`rows_*.json`）、刻印が写っているかの測定（`solvable.json`）、確認用シート（`sheets/`） |
| `04_コードと素材/` | `code_tools_blender.tar.gz`（コードと vast 用のスクリプト）、`assets_hdri_props.tar.gz`（HDRI・小道具・ロゴ）、`step_files.tar.gz`（実部品の STEP 140 個） |

## 結果（合格 = 教師 Bbox を 100% 覆い、面積 2.0 倍未満）

| | ガラス | 布タグ | 樹脂 | レンズ | 吸音材 | 実部品 | 全体 |
|---|---|---|---|---|---|---|---|
| gen（466 枚） | 92.5% | 100% | 95.0% | 95.0% | 97.0% | 93.8% | **95.5%** |

- val（229 枚）は 93.0%
- 刻印がはっきり見えるコマ（勾配比 1.2 以上）に限ると gen 97.2%、かすかなコマ（0.9〜1.2）は 88.9%
- 前の本命 v7_KLFN96（gen 93.8%）から、gen + val の 695 枚で 直った 15 / 壊れた 3
- 推論時間（CPU、評価側の推定）: 約 48ms（手元の PC で 81ms を、同じ PC の v6H との比で換算）
- 信頼度（失敗の検出）: 交差検証 AUROC 0.925、gen 0.912。合格率 98% を保てる受理率 91.0%

**注意: 合成データ（デジタルツイン）での数字。実車での合格率ではない。**

## データの出どころ（再現の注意）

- train の最初の 149 シーンは手元の PC でレンダ、それ以外（追加の train 134、val・gen・conf）は vast.ai の RTX 4090 でレンダ。どちらも `make_dataset.py --plan v7` の同じシード
- Cycles のノイズは機体で変わるので、画像はビット単位では一致しない（同じシードでの合格率の差は 0.2 ポイント程度だった）
- 上の gen / val の数字は、このアーカイブの `ds_v7.tar` の gen / val（vast のレンダ）での値
- 実部品 p70517（STEP のメッシュ化が止まる）と p70529（刻印面の平面性の検査で停止）は除外

## 他の PC での始め方

```
# 1. 展開（プロジェクトの直下で）
tar xzf 04_コードと素材/code_tools_blender.tar.gz
tar xzf 04_コードと素材/assets_hdri_props.tar.gz
mkdir -p out/v7 && tar xf 03_画像データ/ds_v7.tar -C out   # out/ds_v7 ができる
cp 02_学習済みモデル/*.pt 02_学習済みモデル/*.pkl out/v7/

# 2. Python: numpy scipy pillow torch scikit-learn opencv-python-headless==4.10.0.84（EXR のため版を固定）
set OPENCV_IO_ENABLE_OPENEXR=1
set KMP_DUPLICATE_LIB_OK=TRUE

# 3. 評価（材質別の合格率と画像）
python tools/viz_block2.py --root out/ds_v7 --ckpt out/v7/v7_Kp2.pt --split gen --n 9 --scan 2000 --out out/v7/eval_Kp2 --dump out/v7/rows_Kp2_gen.json
python tools/eval_v7.py --root out/ds_v7 --rows out/v7/rows_Kp2_gen.json

# 4. 信頼度（特徴の抽出 -> 学習器の選択と gen での確認）
python tools/conf_v7.py extract --ckpt out/v7/v7_Kp2.pt --out out/v7/feats_Kp2.npz --splits val,gen,conf --refjit 1
python tools/conf_v7.py fit --feats out/v7/feats_Kp2.npz --out out/v7/conf_Kp2.pkl --train-splits conf,val
```

**任意の画像 1 枚での推論（`tools/infer_block.py`）は、まだ v6 専用（1024×768 横長）。v7 対応は次の作業。**

## 学び直す手順

```
# データ（vast の 4 GPU で約 40 分。GPU 1 枚に Blender 4 本まで）
python tools/make_dataset.py --plan v7 --root out/ds_v7 --samples 48 --workers 2 --shard 0/8   # 分割ごとに

# 1 巡目（val 最良が ep4）
python tools/train_block2.py --root out/ds_v7 --arch KLFN96 --init out/v7/v7_KLFN96_init.pt --short 1024 --loss pass \
  --refine 1 --ft 1 --warm 0 --bb-lr 0.3 --lr 1e-4 --epochs 10 --lam 3 --n-val 229 --extent 1 --inward 3 --out out/v7/v7b_Kp.pt

# 続き 10 エポック（最後のエポックの重み = v7_Kp2.pt。out の _last.pt）
python tools/train_block2.py （同じ引数） --init out/v7/v7b_Kp.pt --out out/v7/v7b_Kp2.pt
```

- v7_KLFN96_init.pt 自体は、v6 の v6H.pt から名前と形が合う重みを移し、旧データ（185 シーン）で 16 エポック学習したもの（設計書 §10.7〜10.8）
- vast 用のスクリプト: `out/v7/setup7b.sh`（環境）・`render7b.sh`（レンダ）・`train7b.sh`（学習と評価）・`cont7b.sh`（続きの学習と評価）

## 採らなかったもの（同じ検討を繰り返さないため）

| 案 | 結果 |
|---|---|
| 損失を辺ごとの余白の比率にする | 覆えた割合が 2〜4 ポイント下がった。廃止 |
| 多段の狭める頭（Refiner 3 段） | 1 段より 6 枚少ない合格、推論 +11%。不採用 |
| 信頼度の特徴を原因ごとに束ねたヒートマップ | 原因がほぼ「辺の位置が定まらない」になり、場所も読めない。「箱が間違えそうな場所」を学習する方式に切り替え中 |
