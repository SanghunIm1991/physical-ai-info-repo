# NVIDIA のフィジカルAI開発技術

NVIDIA は、フィジカルAIの開発に使う計算機・ソフトウェア・モデル・データを、まとめて提供しています。このファイルでは、その全体の構成と、主な技術を整理します。1つの企業の技術を集めたテーマなので、性能や使いやすさの評価は載せず、何を提供しているかを開発元の一次情報から整理します。開発元の主張は「開発元によれば」として書きます。

- **選び方**: 開発元が示す構成（下の「3つの計算機」）の各部分から、ロボットと自動運転の開発に直接かかわるものを選びました。同じ目的のものは1つの項目にまとめました（例: Alpamayo の 1 と 1.5、GR00T-Mimic と GR00T-Dreams）。
- **ほかのファイルに載せているもの**: 次の技術は、それぞれのテーマのファイルに項目があるので、このファイルでは項目にせず、下の表で位置づけだけを示します。
  - [Cosmos 3](vlm.md#cosmos-3)（VLM・世界モデルとして使える基盤モデル）と [Isaac GR00T](vlm.md#isaac-gr00t)（人型ロボットなどに向けた VLA）: [`vlm.md`](vlm.md)
  - [Isaac Sim と Isaac Lab](simulator.md#isaac-sim-と-isaac-lab)（シミュレータと学習の枠組み）と [Newton](simulator.md#newton)（物理エンジン。NVIDIA が Google DeepMind・Disney Research とともに始め、現在は Linux Foundation のプロジェクト）: [`simulator.md`](simulator.md)
  - 導入の事例（Foxconn のヒト型ロボットなど）: [`industry.md`](industry.md)
  - Isaac Lab での開発の詳細（環境の設計・学習・実機への展開）: [`isaac-lab.md`](isaac-lab.md)
- **見出しの分け方**: 項目は、種類ではなく技術の役割ごとの見出し（全体像・モデル・シミュレーションとデータ・ロボットのソフトウェアと計算機）に並べています。
- **今回は載せていないもの**: 学習用の計算機（DGX）そのもの、センサーのデータを処理する基盤（Holoscan。[Jetson Thor](#jetson-thor) の項目で、そこで動かすソフトウェアとして触れるだけにした）、工場・都市の映像解析向けの基盤（Metropolis）、自動運転の車載の計算機と認証のための基盤（DRIVE）。フィジカルAIの開発の入口からは一段遠いと判断しました。

## 3つの計算機と各技術の位置づけ

開発元は、フィジカルAIの開発を、次の3種類の計算機の役割に分けて説明しています（[3つの計算機の構成](#3つの計算機の構成)）。

| 役割 | 計算機 | このリポジトリで扱う技術 |
|---|---|---|
| 学習（基盤モデルの事前学習、ロボットの方策の追加学習） | DGX | [Cosmos 3](vlm.md#cosmos-3)、[Isaac GR00T](vlm.md#isaac-gr00t)、[Alpamayo](#alpamayo) |
| シミュレーションと学習用データの生成 | Omniverse と Cosmos を動かすサーバー（RTX PRO Server） | [Omniverse と OpenUSD](#omniverse-と-openusd)、[PhysX](#physx)、[Isaac Sim と Isaac Lab](simulator.md#isaac-sim-と-isaac-lab)、[Newton](simulator.md#newton)、[AlpaSim](#alpasim)、[Isaac GR00T のデータ生成の設計図](#isaac-gr00t-のデータ生成の設計図)、[Physical AI データセット](#physical-ai-データセット) |
| ロボットの上での実行 | Jetson Thor | [Jetson Thor](#jetson-thor)、[Isaac ROS](#isaac-ros) |

- **流れの例（人型ロボット）**: 各項目の一次情報をもとに、このリポジトリがまとめた例です（データの生成の部分は [GR00T-Mimic の開発元の解説](https://developer.nvidia.com/blog/building-a-synthetic-motion-generation-pipeline-for-humanoid-robot-learning) による）。人の操作の記録を、GR00T-Mimic で数を増やし、Cosmos で見た目を多様にする → そのデータで、Isaac Lab で方策を学習させる、または Isaac GR00T を追加学習する → Isaac Sim で試す → Jetson Thor の上で、Isaac ROS などと組み合わせて動かす。
- **NVIDIA の技術で閉じているわけではない。** 例えば Isaac GR00T の追加学習のデータは、LeRobot（Hugging Face のロボット学習の枠組み）の形式を拡張したものを使い（[Isaac GR00T の公式リポジトリ](https://github.com/NVIDIA/Isaac-GR00T)の README の表記）、Isaac ROS は ROS 2 の上で動きます。一方、GPU で計算する部分の多くは NVIDIA の GPU を前提にしています。
- **ライセンスは技術ごとに違う。** コードは Apache-2.0 などのオープンソースのライセンスが多い一方、モデルの重みとデータは NVIDIA の独自の条件（NVIDIA Open Model License、NVIDIA の自動運転のデータセットの利用許諾など）や OpenMDW-1.1 のものがあります。使う前に各項目の一次情報で確かめてください。

## 一覧

| 項目 | 分類 | 種類 | 状態 | 確認日 |
|---|---|---|---|---|
| [3つの計算機の構成](#3つの計算機の構成) | 全体像 | 解説 | — | 2026-10-18 |
| [Alpamayo](#alpamayo) | モデル（自動運転） | OSS | 研究段階 | 2026-10-18 |
| [Omniverse と OpenUSD](#omniverse-と-openusd) | シミュレーションの基盤 | 製品 | 実用 | 2026-10-18 |
| [PhysX](#physx) | シミュレーションの基盤 | OSS | 実用 | 2026-10-18 |
| [AlpaSim](#alpasim) | シミュレーション（自動運転） | OSS | 研究段階 | 2026-10-18 |
| [Isaac GR00T のデータ生成の設計図](#isaac-gr00t-のデータ生成の設計図) | 学習用データの生成 | OSS | 研究段階 | 2026-10-18 |
| [Physical AI データセット](#physical-ai-データセット) | 学習用データ | データセット | 実用 | 2026-10-18 |
| [Isaac ROS](#isaac-ros) | ロボットの上での実行 | OSS | 実用 | 2026-10-18 |
| [Jetson Thor](#jetson-thor) | ロボットの上での実行 | 製品 | 実用 | 2026-10-18 |

## 全体像

### 3つの計算機の構成

- **概要**: NVIDIA が、ロボットの開発に要る計算機を3つの役割で説明した記事（2024-10 に公開、2025-08 に更新）。(1)モデルを学習させる DGX、(2)仮想の環境での試験と学習用データの生成を担う、Omniverse と Cosmos を動かすサーバー（RTX PRO Server）、(3)ロボットの上でセンサーのデータを処理しモデルを動かす Jetson AGX Thor、の3つである。各部分で使うソフトウェアとして、Cosmos・Isaac GR00T・Isaac Sim・Isaac Lab・工場のデジタルツインの設計図 Mega を挙げている。NVIDIA の製品の紹介を兼ねた記事なので、構成の考え方をつかむために読む。
- **一次情報**: [開発元のブログ](https://blogs.nvidia.com/blog/three-computers-robotics)
- **種類**: 解説
- **ライセンス**: —
- **状態**: —（開発元の説明の記事）
- **確認日**: 2026-10-18
- **情報源**: 一次

## モデル

### Alpamayo

- **概要**: NVIDIA の、自動運転に向けた VLA（パラメータ数は 10B）。車のカメラの映像から、走る軌跡（先の 6.4 秒分）と、その判断の理由を言葉で説明した推論の過程を、あわせて出力する。土台に、NVIDIA の推論型の VLM（Cosmos-Reason）を使う。2026-03 に出た **Alpamayo 1.5** は、強化学習での追加の学習、道案内の指示の入力、カメラの台数の変更、映像についての質問への回答に対応した。追加の学習（教師ありの学習と強化学習）の手順は、別のリポジトリ（Alpamayo Recipes）で公開している。最初の版（Alpamayo 1）は旧名 Alpamayo-R1 で、そのリポジトリは保守だけの段階に入り、1.5 を勧めている。
- **一次情報**: [Alpamayo 1.5 の公式リポジトリ](https://github.com/NVlabs/alpamayo1.5)・[Alpamayo 1 の公式リポジトリ](https://github.com/NVlabs/alpamayo)・[論文 arXiv:2511.00088（Alpamayo 1）](https://arxiv.org/abs/2511.00088)・[Alpamayo 1.5 のモデルの公開ページ](https://huggingface.co/nvidia/Alpamayo-1.5-10B)
- **種類**: OSS
- **ライセンス**: コードは Apache-2.0、モデルの重みは OpenMDW-1.1（README の表記）。商用での利用について、リポジトリの FAQ は「ライセンスとモデルの説明に従って可能」とする一方、モデルの公開ページは「非商用の利用に向けた準備ができている。商用の許諾は要望に応じる」としており、表記がそろっていない
- **状態**: 研究段階（README の注意書きで、研究・実験・評価に向けたものとし、自動運転の仕組みの全体ではなく、車載の品質の検証もしていないとしている）
- **確認日**: 2026-10-18
- **情報源**: 一次

## シミュレーションとデータ

### Omniverse と OpenUSD

- **概要**: **Omniverse** は、フィジカルAIのための仮想の空間（シミュレーションにそのまま使える3D の世界）を作るための、NVIDIA のライブラリ・道具・設計図の集まり。3D の場面の共通の形式である **OpenUSD** を土台にし、物理の計算（PhysX・Newton）、GPU での描画とセンサーの模擬、実世界のデータからの場面の再構成（NuRec）、URDF・MuJoCo などの形式からの変換などの道具をそろえる。開発元は近年、人だけでなく AI のエージェントが呼び出して使う道具としても位置づけている。設計図（参照用の作業の流れ）として、複数のロボットが働く工場のデジタルツイン（Mega）や、ロボットの操作の学習用データの生成（GR00T-Mimic）などを用意している。Isaac Sim は、Omniverse の上で動くアプリケーションである。
- **一次情報**: [開発元の Omniverse のページ](https://developer.nvidia.com/omniverse)
- **種類**: 製品
- **ライセンス**: 部品ごとに異なる（開発元のページは、一部の道具をオープンソースとしている。例: PhysX は BSD-3-Clause）
- **状態**: 実用（開発元が製品として提供している）
- **確認日**: 2026-10-18
- **情報源**: 一次

### PhysX

- **概要**: NVIDIA の物理エンジンで、リアルタイムの計算を目的とする。Omniverse の物理の計算の部分（Omniverse PhysX の拡張）として使われ、Isaac Sim もその上で動く。リポジトリには、本体のほか、OpenUSD で書いた場面の物理を Python から計算する窓口（ovphysx）、物の破壊（Blast）、流体や炎（Flow）のライブラリが入っている。開発元は、Omniverse・PhysX・Isaac Sim の版の対応を表で示している。
- **一次情報**: [公式リポジトリ](https://github.com/NVIDIA-Omniverse/PhysX)
- **種類**: OSS
- **ライセンス**: BSD-3-Clause
- **状態**: 実用（正式な版の公開が続いている。確認日の時点で、本体の SDK は 5.12.0 までの版が示されている）
- **確認日**: 2026-10-18
- **情報源**: 一次

### AlpaSim

- **概要**: NVIDIA の、自動運転の方策（運転の判断をするモデル）を試すためのシミュレータ。方策の出力に応じて場面が変わる「閉ループ」の試験を行う。描画・交通・車の動き・制御を別々のプロセスに分け、gRPC でつなぐ構成で、部品を差し替えたり、複数の GPU や計算機に分けて動かしたりできる。描画は、実際の走行のデータから再構成した場面（NuRec）を既定とし、動画を生成するモデルによる描画にも対応する。README によれば、Alpamayo のほか、他の開発元の運転の方策（VaVAM・Transfuser）にも対応する。
- **一次情報**: [公式リポジトリ](https://github.com/NVlabs/alpasim)
- **種類**: OSS
- **ライセンス**: コードは Apache-2.0。場面のデータは、下の Physical AI データセットの条件による
- **状態**: 研究段階（README で研究と開発のためのものとしている。正式な版の番号は示されていない）
- **確認日**: 2026-10-18
- **情報源**: 一次

### Isaac GR00T のデータ生成の設計図

- **概要**: 人型ロボットなどの学習用データを人工的に増やすための、NVIDIA の2つの参照用の作業の流れ。**GR00T-Mimic**（2025-01 に初出）は、少数の人の操作の記録を、Isaac Lab で多数の動きの軌跡に増やし、背景や照明を変えたうえで Cosmos で見た目を写実的にする。今ある作業を得意にさせるためのもの。**GR00T-Dreams**（論文は 2025-05。論文での名前は DreamGen）は、1枚の画像と言葉の指示から、世界モデル（Cosmos Predict）でロボットが新しい作業をする動画を作り、その動画から動作を推定して学習用データにする。新しい作業や環境に広げるためのもの。開発元によれば、GR00T-Dreams で作ったデータを使って、Isaac GR00T N1.5 を 36 時間で開発した。
- **一次情報**: [GR00T-Mimic の公式リポジトリ](https://github.com/NVIDIA-Omniverse-blueprints/synthetic-motion-generation)・[GR00T-Mimic の開発元の解説](https://developer.nvidia.com/blog/building-a-synthetic-motion-generation-pipeline-for-humanoid-robot-learning)・[GR00T-Dreams の公式リポジトリ](https://github.com/NVIDIA/GR00T-Dreams)・[GR00T-Dreams の開発元の解説](https://developer.nvidia.com/blog/enhance-robot-learning-with-synthetic-trajectory-data-generated-by-world-foundation-models)・[論文 arXiv:2505.12705（DreamGen）](https://arxiv.org/abs/2505.12705)
- **種類**: OSS
- **ライセンス**: GR00T-Dreams は Apache-2.0。GR00T-Mimic のリポジトリは、それ自体のライセンスを示さず、使う部品（Isaac Sim・Isaac Lab・Cosmos のモデル）の条件を挙げている
- **状態**: 研究段階（どちらも参照用の作業の流れで、正式な版の番号は示されていない。GR00T-Mimic のリポジトリは Isaac Sim 4.5.0・Isaac Lab 2.0.2 を使う時期のもので、[`simulator.md`](simulator.md#isaac-sim-と-isaac-lab) に載せた Isaac Sim の最新の版〈6.2.0、2026-10-17 に確認〉より古い）
- **確認日**: 2026-10-18
- **情報源**: 一次

### Physical AI データセット

- **概要**: NVIDIA が Hugging Face で公開している、フィジカルAIの開発のためのデータセットの集まり。開発元は「商用に使える品質のオープンなデータセット」としている。自動運転（25か国・2,500以上の都市で集めた約 1,700 時間の走行のデータ。カメラ7台の映像、LiDAR、レーダーを含む）、ロボットの操作（Isaac GR00T の追加学習用の、シミュレーションと人の操作の記録の軌跡など）、倉庫などの場面、世界モデル用の人工的な場面などが含まれる。各データセットの説明によれば、自動運転のデータセットは Alpamayo に、ロボットの操作の例のデータセットは Isaac GR00T の追加学習に使われている。
- **一次情報**: [データセットの集まりのページ](https://huggingface.co/collections/nvidia/physical-ai)・[自動運転のデータセット](https://huggingface.co/datasets/nvidia/PhysicalAI-Autonomous-Vehicles)・[Isaac GR00T の追加学習用のデータセットの例](https://huggingface.co/datasets/nvidia/PhysicalAI-Robotics-GR00T-X-Embodiment-Sim)
- **種類**: データセット
- **ライセンス**: データセットごとに異なる。例えば、自動運転のデータセットは NVIDIA の独自の利用許諾（利用の申し込みが要る。用途を自動運転と運転支援の社内の開発に限り、再配布を禁じ、期限は取得から12か月）、Isaac GR00T の追加学習用の例は CC BY 4.0
- **状態**: 実用（版を重ねて公開され、自動運転のデータセットは確認日の時点で 26.09 が最新）
- **確認日**: 2026-10-18
- **情報源**: 一次

## ロボットのソフトウェアと計算機

### Isaac ROS

- **概要**: ROS 2 でロボットを作るための、NVIDIA の GPU で速く動くパッケージの集まり。カメラによる自己位置の推定と地図づくり（Visual SLAM）、3D の地図と障害物の把握（nvblox）、ステレオカメラからの深度の推定（FoundationStereo）、初めて見る物の位置と姿勢の推定（FoundationPose）、腕の動きの計画（cuMotion。MoveIt の拡張として使える）、ROS 2 のデータの受け渡しを速くする仕組み（NITROS）などを含む。Isaac Sim での試験から、Jetson の上での実行までを想定し、腕の操作と移動ロボットの参照用の作業の流れも用意している。
- **一次情報**: [開発元の Isaac ROS のページ](https://developer.nvidia.com/isaac/ros)・[公式の GitHub の組織](https://github.com/NVIDIA-ISAAC-ROS)・[リリースノート](https://nvidia-isaac-ros.github.io/releases/index.html)
- **種類**: OSS
- **ライセンス**: Apache-2.0（公式の GitHub の組織で確かめた主なリポジトリの表記。パッケージごとに確かめること）
- **状態**: 実用（正式な版の公開が続いており、確認日の時点の最新は 5.0.0〈2026-09-21〉。対応する ROS 2 の版は Lyrical Luth）
- **確認日**: 2026-10-18
- **情報源**: 一次

### Jetson Thor

- **概要**: ロボットに載せて AI を動かすための、NVIDIA の小型の計算機（Blackwell 世代の GPU を使う）。上位の T5000（メモリ 128 GB、消費電力 40〜130 W）と、下位の T4000（メモリ 64 GB、40〜70 W）の2種類の部品があり、開発用のキット（Jetson AGX Thor Developer Kit）は T5000 を使う。開発元によれば、T5000 は前の世代の Jetson AGX Orin と比べて、AI の計算の性能が最大 7.5 倍、電力あたりの性能が 3.5 倍である。Isaac GR00T・Isaac の各ソフトウェア・センサーのデータを処理する仕組み（Holoscan）を動かす先として位置づけられている。
- **一次情報**: [開発元の Jetson Thor のページ](https://www.nvidia.com/en-us/autonomous-machines/embedded-systems/jetson-thor/)
- **種類**: 製品
- **ライセンス**: —
- **状態**: 実用（開発元と販売の協力企業を通じて販売されている）
- **確認日**: 2026-10-18
- **情報源**: 一次
