# VLM（とVLA）: 過去の項目

[`topics/vlm.md`](../vlm.md) から外した項目です。開発の停止や後継の登場などで、最新の情報としては勧めないものを、経緯が追えるように残しています。

## OSS

### Cosmos-Reason2

- **概要**: NVIDIA の、フィジカルAIとロボット向けの推論型の VLM。物理的な常識や空間・時間を踏まえて、身体を持つエージェントの次の行動を言語で計画する。2B・8B（2025-12）と 32B（2026-04）が公開された。前の版 Cosmos-Reason1 は 2025-03 に論文が出ている。Isaac GR00T N1.7 の土台の VLM としても使われている。
- **一次情報**: [公式リポジトリ](https://github.com/nvidia-cosmos/cosmos-reason2)・[論文 arXiv:2503.15558（Cosmos-Reason1）](https://arxiv.org/abs/2503.15558)
- **種類**: OSS
- **ライセンス**: コードは Apache-2.0、モデルの重みは NVIDIA Open Model License（README の表記）
- **状態**: 開発終了（README で、活発な開発を終えて最小限の保守だけになったとしている）
- **確認日**: 2026-10-17
- **情報源**: 一次
- **移した日・理由**: 2026-10-17。README で、活発な開発を終えて最小限の保守だけになったとし、後継として Cosmos 3 を案内しているため。後継: [Cosmos 3](../vlm.md#cosmos-3)
