# 更新履歴

追加・見直し・アーカイブへの移動を、新しいものを上にして記録します。日付は YYYY-MM-DD です。

## 2026-11-13

- `topics/cosmos.md` を追加（NVIDIA Cosmos を使ったフィジカルAIの開発。エンジニア向けの開発の要点〈版と系譜・2つの塔の構成・動かす・追加学習・ロボットでの使い方・注意点〉と5項目: Cosmos Framework、Cosmos Curator、Cosmos Evaluator、Cosmos 3 の作例集、Cosmos 3 の開発元の技術解説）
- `topics/gr00t.md` を追加（Isaac GR00T を使ったフィジカルAIの開発。エンジニア向けの開発の要点〈版の変遷・構成・データの形式・機体のタグ・追加学習・推論と実機への展開・評価・ライセンス〉と3項目: GR00T-WholeBodyControl、LeRobot の GR00T 対応、DreamZero）
- `topics/vlm.md` の Cosmos 3 と Isaac GR00T を確かめ直した（GR00T N1.7 の段階の表記が資料によって違うこと、N2 が未公開であることを追記し、新しいファイルへ案内した）。`topics/nvidia.md`・README・入門書から新しいファイルへ案内した

## 2026-10-18

- `topics/isaac-lab.md` を追加（Isaac Lab を使ったフィジカルAIの開発。エンジニア向けの開発の要点〈版の選び方・環境の設計・アクチュエータのモデル・強化学習・模倣学習・実機への展開〉と9項目: Isaac Lab 3.0、RSL-RL、skrl、Isaac Lab Mimic と SkillGen、Isaac Lab-Arena、LEAPP と Isaac ROS Deploy、HOVER、Isaac Lab の技術報告、歯車の挿入の方策の学習と ROS での展開）。`topics/simulator.md`・`topics/nvidia.md` から案内した

- `topics/nvidia.md` を追加（NVIDIA のフィジカルAI開発技術。3つの計算機と各技術の位置づけの説明と9項目: 3つの計算機の構成、Alpamayo、Omniverse と OpenUSD、PhysX、AlpaSim、Isaac GR00T のデータ生成の設計図、Physical AI データセット、Isaac ROS、Jetson Thor）。項目の種類に「データセット」を加えた
- `topics/vlm.md` の Cosmos 3 を確かめ直した（小型の Edge が 2026-07 に公開されたことを反映）

## 2026-10-17

- `topics/industry.md` を見直し（事例の状態を「実用／試験導入」に変更、鹿島建設の A4CSEL を試験導入に変更、選び方と載せていないものを明記、Mercedes-Benz とコマツの記述を一次情報で確かめ直した）。`topics/simulator.md` の選び方の説明を正した
- `topics/industry.md` を追加（業界ごとの導入の状況。業界横断1・製造業2・販売業2・自動車業4・建設業3・医療と介護2の14項目）。項目の種類に「事例」「報告」を加えた
- `topics/simulator.md` を追加（シミュレータの分類と選ぶときの観点の説明と8項目: MuJoCo、Newton、Isaac Sim と Isaac Lab、ManiSkill、Genesis World、Gazebo、O3DE、シミュレータと世界モデルの総説）。入門書の案内も更新
- 入門書 `primer.md` を追加（全体像・3つの働き・技術の地図〈図〉・テーマの位置づけ・用語集）
- `topics/vlm.md` を追加（VLM と VLA の関係の説明と9項目）。後継が出ていた Cosmos-Reason2 は `topics/archive/vlm.md` に置いた
- `topics/rai.md` を追加（RAI 本体、agentic-mobile-manipulator、RAI の論文、RHO の論文の4項目）
- リポジトリの骨組みを作成（README・LICENSE・更新履歴・項目の書式・調べ方の手順書）
