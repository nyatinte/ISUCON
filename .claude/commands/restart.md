---
allowed-tools: ["Bash"]
description: アプリケーションと関連サービスを安全に再起動
argument-hint: "[all|app|db|cache|web]"
---

アプリケーションと関連サービスを安全に再起動します。

対象サービス: ${ARGUMENTS:-all}

!echo "=== Service Restart ==="
!echo "Target: ${ARGUMENTS:-all}"
!echo ""

再起動手順：

1. ヘルスチェック
!curl -s http://localhost/health || echo "Health check failed"

2. サービス再起動
!if [ "${ARGUMENTS:-all}" = "all" ] || [ "${ARGUMENTS}" = "app" ]; then
  sudo systemctl restart isucon.service
  echo "Application restarted"
fi

!if [ "${ARGUMENTS:-all}" = "all" ] || [ "${ARGUMENTS}" = "db" ]; then
  sudo systemctl restart mysql || sudo systemctl restart postgresql
  echo "Database restarted"
fi

!if [ "${ARGUMENTS:-all}" = "all" ] || [ "${ARGUMENTS}" = "cache" ]; then
  sudo systemctl restart redis || sudo systemctl restart memcached
  echo "Cache service restarted"
fi

!if [ "${ARGUMENTS:-all}" = "all" ] || [ "${ARGUMENTS}" = "web" ]; then
  sudo systemctl restart nginx || sudo systemctl restart caddy
  echo "Web server restarted"
fi

3. サービス状態確認
!sudo systemctl status isucon.service --no-pager | head -10

4. 再起動後のヘルスチェック
!sleep 3
!curl -s http://localhost/health && echo "Service is healthy" || echo "Service is not responding"