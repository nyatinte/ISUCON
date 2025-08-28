---
allowed-tools: ["Bash", "Read"]
description: コード変更をデプロイして本番環境に反映
argument-hint: "[branch-name]"
---

コードの変更を安全にデプロイします。

ブランチ: ${ARGUMENTS:-main}

デプロイ手順を実行：

1. 現在の状態を保存
!git stash push -m "Before deployment $(date +%Y%m%d_%H%M%S)"

2. 最新コードを取得
!git fetch origin
!git checkout ${ARGUMENTS:-main}
!git pull origin ${ARGUMENTS:-main}

3. 依存関係の更新
!if [ -f go.mod ]; then
  go mod download
  go build -o app
elif [ -f package.json ]; then
  npm install --production
elif [ -f requirements.txt ]; then
  pip install -r requirements.txt
elif [ -f Gemfile ]; then
  bundle install
fi

4. データベースマイグレーション（必要な場合）
!if [ -d migrations ] || [ -d db/migrate ]; then
  echo "Migrations detected - please run manually if needed"
fi

5. アセットのビルド
!if [ -f webpack.config.js ]; then
  npm run build
fi

6. サービス再起動
!sudo systemctl restart isucon.service
!sleep 3

7. デプロイ確認
!curl -s http://localhost/health && echo "Deployment successful" || echo "Deployment verification failed"

8. ベンチマーク実行（オプション）
!echo "Run benchmark to verify performance: /bench"