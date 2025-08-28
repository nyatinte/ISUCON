---
allowed-tools: ["Bash"]
description: システムリソースをリアルタイム監視
argument-hint: "[duration-seconds]"
---

システムリソース（CPU、メモリ、ディスクI/O、ネットワーク）をモニタリングします。

監視時間: ${ARGUMENTS:-60}秒

!echo "=== System Resource Monitoring ==="
!echo "Duration: ${ARGUMENTS:-60} seconds"
!echo ""

以下のコマンドを並行して実行し、結果を分析：

1. CPU使用率の監視
!vmstat 1 ${ARGUMENTS:-60} | tee /tmp/vmstat.log

2. メモリ使用状況
!free -h -s 1 -c ${ARGUMENTS:-60} | tee /tmp/memory.log

3. ディスクI/O
!iostat -x 1 ${ARGUMENTS:-60} | tee /tmp/iostat.log

4. ネットワーク接続
!ss -s ; netstat -i

5. プロセス状況
!top -b -n ${ARGUMENTS:-60} -d 1 | head -50

監視結果から以下を報告してください：
- ボトルネックになっているリソース
- 異常値の検出
- チューニングポイントの提案