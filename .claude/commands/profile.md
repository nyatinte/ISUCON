---
allowed-tools: ["Bash", "Read", "Write"]
description: アプリケーションのプロファイリングを実行
argument-hint: "[go|python|ruby|node] [duration-seconds]"
---

アプリケーションのプロファイリングを実行して、CPUやメモリの使用状況を分析します。

言語: $ARGUMENTS

!echo "プロファイリングを開始します..."

対象言語に応じて適切なプロファイリングツールを使用：

- Go: pprof を使用
  - CPU プロファイル: `go tool pprof -http=:6060 cpu.prof`
  - メモリプロファイル: `go tool pprof -http=:6060 mem.prof`
  
- Python: py-spy または cProfile を使用
  - `py-spy record -o profile.svg --duration 30 -- python app.py`
  
- Ruby: stackprof または ruby-prof を使用
  - `bundle exec stackprof --mode cpu --out tmp/stackprof.dump`
  
- Node.js: clinic または 0x を使用
  - `clinic doctor -- node app.js`

プロファイリング結果から以下を分析してください：
1. CPU使用率が高い関数
2. メモリリークの可能性がある箇所
3. 最適化可能な処理
4. 改善提案