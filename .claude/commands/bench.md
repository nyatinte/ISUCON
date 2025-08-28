---
allowed-tools: ["Bash"]
description: ISUCONベンチマークを実行してスコアを測定
argument-hint: "[target-host] [options]"
---

ISUCONベンチマークを実行してスコアを測定します。

ターゲットホスト: $ARGUMENTS

!cd /home/isucon/bench && ./bench -target https://${ARGUMENTS:-localhost:443} -output bench_$(date +%Y%m%d_%H%M%S).json

実行結果を解析して以下を表示してください：
- スコア
- 成功/失敗リクエスト数
- レスポンスタイムの統計
- エラーがあればその詳細