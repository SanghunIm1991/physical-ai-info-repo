# Isaac Lab を使ったフィジカルAIの開発

Isaac Lab は、NVIDIA が開発する、ロボットの学習（強化学習・模倣学習）のための GPU で並列に動く枠組みです。このファイルは、Isaac Lab で実際に方策を作るエンジニア向けに、ほかのファイルより一歩踏み込んで、環境の設計・学習・実機への展開の要点と、周辺の道具を整理します。Isaac Lab の位置づけと、ほかのシミュレータとの違いは [`simulator.md`](simulator.md#isaac-sim-と-isaac-lab) を、NVIDIA の技術全体の中での位置づけは [`nvidia.md`](nvidia.md) を先に読んでください。

- **前提の知識**: 強化学習の基本（方策・報酬・PPO など）と、Python・PyTorch を使えることを前提にします。用語は、Isaac Lab の公式の文書の英語の名前（クラス名・設定の名前）をそのまま併記します。
- **版の扱い**: 確認日の時点で、安定の版は 2.3 系（2.3.2 が最後）で、3.0 は先行の版（EA）です。下の説明は、2.3 系の公式の文書を土台にし、3.0 で変わる点はその都度書きます（[版の選び方](#版の選び方)）。数字（成功率・時間など）は、特に断らない限り開発元の文書が示す値で、条件が違えば変わります。
- **選び方**: 開発の流れ（環境を作る → 学習させる → 評価する → 実機に移す）の各段階から、Isaac Lab の公式の文書が案内している道具と手引きを選びました。強化学習のライブラリは、2.3 系の文書の比較の表に載る4つのうち、論文の著者に Isaac Lab の技術報告の筆頭の著者が含まれる RSL-RL と、同じ表で文書が最も充実している（Comprehensive）とされる skrl を項目にしました。
- **今回は載せていないもの**: 個々の学習の環境（四脚の歩行、手の操作など）の解説、クラウドやクラスタでの運用（コンテナ・Ray など）、Isaac Sim 側の機能（アセットの作り方・ROS とつなぐ仕組み）の詳細。

## Isaac Lab での開発の要点

### 版の選び方

| 版 | 対応する Isaac Sim | 状況 |
|---|---|---|
| 2.3 系（`main` の枝） | 4.5・5.0・5.1 | 正式な版。2.3.2（2026-02-02）が `main` の枝の最後の版で、開発は 3.0 に移った |
| 3.0.0-EA | 6.1 | 先行の版（2026-09-16）。開発元は、正式な版（GA）を 2026 年 10 月末に出すことを目標としている |
| `release/3.0.0` の枝・`develop` の枝 | 6.2 | 3.0 の正式な版に向けた開発中のもの |

3.0 は、内部の構成が大きく変わります（[GitHub のリリースの一覧](https://github.com/isaac-sim/IsaacLab/releases)にある 3.0.0-EA と beta の説明による。上の表の GA の目標も同じ）。

- **計算の部分を選べる**: 物理の計算（Isaac Sim の PhysX、Newton、Kit を使わない PhysX〈OVPhysX〉）、描画、可視化を、それぞれ別に選べる。対応する環境は、環境のコードを書き換えずに切り替えられる。
- **Isaac Sim なしで動かせる**: Newton の物理と描画を使えば、Isaac Sim（Omniverse Kit）を入れずに学習できる。Isaac Sim が要るのは、XR の遠隔操作・ROS とつなぐ仕組み・Isaac Sim の PhysX や RTX の描画を使う場合などに限られる。
- **互換性のない変更**: 主なものは次のとおり。2.3 系のコードを移すときは、リリースノートの一覧で確かめる。
  - 四元数の並びが WXYZ から XYZW に変わる
  - 状態を読む属性（`.data.*`）が `torch.Tensor` ではなく Warp の配列を包んだもの（`ProxyArray`。`.torch` で PyTorch のテンソルとして取り出す）を返す
  - `--headless` が `--viz` に変わる
  - `TiledCamera` が `Camera` に統合される
  - `isaaclab.sh` は非推奨になり、操作は `uv run isaaclab <サブコマンド>` の形の1つのコマンドにまとまる。例えば、学習は `uv run isaaclab train --rl_library rsl_rl --task <課題の名前>`、学習させた方策の再生は `play`、課題の一覧は `list_envs`。使うライブラリや計算の部分の追加の部品は、`uv run --extra skrl isaaclab ...` のように `--extra` で指定する（[3.0 の文書](https://isaac-sim.github.io/IsaacLab/release/3.0.0/source/developer-tools/template_generator.html)による）

- **Newton の扱い**: 2.3 系の文書（`main` の枝）では、Newton との連携は試験的な機能とされ、対応する環境は古典的な課題と平地の歩行に限られていた。3.0 の文書（`release/3.0.0` の枝）では、[物理の計算の部分の1つ](https://isaac-sim.github.io/IsaacLab/release/3.0.0/source/concepts/physics_backends.html)として説明され、ソルバーは、主に確かめられた経路である MuJoCo Warp（MJWarp）のほか、剛体向けの Kamino、布や柔らかい物体向けの VBD、粒子の材料向けの MPM などを選べる。どの課題がどの計算の部分に対応するかは課題ごとに違い、各課題の `--help` や環境の一覧で確かめる（[Newton の説明](https://isaac-sim.github.io/IsaacLab/release/3.0.0/source/concepts/physics-backends/newton.html)）。

新しく始める場合、正式な版で安定して動かしたいなら 2.3 系、3.0 の機能（Newton、Isaac Sim なしでの学習、LEAPP での書き出し）を使いたいなら 3.0 を選ぶ、というのが判断の軸になります（このリポジトリの整理）。2.3 系の開発は終わっているので、長く使うコードは、3.0 への移行を見込んで書くのが無難です。

### 環境の設計

環境（task）は、観測を返し、行動を受け取り、シミュレーションを進める部分です。Isaac Lab には、2つの書き方があります（[公式の文書](https://isaac-sim.github.io/IsaacLab/main/source/overview/core-concepts/task_workflows.html)）。

| | マネージャー型（manager-based） | 直接型（direct） |
|---|---|---|
| 基底のクラス | `ManagerBasedRLEnv`（設定は `ManagerBasedRLEnvCfg`） | `DirectRLEnv`、複数のエージェントは `DirectMARLEnv` |
| 書き方 | 観測・行動・報酬などを、役割ごとの「マネージャー」に、設定のクラスで項（term）として登録する | 観測・行動・報酬・リセットを、1つのクラスのメソッドとして書く |
| 向いている場面 | 部品の差し替え・比較、チームでの分担、同じ課題を別のロボットやセンサーで試す | 部品に分けにくい複雑な処理、PyTorch の JIT や Warp での高速化。IsaacGymEnvs から移る人にもなじみやすい |
| 実機への展開の道具 | 下の IO 記述子・LEAPP の対象 | LEAPP は、注記を足せば RSL-RL で学習したものだけ対象（3.0） |

マネージャー型で使う主なマネージャーは、次のとおりです（[基本の環境の手引き](https://isaac-sim.github.io/IsaacLab/main/source/tutorials/03_envs/create_manager_base_env.html)・[強化学習の環境の手引き](https://isaac-sim.github.io/IsaacLab/main/source/tutorials/03_envs/create_manager_rl_env.html)）。

| マネージャー | 役割 | 設計の要点 |
|---|---|---|
| 行動（Action） | 方策の出力を、関節の目標値などの指令に変える | 項ごとに対象の関節と倍率（scale）を決める。倍率は1歩で動ける量を決める（歯車の挿入の手引きでは、倍率を少し上げると静止摩擦を越えて動けるようになった） |
| 観測（Observation） | 方策に渡す値を計算する | 項をグループにまとめる。`policy` のグループは、Isaac Lab の各種のラッパーが前提にするので必須。項ごとにノイズ・切り詰め・倍率を付けられ、グループごとにノイズを有効にするかを選ぶ |
| イベント（Event） | 決まった時点で処理をする。ドメインランダム化（物理や見た目の条件をばらつかせること）の中心 | 時点は、起動の時（`startup`。質量など重い変更を1回だけ）・リセットの時（`reset`。初期の姿勢など毎回変える）・一定の歩数ごと（`interval`） |
| 報酬（Reward） | 項ごとの関数と重みの和で報酬を計算する | 項ごとの寄与が記録されるので、どの項が学習を引っ張っているかを確かめられる |
| 終了（Termination） | エピソードを終える条件を決める | 時間切れの項には `time_out=True` を付け、失敗による終了と区別する（Gymnasium の打ち切り〈truncation〉の扱い） |
| 指令（Command） | 目標（歩く速さ・手先の目標の姿勢など）を作り、更新する | 観測にも入れて、方策が目標に応じて動けるようにする |
| カリキュラム（Curriculum） | 学習の進み具合に応じて難しさを上げる | 歩行の地形や操作の課題で使われている |

- **時間の刻み**: 物理の計算の刻み（`sim.dt`）と、何刻みごとに方策を1回動かすか（`decimation`）で、制御の周期が決まる。例えば 5 ms（200 Hz）の刻みで `decimation` を 4 にすると、方策は 50 Hz で動く。実機の制御の周期と合わせることが、実機に移すときの前提になる。
- **自分の課題を作る**: 雛形の生成の機能（2.3 系では `isaaclab.sh --new`、3.0 では `uv run isaaclab --new`）で、Isaac Lab のリポジトリの外に、独立したプロジェクトとして作るのが推奨されている。生成のときに、書き方（マネージャー型か直接型か）と強化学習のライブラリを選ぶ（[公式の文書](https://isaac-sim.github.io/IsaacLab/main/source/overview/own-project/template.html)）。

### アクチュエータのモデル

実機に移すときに最も効く設定の1つが、関節の駆動の模擬（アクチュエータのモデル）です（[公式の文書](https://isaac-sim.github.io/IsaacLab/main/source/overview/core-concepts/actuators.html)）。

- **暗黙のモデル（implicit）**: PD 制御を物理エンジンに任せる（`ImplicitActuator`）。計算は安定しやすい。
- **明示のモデル（explicit）**: 指令からトルクを自分で計算し、モーターの限界で切り詰めてから与える。用意されているのは、理想的な PD（`IdealPDActuator`）、速度に応じてトルクの上限が下がる DC モーター（`DCMotor`）、指令の遅れ（`DelayedPDActuator`）、角度に応じた上限（`RemotizedPDActuator`）、実機のデータから学習したモデル（`ActuatorNetMLP`・`ActuatorNetLSTM`）。
- **注意**: 暗黙のモデルで学習した方策は、同じロボットを明示のモデルにすると、うまく動かないことがある。明示のモデルは数値的に不安定になりやすく、関節の `armature` を大きくすると落ち着くことがある。同じ駆動の関節をまとめて、1つのモデルを割り当てる（例: 四脚の全関節が同じ駆動）。

### 強化学習

Isaac Lab は、強化学習のライブラリにつなぐラッパーを用意しています。下の表は、2.3 系の文書の比較の表の値です（[2.3 系の比較](https://isaac-sim.github.io/IsaacLab/main/source/overview/reinforcement-learning/rl_frameworks.html)）。

| ライブラリ | 主なアルゴリズム | 計算の枠組み | 複数の GPU | 複数のエージェント |
|---|---|---|---|---|
| [RSL-RL](#rsl-rl) | PPO、蒸留（教師と生徒） | PyTorch | 対応 | PPO |
| RL-Games | PPO、SAC、A2C | PyTorch | 対応 | PPO |
| [skrl](#skrl) | 多数 | PyTorch、JAX（skrl 自体は Warp でも実装されている。[項目](#skrl)を参照） | 対応 | PPO と複数のエージェント向けのもの |
| Stable-Baselines3 | 多数 | PyTorch | 非対応（並列の環境にも非対応） | — |

3.0 の文書の比較（[3.0 の強化学習の説明](https://isaac-sim.github.io/IsaacLab/release/3.0.0/source/concepts/reinforcement_learning.html)）は、表の形が変わり、「何のために選ぶか」で整理しています。主な違いは次のとおりです。

- RSL-RL は既定で入り、対称性を使ったデータの水増しや RND（探索を促す手法）なども挙げられている。
- 複数のエージェントの学習（MAPPO・IPPO）は skrl でだけ説明されている。
- VLA（Isaac GR00T・OpenVLA）を強化学習で追加学習するための RLinf と、PyTorch の部品で独自のアルゴリズムを組むための TorchRL が加わった。

- **速さ**: 開発元の文書によれば、人型の課題（`Isaac-Humanoid-v0`）を、RTX 4090 の1枚で、描画なし・4,096 の並列の環境で 6,550 万歩だけ学習させた時間は、RSL-RL・RL-Games・skrl がほぼ同じ（約 200 秒）で、Stable-Baselines3 は約 290 秒だった。速さよりも、使いたいアルゴリズム・文書・既存の例の多さで選ぶことになる。
- **並列の環境の数**: 多いほど1回の更新のデータが増えるが、ある所から先は伸びが鈍り、1歩の計算が遅くなる分が上回る。上限を決めるのはふつう GPU のメモリで、カメラの画像の観測・柔らかい物体・細かい衝突の形状がメモリを使う。少ない環境数では、バッチの大きさや1回の試行の長さを増やして補う（[学習の手引き](https://isaac-sim.github.io/IsaacLab/main/source/overview/reinforcement-learning/training_guide.html)）。
- **観測が NaN になる**: 原因はたいてい学習のアルゴリズムではなく、シミュレーションの不安定さ（大きすぎる行動、不正なリセット）にある。関節・速度・力の上限、ゲイン、物理の刻み、ソルバーの反復の回数を見直す（同じ手引き）。
- **複数の GPU と計算機**: PyTorch の torchrun（DistributedDataParallel）で、GPU ごとに1つのプロセスを立て、それぞれが自分の並列の環境と方策の写しを持ち、勾配の更新のときだけ通信する。Linux でだけ使え、対応は RL-Games・RSL-RL・skrl。計算機をまたぐ場合は、通信の遅れで1台より遅くなることもある（[公式の文書](https://isaac-sim.github.io/IsaacLab/main/source/features/multi_gpu.html)）。
- **複数の方策を競わせる学習（Population Based Training）**: 複数の方策を並行して学習させ、成績の悪いものを良いものの写しに置き換えながら、ハイパーパラメータを変えていく。対応は RL-Games だけで、開発元は6つ以上の方策で使うことを勧めている（[公式の文書](https://isaac-sim.github.io/IsaacLab/main/source/features/population_based_training.html)）。

### 模倣学習

人の操作の記録から方策を学ばせる流れは、次のとおりです（[公式の文書](https://isaac-sim.github.io/IsaacLab/main/source/overview/imitation-learning/teleop_imitation.html)。詳しくは [Isaac Lab Mimic と SkillGen](#isaac-lab-mimic-と-skillgen)）。3.0 の文書の目次では、模倣学習の機能は「試験的な機能（Experimental Features）」の欄に置かれています。

1. **記録**: キーボード・SpaceMouse・XR の手の追跡（CloudXR）で、シミュレーションの中のロボットを操作し、成功した操作を HDF5 の形式で記録する。目安は 10 回程度。短く、まっすぐ、止まらずに動かすのがよい。
2. **区切りの注記**: 記録を、物体ごとの小さな作業（つかむ・運ぶ・置くなど）に区切る。手作業、または課題が用意した判定で自動で行う。区切りが少ないほど、つなぎ合わせが滑らかになる。
3. **生成**: Isaac Lab Mimic で、区切りを新しい物体の配置に合わせて変換してつなぎ、成功したものだけを残して、例えば 1,000 回分に増やす。
4. **学習**: robomimic の行動の模倣（BC-RNN）で方策を学習させる。
5. **評価**: 最後のチェックポイントが最良とは限らないので、複数の時点のチェックポイントを試す。

見た目の変化（照明・背景・質感）への強さが要る場合は、生成した動画を Cosmos Transfer で別の見た目に変えてから学習させる方法も用意されています。開発元の文書の例（立方体を積む課題）では、照明や背景を変えた評価で成功率が大きく上がった一方、条件を変えない評価では、増やさずに学習させたものより少し下がりました（[公式の文書](https://isaac-sim.github.io/IsaacLab/main/source/overview/imitation-learning/augmented_imitation.html)）。

### 実機への展開

開発元の、歯車の挿入の手引き（[項目](#歯車の挿入の方策の学習と-ros-での展開)）は、シミュレーションと実機の差を小さくするための具体的な教訓をまとめています。

- 観測には、実機で手に入る値だけを使い、カメラによる推定の値には、実際の誤差に合わせたノイズを加える。
- 関節の摩擦（静止摩擦）を模擬する。模擬しないと、方策は指令した位置にぴったり届く前提で動いてしまう。
- 駆動のゲインと物体の位置をばらつかせる。ただし、ばらつかせすぎると学習が難しくなる。
- 実機の制御は、シミュレーションとずれる設定の少ないもの（インピーダンス制御など）を選ぶ。
- 本格的に学習させる前に、摩擦や接触のソルバーの設定を、実機と見比べて合わせる。
- うまく動かないときは、実機の最初の観測をシミュレーションで再現し、応答を比べる。

方策を実機の側に渡す道具は、次のとおりです。

- **IO 記述子（IO descriptors。2.3 系）**: マネージャー型の環境で学習させた方策の、行動と観測の項の並び・関節の名前・単位・時間の刻みを YAML に書き出す。実機の側で、観測の作り方を手で写し直さずに済む（[公式の文書](https://isaac-sim.github.io/IsaacLab/main/source/policy_deployment/01_io_descriptors/io_descriptors_101.html)）。
- **LEAPP と Isaac ROS Deploy（3.0）**: 方策を、観測と行動の処理ごと ONNX などに書き出し、ROS 2 の上で動かす（[項目](#leapp-と-isaac-ros-deploy)）。LEAPP には次の制約がある（[公式の文書](https://isaac-sim.github.io/IsaacLab/release/3.0.0/source/policy_deployment/05_leapp/exporting_policies_with_leapp.html)）。
  - マネージャー型の環境では、RSL-RL・RL-Games・skrl・Stable-Baselines3 で学習させた方策に対応する。直接型の環境では、注記を足した RSL-RL の方策だけに対応する。
  - テンソルの値によって分岐や繰り返しが変わる処理と、Warp の処理は書き出せない。外部のライブラリの呼び出しも、たどれないことがある。
  - 書き出すときは、学習のときと同じ物理の計算の部分と設定を使う。
- **計算の部分をまたいだ確かめ**: 3.0 の文書は、PhysX と Newton の間で方策を移して確かめること（sim-to-sim）を勧めている。2.3 系の文書（`main` の枝）によれば、Newton で学習させた歩行の方策を実機（Unitree G1）で動かした（[2.3 系の文書](https://isaac-sim.github.io/IsaacLab/main/source/experimental-features/newton-physics-integration/index.html)）。

## 一覧

| 項目 | 分類 | 種類 | 状態 | 確認日 |
|---|---|---|---|---|
| [Isaac Lab 3.0](#isaac-lab-30) | 学習の枠組み | OSS | 研究段階 | 2026-10-18 |
| [RSL-RL](#rsl-rl) | 強化学習 | OSS | 実用 | 2026-10-18 |
| [skrl](#skrl) | 強化学習 | OSS | 実用 | 2026-10-18 |
| [Isaac Lab Mimic と SkillGen](#isaac-lab-mimic-と-skillgen) | 模倣学習 | OSS | 実用 | 2026-10-18 |
| [Isaac Lab-Arena](#isaac-lab-arena) | 評価 | OSS | 研究段階 | 2026-10-18 |
| [LEAPP と Isaac ROS Deploy](#leapp-と-isaac-ros-deploy) | 実機への展開 | OSS | 研究段階 | 2026-10-18 |
| [HOVER](#hover) | 人型ロボットの全身の制御 | OSS | 研究段階 | 2026-10-18 |
| [Isaac Lab の技術報告](#isaac-lab-の技術報告) | 学習の枠組み | 論文 | 研究段階 | 2026-10-18 |
| [歯車の挿入の方策の学習と ROS での展開](#歯車の挿入の方策の学習と-ros-での展開) | 実機への展開 | 解説 | — | 2026-10-18 |

## OSS

### Isaac Lab 3.0

- **概要**: Isaac Lab の次の大きな版。物理の計算・描画・可視化を別々に選べる構成に変わり、Newton を使えば Isaac Sim なしで学習できるようになる。強化学習のライブラリの学習・実行・性能の測定を、1つのコマンド（`isaaclab`）にまとめる。四元数の並び、状態の値の型、コマンドの引数などに、2.3 系と互換性のない変更がある（上の「[版の選び方](#版の選び方)」）。
- **一次情報**: [公式リポジトリ](https://github.com/isaac-sim/IsaacLab)・[リリースの一覧](https://github.com/isaac-sim/IsaacLab/releases)
- **種類**: OSS
- **ライセンス**: 本体は BSD-3-Clause、模倣学習の拡張（`isaaclab_mimic`）は Apache-2.0。Isaac Sim を使う使い方では、Isaac Sim 側の条件の部品もかかわる（README の表記）
- **状態**: 研究段階（確認日の時点の最新は先行の版〈3.0.0-EA、2026-09-16〉で、正式な版〈GA〉は 2026 年 10 月末が目標〈リリースの一覧の表記〉）
- **確認日**: 2026-10-18
- **情報源**: 一次

### RSL-RL

- **概要**: ロボットの研究のための強化学習のライブラリ。LICENSE の著作権者は ETH Zurich と NVIDIA で、論文の著者には Isaac Lab の技術報告の筆頭の著者も含まれる。PPO と、教師の方策から生徒の方策を学ばせる蒸留を中心にし、読みやすく手を入れやすい小さなコードと、GPU だけで学習させることを重視している。複数の GPU での学習に対応する。Isaac Lab のほか、MuJoCo Playground などでも使われている。
- **一次情報**: [公式リポジトリ](https://github.com/leggedrobotics/rsl_rl)・[論文 arXiv:2509.10771](https://arxiv.org/abs/2509.10771)・[パッケージの公開ページ](https://pypi.org/project/rsl-rl-lib/)
- **種類**: OSS
- **ライセンス**: BSD-3-Clause
- **状態**: 実用（正式な版の公開が続いており、確認日の時点の最新は 5.5.1〈2026-09-09〉）
- **確認日**: 2026-10-18
- **情報源**: 一次

### skrl

- **概要**: 部品に分けた作りと、アルゴリズムの実装の読みやすさを重視した強化学習のライブラリ。PyTorch・JAX・NVIDIA Warp で実装され、Isaac Lab・MuJoCo Playground・Gymnasium・PettingZoo などの環境を読み込める。複数のエージェントを1回の学習で同時に学ばせる仕組みを持つ。Isaac Lab で JAX を使って複数の GPU で学習させる場合は、skrl を使う。2.0 で設定の書き方などに互換性のない変更があり、2.1 では学習の挙動の一部が変わったので、以前の結果と比べるときは注意が要る。
- **一次情報**: [公式リポジトリ](https://github.com/Toni-SM/skrl)・[リリースの一覧](https://github.com/Toni-SM/skrl/releases)
- **種類**: OSS
- **ライセンス**: MIT
- **状態**: 実用（正式な版の公開が続いており、確認日の時点の最新は 2.1.0〈2026-05-10〉）
- **確認日**: 2026-10-18
- **情報源**: 一次

### Isaac Lab Mimic と SkillGen

- **概要**: 少数の人の操作の記録から、多数の操作の記録を自動で作る、Isaac Lab の模倣学習の機能。**Isaac Lab Mimic** は、記録を物体ごとの小さな作業に区切り、新しい物体の配置に合わせて変換してつなぎ、成功したものだけを残す。開発元の文書によれば、立方体を積む課題では生成の成功率が約 50%、人型ロボット（GR-1）でつかんで置く課題では 65〜80% で、それで学習させた方策の成功率は課題により 40〜86% だった。**SkillGen** は、区切りの間の移動を、GPU で動く衝突を避ける動作の計画（cuRobo）でつなぐことで、狭い箱の中のような障害物のある場面にも対応する。Mimic は Linux でだけ使え、SkillGen は、開発元によれば、1〜2 の環境で 24 GB 以上の GPU のメモリが要る。
- **一次情報**: [遠隔操作と Mimic の公式の文書](https://isaac-sim.github.io/IsaacLab/main/source/overview/imitation-learning/teleop_imitation.html)・[SkillGen の公式の文書](https://isaac-sim.github.io/IsaacLab/main/source/overview/imitation-learning/skillgen.html)
- **種類**: OSS
- **ライセンス**: `isaaclab_mimic` は Apache-2.0。SkillGen が使う cuRobo は独自の利用許諾（Isaac Lab の README と文書の表記）
- **状態**: 実用（正式な版の Isaac Lab 2.3 に含まれる。SkillGen は 2.3.0 で追加された。ただし、SkillGen は独自の利用許諾の cuRobo に頼るので、その条件に注意が要る。また、3.0 の文書では、模倣学習の機能は「試験的な機能」の欄に置かれている）
- **確認日**: 2026-10-18
- **情報源**: 一次

### Isaac Lab-Arena

- **概要**: Isaac Lab の上で、ロボットの方策を大規模に評価するための枠組み。場面（配置と物体）・ロボット（機体・センサー・制御）・課題（何をさせるか）の3つを別々に定義して組み合わせ、Isaac Lab の環境を作る。物体は「上に」「隣に」のような関係で配置できる。GPU で並列の数千の環境で評価を回し、小さな作業ごとの成否の判定や、条件の変化への敏感さの分析で、方策がどこで失敗するかを調べる。評価する方策として、Isaac GR00T・π0.5 などを、クライアントとサーバーの形でつなぐ例がある。
- **一次情報**: [公式リポジトリ](https://github.com/isaac-sim/IsaacLab-Arena)
- **種類**: OSS
- **ライセンス**: Apache-2.0。動かすには Isaac Sim（別の条件の部品を含む）が要る
- **状態**: 研究段階（README で、試作の段階〈Alpha〉で API が安定しておらず、本番での利用は勧めないとしている。確認日の時点の版は 0.3.1 で、Isaac Lab 3.0.0-EA と組み合わせる）
- **確認日**: 2026-10-18
- **情報源**: 一次

### LEAPP と Isaac ROS Deploy

- **概要**: Isaac Lab で学習させた方策を、実機で動かすための2つの道具（Isaac Lab 3.0 の世代）。**LEAPP**（Lightweight Export Annotations for Policy Pipelines）は、方策のネットワークだけでなく、観測の作り方・行動の変換・再帰の状態の扱いまでをまとめて書き出す。出力は、モデル（既定は ONNX）、実行の周期などの情報、再帰の状態や前回の行動の初期値（safetensors）である。対応する学習の仕方と制約は、上の「[実機への展開](#実機への展開)」に書いた。**Isaac ROS Deploy** は、LEAPP で書き出した方策を、ROS 2 の上で NVIDIA Triton で推論して動かすパッケージの集まりで、方策の入出力を ROS のトピックや ros2_control につなぎ、出力を安全のための制御器で制限することもできる。VLA の方策にも対応するとしている。
- **一次情報**: [LEAPP での書き出しの公式の文書](https://isaac-sim.github.io/IsaacLab/release/3.0.0/source/policy_deployment/05_leapp/exporting_policies_with_leapp.html)・[Isaac ROS Deploy の公式リポジトリ](https://github.com/NVIDIA-ISAAC-ROS/isaac_ros_deploy)
- **種類**: OSS
- **ライセンス**: Isaac ROS Deploy は Apache-2.0。LEAPP のパッケージ自体のライセンスは、読んだ文書には書かれていなかった（未確認）
- **状態**: 研究段階（LEAPP は正式な版の前の Isaac Lab 3.0 の機能。Isaac ROS Deploy は版の番号を示しておらず、最新の更新は 2026-09-21。Isaac ROS Deploy で、Isaac Sim の中のロボットに方策を動かす機能は、確認日の時点の最新の公開には含まれていない）
- **確認日**: 2026-10-18
- **情報源**: 一次

### HOVER

- **概要**: 人型ロボットの全身の制御の方策を、Isaac Lab で学習させるための拡張。人の動きのデータを手本に、まず教師の方策を学習させ、それを使って生徒の方策を蒸留で学ばせる。論文によれば、移動では胴体の速度、机の上の作業では上半身の関節の角度、のように用途ごとに違う指令の与え方を、用途ごとに別の方策を作らずに1つの方策にまとめ、切り替えて使えるようにすることを狙っている（ICRA 2025）。MuJoCo の上での確かめ（sim-to-sim）と、実機（Unitree H1）への展開の手順を含む。開発元は、Isaac Lab の公式の文書で、学習と展開の手引きとして紹介している。
- **一次情報**: [公式リポジトリ](https://github.com/NVlabs/HOVER)・[論文 arXiv:2410.21229](https://arxiv.org/abs/2410.21229)・[公式の文書の手引き](https://isaac-sim.github.io/IsaacLab/main/source/policy_deployment/00_hover/hover_policy.html)
- **種類**: OSS
- **ライセンス**: Apache-2.0
- **状態**: 研究段階（論文の実装で、正式な版の公開がない。動作を確かめた版は Isaac Lab 2.0.0・Isaac Sim 4.5.0 で、確認日の時点の最新より古い。実機への展開は H1 だけに対応）
- **確認日**: 2026-10-18
- **情報源**: 一次

## 論文

### Isaac Lab の技術報告

- **概要**: Isaac Lab の全体を説明した NVIDIA の技術報告（2025-11）。Isaac Gym の後継として、GPU で並列の物理の計算と写実的な描画、部品を組み合わせる環境の作り、アクチュエータのモデル、周期の違う複数のセンサーの模擬、データを集める仕組みとドメインランダム化の道具をまとめている。強化学習と模倣学習を大規模に行うときのやり方も整理し、全身の制御、機体をまたぐ移動、接触の多い器用な操作、人の手本からの技能の学習を応用の例として示している。Newton との連携の計画にも触れている。Isaac Lab を引用するときは、この報告を挙げるよう README が求めている。
- **一次情報**: [論文 arXiv:2511.04831](https://arxiv.org/abs/2511.04831)
- **種類**: 論文
- **ライセンス**: —
- **状態**: 研究段階（論文）
- **確認日**: 2026-10-18
- **情報源**: 一次

## 解説

### 歯車の挿入の方策の学習と ROS での展開

- **概要**: Isaac Lab の公式の文書にある、強化学習で学習させた方策を実機の腕（UR10e とグリッパー）で動かすまでの手引き。歯車をつかんだ状態から、カメラで推定した軸の位置を手がかりに、関節の位置を少しずつ動かして差し込む。1つの方策で3つの大きさの歯車に対応する。学習は RSL-RL で、カメラの推定の誤差に合わせた観測のノイズ、駆動のゲインと関節の摩擦のばらつきを入れる。実機の側は、Isaac ROS の物体の位置と姿勢の推定（FoundationPose）・動作の計画（cuMotion）と、方策を動かす ROS のノードを組み合わせる。シミュレーションと実機の差を小さくするための教訓は、上の「[実機への展開](#実機への展開)」にまとめた。
- **一次情報**: [公式の文書の手引き](https://isaac-sim.github.io/IsaacLab/main/source/policy_deployment/02_gear_assembly/gear_assembly_policy.html)
- **種類**: 解説
- **ライセンス**: —
- **状態**: —（開発元の手引き）
- **確認日**: 2026-10-18
- **情報源**: 一次
