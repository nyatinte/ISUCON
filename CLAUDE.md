# CLAUDE.md - ISUCON プロジェクト

@import .docs/isucon_node.md

## 🤖 ISUCON専用エージェント

### performance-analyzer エージェント
ISUCONのボトルネック分析と計測を専門に行うエージェント。以下の場合に必ず使用：
- 初回ベンチマーク後のボトルネック特定
- 改善後の効果測定と次のボトルネック探索
- パフォーマンス問題の深掘り調査

使用例：
```
「ボトルネックを特定して」
「パフォーマンスを計測して」
「なぜ遅いのか分析して」
```

このエージェントは以下を自動実行：
1. システムリソース（CPU/メモリ/IO）の確認
2. アクセスログ解析（alp）で遅いエンドポイント特定
3. スロークエリ分析（pt-query-digest）
4. Node.jsプロファイリング（0x）
5. 結果を総合して最重要ボトルネックを報告

## 🎯 ISUCON 基本原則

### 最重要原則

- **「推測するな、計測せよ」** - すべての改善は計測データに基づく
- **問題を最小単位に分割** - 一度に一つの改善のみ実施し、効果を個別に測定
- 計測 → 改善 → 検証のサイクルを徹底する
- 再起動試験に合格することを常に意識する（改善の永続化）

## 📊 計測ツール使用順序

### 1. システムレベル監視

```bash
# CPU/メモリ確認
top
htop

# 包括的リソース監視（CPU, ディスクI/O, ネットワーク）
dstat
```

### 2. アクセスログ分析

```bash
# Nginxアクセスログ解析
alp ltsv --file /var/log/nginx/access.log -m "/api/.*" --sort sum -r

# より詳細な解析が必要な場合
kataribe -f /var/log/nginx/access.log
```

### 3. データベース分析

```bash
# スロークエリログ有効化（my.cnf）
# slow_query_log = 1
# long_query_time = 0
# slow_query_log_file = /var/log/mysql/slow.log

# スロークエリ解析
pt-query-digest /var/log/mysql/slow.log
```

### 4. Node.js プロファイリング

```bash
# プロファイリング実行
npx 0x -- node index.js

# flamegraph.htmlをブラウザで確認
# 「Sync」で検索してブロッキング処理を特定
```

## 🚀 最重要改善ポイント

### 最初に確認すること

1. **N+1 問題** - DataLoader パターンで解決
2. **インデックス不足** - EXPLAIN でフルスキャンを確認
3. **同期処理** - fs.readFileSync などのブロッキング処理を排除

## 🎮 コマンド集

### 頻用コマンド

```bash
# ベンチマーク実行
./bench

# Git操作
git add -A && git commit -m "改善内容"
git tag score-XXXX  # スコア記録

# プロセス確認・再起動
systemctl status isucon-app
systemctl restart isucon-app
systemctl restart nginx
systemctl restart mysql

# ログ確認
tail -f /var/log/nginx/error.log
tail -f /var/log/mysql/error.log
journalctl -u isucon-app -f
```

## ⚠️ 注意事項

### やってはいけないこと

- 言語の変更（時間の無駄）
- 推測による改善（必ず計測データに基づく）
- 複数の改善を同時実施（効果測定不可）
- 再起動試験の考慮なし（失格リスク）

### 必須確認事項

- サーバーのメモリサイズ（キャッシュ設計に影響）
- CPU コア数（worker_processes 設定）
- ディスク容量（ログファイルのサイズに注意）
