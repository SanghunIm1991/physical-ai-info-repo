# Isaac GR00T を使ったフィジカルAIの開発

Isaac GR00T（以下 GR00T）は、NVIDIA が開発する、人型ロボットなどに向けた VLA（映像と言葉の指示から、ロボットの動作を出すモデル）です。このファイルは、GR00T を自分のロボットに合わせて追加学習し、動かすエンジニア向けに、ほかのファイルより一歩踏み込んで、モデルの構成・データの形式・追加学習・推論と実機への展開の要点と、周辺の道具を整理します。GR00T の短い紹介は [`vlm.md`](vlm.md#isaac-gr00t) を、NVIDIA の技術全体の中での位置づけは [`nvidia.md`](nvidia.md) を先に読んでください。

- **前提の知識**: 模倣学習の基本（人の操作の記録から方策を学ばせること）と、Python・PyTorch を使えることを前提にします。コマンド・引数・クラスの名前は、公式リポジトリの表記をそのまま併記します。
- **版の扱い**: 確認日の時点の最新は N1.7 です。下の説明は、[公式リポジトリ](https://github.com/NVIDIA/Isaac-GR00T)の `main` の枝の README と手引き（N1.7 向け）を土台にしています。数字（速さ・パラメータ数など）は、特に断らない限り開発元が示す値で、条件が違えば変わります。
- **選び方**: GR00T の公式リポジトリが案内している周辺の道具（人型ロボットの全身の制御、LeRobot での利用）と、次の世代（N2）の土台として開発元が挙げた研究を項目にしました。
- **ほかのファイルに載せているもの**: GR00T の学習用データを人工的に増やす GR00T-Mimic・GR00T-Dreams は [`nvidia.md`](nvidia.md#isaac-gr00t-のデータ生成の設計図) に、GR00T をシミュレーションで大規模に評価する Isaac Lab-Arena は [`isaac-lab.md`](isaac-lab.md#isaac-lab-arena) にあります。
- **今回は載せていないもの**: 以前の版（N1.5・N1.6）の使い方の詳細、強化学習での追加学習（RLinf など）の詳細、個々のベンチマークの数字の比較。

## GR00T での開発の要点

### 版の変遷

| 版 | 主な変化 | 出典 |
|---|---|---|
| N1 | 最初の版（論文は 2025-03）。VLM の部分（System 2）が場面と指示を読み、拡散 Transformer の部分（System 1）が実時間で動作を作る、2段の構成。実機のロボットの軌跡・人の動画・人工的なデータを混ぜて学習 | [論文 arXiv:2503.14734](https://arxiv.org/abs/2503.14734) |
| N1.5・N1.6 | 公式リポジトリの別の枝（`n1d5`・`n1d6`）に残る。N1.6 の VLM の部分は Eagle | 公式リポジトリの README |
| N1.7 | VLM の部分を Cosmos-Reason2-2B に替えた。人の動画も使う、手先の相対の動きの表し方を導入 | 公式リポジトリの README |
| N2 | 2026-03 の GTC で予告。世界と動作をあわせて予測する新しい構成（[DreamZero](#dreamzero) の研究が土台）で、2026 年末までの公開を目指す | [開発元の発表](https://nvidianews.nvidia.com/news/nvidia-and-global-robotics-leaders-take-physical-ai-to-the-real-world) |

- **N1.7 の段階の表記が、資料によって違う。** 2026-03-16 の開発元の発表と、モデルの公開ページ（Hugging Face）は「先行の版（early access）」としている。一方、公式リポジトリの README は「一般提供（General Availability）」の版としている。製品のページ（[Isaac GR00T の製品のページ](https://www.nvidia.com/en-us/industries/robotics/isaac-groot/)）は、商用のライセンスで使える最新の版としている。
- **N2 は確認日の時点で公開されていない。** 公式リポジトリの README・製品のページには、N2 の記載がない。開発元によれば、N2 は新しい作業・新しい環境での成功が、主な VLA の2倍を超える。

### 構成

N1.7 の構成は次のとおりです（公式リポジトリの README・[モデルの公開ページ](https://huggingface.co/nvidia/GR00T-N1.7-3B)）。

- **全体**: 映像と言語の基盤モデル（VLM）と、連続の値の動作を、雑音を除きながら作る拡散 Transformer（DiT）の頭部を組み合わせる。パラメータ数は 3B。
- **VLM の部分**: Cosmos-Reason2-2B（Qwen3-VL の構成）。N1.6 の Eagle から替えた。最初に使うときに読み込むが、利用の申し込みが要るモデルなので、Hugging Face で事前に申し込む。
- **動作の頭部**: フローマッチングの DiT（16 層。N1.6 は 32 層）。ロボットの状態（関節の角度など）と動作は、機体ごとに用意した小さなネットワーク（MLP）で出し入れする。
- **一度に出す動作の長さ**: 40 歩分（N1.6 は 16）。状態と動作の次元の上限は 132（N1.6 は 29）。
- **学習のデータ**: 人の視点の動画（EgoScale）2 万時間と、双腕・上半身の人型・人型のロボットの操作の記録。開発元によれば、N1.6 と同程度の性能で、新しい場面への対応と、言葉の指示に従う力が上がった。
- **手先の相対の動き**: N1.7 は、手先（エンドエフェクタ）の相対の動きで動作を表す形を、ロボットと人で共通に使う。人の動画から学んだことを、ロボットの動作に生かすための工夫と読める（このリポジトリの解釈）。

### データの形式

- **形式**: LeRobot（Hugging Face のロボット学習の枠組み）の v2 の形式に、`meta/modality.json` を足したもの。`meta/` に情報のファイル、`data/` に parquet、`videos/` に mp4 を置く。
- **`modality.json` の役割**: つなげて並べた状態と動作の配列を、名前の付いた部分（腕の関節・グリッパーなど）に切り分ける位置と、カメラの映像の名前を示す。
- **LeRobot v3 のデータ**: 用意されたスクリプト（`scripts/lerobot_conversion/convert_v3_to_v2.py`）で v2 に変換する。
- **動画の読み込み**: `torchcodec` だけを使うので、FFmpeg は 4〜7 の版が要る。以前の decord・pyav は使えなくなった。

### 機体のタグ

推論・追加学習のどのコマンドでも、どの機体のデータかを示すタグ（`--embodiment-tag`）を指定します（[方策の API の手引き](https://github.com/NVIDIA/Isaac-GR00T/blob/main/getting_started/policy.md)）。

| 区分 | タグの例 | 意味 |
|---|---|---|
| 事前学習のタグ | `OXE_DROID_RELATIVE_EEF_RELATIVE_JOINT`、`REAL_G1`、`XDOF` など | 基本のモデル（`nvidia/GR00T-N1.7-3B`）をそのまま試せる |
| 追加学習済みのタグ | `LIBERO_PANDA`、`SIMPLER_ENV_GOOGLE`、`SIMPLER_ENV_WIDOWX`、`UNITREE_G1_SONIC` など | 対応する追加学習済みのモデル（`nvidia/GR00T-N1.7-LIBERO` など）と組で使う |
| 独自のロボット | `NEW_EMBODIMENT` | 自分のロボットに追加学習するときに使う。1つの Python のプロセスで登録できる設定は1つだけ |

### 自分のロボットへの追加学習

[新しい機体への追加学習の手引き](https://github.com/NVIDIA/Isaac-GR00T/blob/main/getting_started/finetune_new_embodiment.md)は、市販の小型の腕（SO-100）を例に、次の手順を示しています。

1. **データを用意する**: 上の形式でデータセットを作る。
2. **機体の設定を書く**: カメラの名前、状態と動作の部分の名前、動作の表し方（腕は相対の値、グリッパーは絶対の値など）、何歩先までの動作を学ばせるか、言葉の指示の項目を、Python の設定に書き、`NEW_EMBODIMENT` のタグで登録する。部分の名前は `modality.json` と合わせる。
3. **学習させる**: `gr00t/experiment/launch_finetune.py` に、基本のモデル・データセット・タグ・設定のファイル（`--modality-config-path`。独自のロボットのときだけ要る）を渡す。手引きの例は、色の揺らぎによるデータの水増しを入れて 2,000 歩を学習させる。検証は既定では行わないので、要るときは引数で有効にする。
4. **記録と比べる**: `gr00t/eval/open_loop_eval.py` で、記録の動作とモデルの出す動作の差（MSE）を確かめる。手引きは、5 回分だけの見本のデータで目標の値を示していない。開発元の例（H100 の 1 枚・2,000 歩）では、学習が進むにつれて差が下がっていく様子を示している。

- **計算機の目安**: 開発元によれば、推論は GPU のメモリ 16 GB 以上（RTX 4090・L40・H100・Jetson AGX Thor など）、追加学習は 40 GB 以上を勧め、H100 か L40 を推奨している。A6000 でもできるが時間がかかる。
- **環境**: パッケージの管理には `uv` を使い、Python 3.12 で入れる。Jetson（Thor・Orin）や DGX Spark のような aarch64 の計算機では、`uv run` ではなく、そのまま `python` で動かすよう注意書きがある。

### 推論と実機への展開

- **方策の API**: `Gr00tPolicy` に、カメラごとの映像（uint8）、名前ごとの状態（float32）、言葉の指示を辞書で渡すと、名前ごとの動作の系列が返る。動作は正規化されていない、物理の単位の値。開発中は、入力の形を確かめる設定（`strict=True`）を勧めている（[方策の API の手引き](https://github.com/NVIDIA/Isaac-GR00T/blob/main/getting_started/policy.md)）。
- **サーバーとクライアント**: `gr00t/eval/run_gr00t_server.py` でモデルをサーバーとして立て（既定の通信口は 5555）、ロボットやシミュレータの側の `PolicyClient` から呼び出す。通信は ZeroMQ を使う。重い推論の計算機と、ロボットの制御の計算機を分けられる。
- **出した動作をどこまで使うか**: モデルは 40 歩分を出すが、実際に実行するのはその先頭の一部（`--execution-horizon`。実時間で動かすときは 8 がよく使われる）にして、次の推論の結果につなぐ。
- **TensorRT への変換**: `scripts/deployment/build_trt_pipeline.py` で、ONNX への書き出し・TensorRT のエンジンの作成・PyTorch との出力の一致の確かめを、まとめて行う。エンジンは GPU の世代ごとに作り直す（[展開の手引き](https://github.com/NVIDIA/Isaac-GR00T/blob/main/scripts/deployment/README.md)）。
- **Jetson**: Thor と Orin は、JetPack 7.2（CUDA 13.2）が要る。古い JetPack（Orin の 6.x など）は使えなくなった。
- **速さ**: 条件（カメラの数・量子化・測り方）によって値が大きく変わる。例えば、展開の手引きの表（カメラ2台・1回の推論の全体の時間）では、H100 で TensorRT にすると約 23 ms、Jetson AGX Thor では TensorRT で約 77 ms、さらに一部を NVFP4 に量子化すると約 36 ms。README は、Thor で約 28 Hz としている。手引きは、これらの値は閉ループでの作業の成功を測ったものではないと注意している。

### 評価

- **記録と比べる評価（開ループ）**: 上の `open_loop_eval.py` で、記録の動作との差を測る。学習がうまく進んでいるかの確かめに使う。
- **動かして試す評価（閉ループ）**: サーバーとクライアントの形で、シミュレーション（LIBERO・SimplerEnv・RoboCasa など）または実機で、作業の成否を確かめる。N1.7 の README は、Unitree G1 の実機の評価も挙げている。開ループの差が小さくても、閉ループで成功するとは限らないので、両方で確かめる（このリポジトリの整理）。

### ライセンス

- コードは Apache-2.0、モデルの重みは NVIDIA Open Model License（README・モデルの公開ページの表記）。モデルの公開ページは、商用・非商用のどちらにも使えるとしている。
- 土台の Cosmos-Reason2-2B は、利用の申し込みが要るモデル。使う前に、それぞれの条件を確かめる。

## 一覧

| 項目 | 分類 | 種類 | 状態 | 確認日 |
|---|---|---|---|---|
| [GR00T-WholeBodyControl](#gr00t-wholebodycontrol) | 人型ロボットの全身の制御 | OSS | 研究段階 | 2026-11-13 |
| [LeRobot の GR00T 対応](#lerobot-の-gr00t-対応) | 他の枠組みからの利用 | OSS | 研究段階 | 2026-11-13 |
| [DreamZero](#dreamzero) | 次の世代の土台の研究 | OSS | 研究段階 | 2026-11-13 |

GR00T のモデルそのものの項目は [`vlm.md`](vlm.md#isaac-gr00t) にあります。

## OSS

### GR00T-WholeBodyControl

- **概要**: 人型ロボットの全身の制御器を、学習・評価・展開するための NVIDIA の研究所（NVlabs）の基盤。2つの制御器を含む。**分離型の全身の制御（Decoupled WBC）** は、下半身を強化学習の方策、上半身を逆運動学で動かすもので、GR00T N1.5・N1.6 で使われた。**GEAR-SONIC** は、人の動きの記録をまねる学習（動きの追従）を大規模に行って作った、人型ロボットの動きの基盤モデルで、VR の遠隔操作と VLA の両方からの指令を1つの方策で受ける（論文の要旨）。Unitree G1 向けの学習済みのモデルを公開し、制御は 50 Hz で、C++ と TensorRT で動かす。2026-05 には、GR00T N1.7 と組み合わせて、データの収集・追加学習・展開を G1 で通して行う手順が加わった。
- **一次情報**: [公式リポジトリ](https://github.com/NVlabs/GR00T-WholeBodyControl)・[論文 arXiv:2511.07820（SONIC）](https://arxiv.org/abs/2511.07820)
- **種類**: OSS
- **ライセンス**: コードは Apache-2.0、モデルの重みは NVIDIA Open Model License（README の表記）
- **状態**: 研究段階（研究所の公開で、リポジトリ全体の正式な版の番号がない。同梱の MotionBricks は試作の公開〈preview〉。GR00T N1.7 と通して使う手順は `UNITREE_G1_SONIC` のタグでだけ対応〈Isaac GR00T の README〉）
- **確認日**: 2026-11-13
- **情報源**: 一次

### LeRobot の GR00T 対応

- **概要**: Hugging Face のロボット学習の枠組み LeRobot から、GR00T N1.7 を方策の1つ（`--policy.type=groot`）として学習・評価・実行できるようにしたもの。LeRobot のコマンド（`lerobot-train`・`lerobot-eval` など）で、基本のモデルを指定して追加学習し、LIBERO などのシミュレーションで評価できる。実行のときは、動作の系列をつなぎながら実時間で動かす仕組み（RTC）を選べる。現在の版は N1.7 だけに対応し、N1.5 を使い続けるには LeRobot の古い版（0.5.1）に固定する必要がある。開発元は、LIBERO での結果を予備的なものとして示している。
- **一次情報**: [LeRobot の GR00T の文書](https://huggingface.co/docs/lerobot/groot)
- **種類**: OSS
- **ライセンス**: GR00T N1.7 のモデルは NVIDIA Open Model License（文書の表記）。LeRobot 本体のライセンスは、今回読んだ文書には書かれていなかった（未確認）
- **状態**: 研究段階（文書が結果を「予備的」としている。N1.5 の対応を外すなど、互換性のない変更がある）
- **確認日**: 2026-11-13
- **情報源**: 一次

### DreamZero

- **概要**: NVIDIA の研究所（GEAR Lab）による、世界と動作をあわせて予測するモデル（World Action Model）の研究（論文は 2026-02）。学習済みの動画の拡散モデルを土台にし、先の映像と動作を一緒に作る。論文によれば、VLA は意味の理解では一般化できるが、新しい環境での見たことのない動きには弱く、DreamZero は実機の実験で、新しい作業と環境への一般化が、最高水準の VLA の2倍を超えた。14B の動画の拡散モデルを 7 Hz の閉ループで動かし、ほかのロボットや人の動画だけの手本を 10〜20 分加えると、見たことのない作業の成績が 42% 以上上がったとしている。開発元は、GR00T N2 をこの研究に基づくものとしている。
- **一次情報**: [公式リポジトリ](https://github.com/dreamzero0/dreamzero)・[論文 arXiv:2602.15922](https://arxiv.org/abs/2602.15922)・[研究のページ](https://dreamzero0.github.io/)
- **種類**: OSS
- **ライセンス**: コードは Apache-2.0（README の表記）。公開されたモデルの重みのライセンスは、読んだ README には書かれていなかった（未確認）
- **状態**: 研究段階（README で研究のプロジェクトとしている。DROID で学習させた 14B のモデル・AgiBot 向けのモデル・学習のコード・新しい機体への手引きを公開済みで、一部のシミュレーション環境への対応は「近日公開」）
- **確認日**: 2026-11-13
- **情報源**: 一次
