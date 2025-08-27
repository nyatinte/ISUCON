# ISUCON初参加者向け完全ガイド - Node.js実践編

## パフォーマンス計測の具体的手法

Node.jsアプリケーションの性能測定は、ISUCONで高スコアを獲得する上で最も重要なステップです。以下の計測ツールと手法を組み合わせることで、ボトルネックを特定し、効果的な最適化が可能になります。

### 主要計測ツールと使用方法

**0x - フレームグラフプロファイラー（推奨度：★★★★★）**

0xは使いやすさと詳細な分析能力を兼ね備えた、ISUCON向けの最適なプロファイリングツールです。

```bash
# インストール
npm install -g 0x

# 基本的な使用方法
0x -- node server.js

# 統合負荷テスト付き実行
0x -P 'autocannon localhost:$PORT' -- node server.js

# 本番環境でのデータ収集のみ
0x --collect-only -- node server.js

# 後から可視化
0x --visualize-only /path/to/profile/folder
```

分析時のポイント：

- 「Tiers」ビューを有効化し、「core」と「inlinable」を無効化すると、アプリケーション固有の処理が明確になります
- 幅の広い水平セクションがCPU時間の大部分を占めている箇所で、これが最適化対象となります
- 暗赤色のセクションはイベントループをブロックしている「ホット」な関数を示しています

**Node.js Chrome DevTools統合**

```bash
# インスペクター付きで起動
node --inspect server.js

# Chrome で chrome://inspect を開く
# 「Open dedicated DevTools for Node」をクリック
```

Performanceパネルでの分析ポイント：

- **Main thread timeline**: 関数呼び出しスタックの時系列表示
- **Bottom-Up view**: CPU時間を最も消費している関数
- **Call Tree**: 関数呼び出しの階層表示
- **Event Log**: タイムスタンプ付きの詳細イベント

**autocannon - 負荷テストツール**

ISUCONのベンチマーカーに近い負荷パターンを再現できる重要なツールです。

```bash
# 高並列性テスト
autocannon -c 100 -d 30 -p 10 http://localhost:3000

# POST リクエストでの負荷テスト
autocannon -m POST -H "Content-Type: application/json" \
  -b '{"key":"value"}' -c 50 -d 20 http://localhost:3000/api

# 複数ワーカーで最大負荷
autocannon -c 100 -d 30 -w 4 http://localhost:3000
```

出力の読み方：

- **Requests/sec**: スループット（高いほど良い）
- **Latency percentiles**: レスポンス時間の分布
- **P99**: 99%のリクエストがこの時間以内に処理される（低いほど良い）
- **Errors**: 失敗リクエスト（0であるべき）

### ボトルネック特定の具体的手法

**CPU vs I/Oボトルネックの識別**

```javascript
// イベントループ監視による検出
const blocked = require('blocked-at');

blocked((time, stack) => {
  console.log(`Blocked for ${time}ms, operation started here:`, stack);
}, { threshold: 10 }); // 10ms以上のブロッキングを検出

// カスタムイベントループモニタリング
function measureEventLoop() {
  const start = process.hrtime();
  
  setImmediate(() => {
    const delta = process.hrtime(start);
    const millisec = delta[0] * 1e3 + delta[1] / 1e6;
    
    if (millisec > 10) {
      console.warn(`Event loop blocked for ${millisec.toFixed(2)}ms`);
    }
    
    measureEventLoop();
  });
}

measureEventLoop();
```

**データベースクエリ性能分析**

```javascript
// MySQL クエリログ設定
const mysql = require('mysql2');
const connection = mysql.createConnection({
  // ... 設定
  debug: true, // クエリログ有効化
});

// スロークエリの検出と記録
function logQuery(query, params, callback) {
  const start = process.hrtime();
  connection.query(query, params, (err, results) => {
    const [seconds, nanoseconds] = process.hrtime(start);
    const duration = seconds * 1000 + nanoseconds / 1000000;
    if (duration > 100) { // 100ms以上のクエリをログ
      console.warn(`Slow query (${duration.toFixed(2)}ms):`, query);
    }
    callback(err, results);
  });
}
```

### ベンチマーク結果の分析手法

ISUCONでの最適化サイクル：

1. **ベースライン測定**

```bash
0x -- node server.js &
SERVER_PID=$!
autocannon -c 50 -d 60 http://localhost:3000
kill -SIGINT $SERVER_PID
```

2. **ホットスポットの特定**
   - フレームグラフの幅広いセクションを確認
   - CPU集約的な関数を識別
   - ブロッキング操作を検出

3. **改善の測定と検証**
   - 同じ条件で再プロファイリング
   - スコアの改善を確認
   - エラー率が5%未満であることを確認

## ISUCONで頻出する課題と具体的対処法

### N+1問題

**問題のパターン**

```javascript
// 問題のあるコード - N+1クエリ
const users = await User.findAll(); // 1クエリ
for (const user of users) {
  user.posts = await Post.findAll({ where: { userId: user.id } }); // Nクエリ
}
```

**DataLoaderパターンによる解決**

```javascript
const DataLoader = require('dataloader');

// PostのDataLoader作成
const postLoader = new DataLoader(async (userIds) => {
  const posts = await Post.findAll({
    where: { userId: { $in: userIds } }
  });
  
  // userIdごとにグループ化
  const postsMap = posts.reduce((map, post) => {
    if (!map[post.userId]) map[post.userId] = [];
    map[post.userId].push(post);
    return map;
  }, {});
  
  // リクエストされたuserIdsの順序で結果を返す
  return userIds.map(userId => postsMap[userId] || []);
});

// 使用例 - 単一クエリで全ポストを解決
const users = await User.findAll();
await Promise.all(users.map(async user => {
  user.posts = await postLoader.load(user.id);
}));
```

### データベース関連の最適化

**インデックス最適化とEXPLAIN**

```sql
-- よくあるクエリパターン用のインデックス追加
CREATE INDEX idx_posts_user_id ON posts(user_id);
CREATE INDEX idx_posts_created_at ON posts(created_at);
CREATE INDEX idx_posts_user_created ON posts(user_id, created_at);
```

**コネクションプール設定（MySQL）**

```javascript
const mysql = require('mysql2/promise');

const pool = mysql.createPool({
  host: 'localhost',
  database: 'mydb',
  connectionLimit: 20,          // 最大接続数
  acquireTimeout: 60000,        // 接続取得タイムアウト
  timeout: 60000,               // クエリタイムアウト
  queueLimit: 0,                // 無制限キュー
  namedPlaceholders: true,      // 名前付きプレースホルダ有効
});
```

**プリペアドステートメント**

```javascript
// MySQL
const statement = await pool.prepare('SELECT * FROM users WHERE id = ? AND status = ?');
const [rows] = await statement.execute([userId, 'active']);
await statement.close();

// PostgreSQL
const query = {
  name: 'fetch-user',
  text: 'SELECT * FROM users WHERE id = $1 AND status = $2',
  values: [userId, 'active']
};
const result = await pool.query(query);
```

### キャッシュ戦略

**Redisによる多層キャッシュ実装**

```javascript
class MultiLayerCache {
  constructor(l1Cache, redisClient) {
    this.l1 = l1Cache;  // メモリ内LRUキャッシュ
    this.l2 = redisClient;  // Redisキャッシュ
  }
  
  async get(key) {
    // L1キャッシュを最初に試行
    let value = this.l1.get(key);
    if (value) return value;
    
    // L2キャッシュを試行
    const cached = await this.l2.get(key);
    if (cached) {
      value = JSON.parse(cached);
      this.l1.set(key, value);  // L1にも格納
      return value;
    }
    
    return null;
  }
  
  async set(key, value, ttl) {
    this.l1.set(key, value);
    await this.l2.setex(key, ttl, JSON.stringify(value));
  }
}
```

**LRUキャッシュによるメモリ内キャッシング**

```javascript
const LRU = require('lru-cache');

const cache = new LRU({
  max: 500,                    // 最大アイテム数
  maxAge: 1000 * 60 * 15       // 15分のTTL
});

const getWithCache = async (key, fetchFunction) => {
  let result = cache.get(key);
  
  if (!result) {
    result = await fetchFunction();
    cache.set(key, result);
  }
  
  return result;
};
```

### その他の頻出問題と対策

**静的ファイル配信の最適化**

Nginxを使用した静的ファイル配信（推奨）：

```nginx
server {
    listen 80;
    
    # 静的ファイルを直接配信
    location /static/ {
        alias /var/www/static/;
        expires 1y;
        add_header Cache-Control "public, immutable";
        gzip on;
        gzip_types text/css application/javascript;
    }
    
    # 動的コンテンツはNode.jsへプロキシ
    location / {
        proxy_pass http://localhost:3000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

**ログ出力によるI/O負荷の削減**

```javascript
// バッチロギングでI/O負荷を削減
class BatchLogger {
  constructor(logger, batchSize = 100, flushInterval = 5000) {
    this.logger = logger;
    this.batch = [];
    this.batchSize = batchSize;
    
    setInterval(() => this.flush(), flushInterval);
  }
  
  log(level, message, meta = {}) {
    this.batch.push({ level, message, meta, timestamp: new Date() });
    
    if (this.batch.length >= this.batchSize) {
      this.flush();
    }
  }
  
  flush() {
    if (this.batch.length === 0) return;
    
    const toLog = this.batch.splice(0);
    for (const entry of toLog) {
      this.logger.log(entry.level, entry.message, entry.meta);
    }
  }
}
```

**セッション管理の最適化（JWT推奨）**

```javascript
const jwt = require('jsonwebtoken');

const generateJWT = (user) => {
  return jwt.sign(
    { userId: user.id, role: user.role },
    process.env.JWT_SECRET,
    { expiresIn: '1h' }
  );
};

// 認証ミドルウェア
const authenticateJWT = (req, res, next) => {
  const token = req.headers.authorization?.split(' ')[1];
  
  if (!token) {
    return res.status(401).json({ error: 'No token provided' });
  }
  
  try {
    req.user = jwt.verify(token, process.env.JWT_SECRET);
    next();
  } catch (error) {
    res.status(403).json({ error: 'Invalid token' });
  }
};
```

**画像処理の最適化（Sharp推奨 - Jimpの4-5倍高速）**

```javascript
const sharp = require('sharp');

// ストリーム処理で大容量ファイルに対応
const processStream = (inputStream, outputStream) => {
  return inputStream
    .pipe(sharp()
      .resize(800, 600)
      .jpeg({ quality: 80, mozjpeg: true })
    )
    .pipe(outputStream);
};

// バッチ処理
const processBatch = async (images) => {
  return Promise.all(
    images.map(async (image) => {
      return sharp(image.path)
        .resize(800, 600)
        .jpeg({ quality: 80, mozjpeg: true })
        .toFile(image.outputPath);
    })
  );
};
```

## Node.js特有の最適化技法

### ClusterとWorker Threadsの使い分け

**Cluster API - I/Oバウンドタスク向け**

```javascript
import cluster from 'cluster';
import { cpus } from 'os';

const numCPUs = cpus().length;

if (cluster.isPrimary) {
    // ワーカープロセスをフォーク
    for (let i = 0; i < numCPUs; i++) {
        cluster.fork();
    }
    
    // 死んだワーカーを再起動
    cluster.on('exit', (worker, code, signal) => {
        console.log(`Worker ${worker.process.pid} died`);
        cluster.fork();
    });
} else {
    // ワーカープロセスでサーバー起動
    app.listen(8000);
}
```

**Worker Threads - CPUバウンドタスク向け**

```javascript
import { Worker, isMainThread, parentPort, workerData } from 'worker_threads';

if (isMainThread) {
    // CPU集約的タスクをワーカーで実行
    const worker = new Worker(new URL(import.meta.url), {
        workerData: { task: 'heavy-computation' }
    });
    
    worker.on('message', (result) => {
        console.log('Result:', result);
    });
} else {
    // ワーカー内でタスク実行
    const result = performHeavyComputation(workerData);
    parentPort.postMessage(result);
}
```

### Stream APIによるメモリ効率化

**大容量ファイル処理の改善**

```javascript
const { pipeline } = require('stream/promises');
const fs = require('fs');
const crypto = require('crypto');

// ストリームベース（メモリ効率的）
await pipeline(
    fs.createReadStream('./largefile.txt'),
    crypto.createHash('sha256'),
    fs.createWriteStream('./checksum.txt')
);
```

### 並列処理の最適化

**制御された並行処理**

```javascript
import pLimit from 'p-limit';

// 同時実行数を制限
const limit = pLimit(3);

const tasks = urls.map(url => 
    limit(() => fetchData(url))
);

const results = await Promise.all(tasks);
```

**バッチ処理パターン**

```javascript
async function processBatches(items, batchSize = 10, concurrency = 3) {
    const batches = [];
    for (let i = 0; i < items.length; i += batchSize) {
        batches.push(items.slice(i, i + batchSize));
    }
    
    const limit = pLimit(concurrency);
    const results = await Promise.all(
        batches.map(batch => 
            limit(() => processBatch(batch))
        )
    );
    
    return results.flat();
}
```

### メモリ管理とリーク対策

**ヒープサイズの最適化**

```bash
# メモリ集約的アプリケーション用
node --max-old-space-size=4096 --max-semi-space-size=64 app.js
```

**バッファプーリング戦略**

```javascript
class BufferPool {
    constructor(bufferSize = 1024, poolSize = 10) {
        this.bufferSize = bufferSize;
        this.pool = [];
        
        // バッファを事前割り当て
        for (let i = 0; i < poolSize; i++) {
            this.pool.push(Buffer.allocUnsafe(bufferSize));
        }
    }
    
    getBuffer() {
        return this.pool.pop() || Buffer.allocUnsafe(this.bufferSize);
    }
    
    returnBuffer(buffer) {
        if (buffer.length === this.bufferSize) {
            buffer.fill(0); // バッファをクリア
            this.pool.push(buffer);
        }
    }
}
```

**WeakMapによるメモリリーク防止**

```javascript
const requestMetadata = new WeakMap();

function handleRequest(req) {
    requestMetadata.set(req, {
        startTime: Date.now(),
        userId: req.user.id
    });
    // reqが破棄されるとメタデータも自動的にGC対象に
}
```

## 実践的な参考リンク集

### 必読のブログ記事とWriteup

**ISUCON6の予選をNode.jsで突破する - Qiita**  
<https://qiita.com/momoiroshikibu/items/5755ba1cd39bc18102d8>  
Node.js固有の最適化テクニック（トライ木実装、クラスタプロセス最適化、ドメインソケット接続）が詳細に解説されています。各技術の性能改善度（★1-5）付きで、Node.js開発者必読の内容です。

**ISUCON13 技術ログと実装詳細**  
<https://zenn.dev/devopslead/articles/b76021fee9b377>  
pt-query-digestを使用した実際のスロークエリ分析例が含まれており、デバッグと最適化プロセスが詳細に記録されています。

### GitHubリソース

**ISUCON公式Organization**  
<https://github.com/isucon>  
全ISUCON問題とリファレンス実装が含まれています。特にISUCON13（<https://github.com/isucon/isucon13）にはNode.js実装とAnsibleプロビジョニングが含まれています。>

**matsuu/vagrant-isucon**  
<https://github.com/matsuu/vagrant-isucon>  
ISUCON 1-12の過去問題のVagrant環境。オフライン練習に必須のツールです。

### 公式ドキュメント

**ISUCON公式Blog**  
<https://isucon.net/>  
公式アナウンスと各回の問題解説。出題意図と想定解法を理解するために重要です。

**Node.js Profiling Guide**  
<https://nodejs.org/en/learn/getting-started/profiling>  
Node.js公式のパフォーマンスプロファイリングドキュメント。

**Chrome DevTools Node.js Profiling**  
<https://developer.chrome.com/docs/devtools/performance/nodejs>  
ビジュアルプロファイリングのための詳細ガイド。

### ツールとユーティリティ

**pt-query-digest** - MySQLスロークエリアナライザー  
複数の優勝チームが使用。スロークエリ閾値を0に設定して全クエリを分析します。

**netdata** - リアルタイムシステムモニタリング  
Webインターフェース（ポート19999）で直感的なパフォーマンス可視化が可能。

**alp** - アクセスログプロファイラー  
遅いエンドポイントを特定するためにISUCONで頻繁に使用されます。

## 重要なポイントまとめ

ISUCONで高スコアを獲得するための重要な戦略：

1. **プロファイリングファースト**: 0xやChrome DevToolsを使用してボトルネックを特定
2. **Node.js固有の最適化を活用**: クラスタプロセス、テンプレートキャッシング、ドメインソケット、トライ木アルゴリズム
3. **メモリ効率を重視**: クラウド環境ではCPUよりメモリ効率の最適化が重要
4. **制御された並行処理**: 無制限のPromise.all()は避け、p-limitなどで同時実行数を制御
5. **ストリーミング処理**: 大容量データは必ずストリームAPIを使用
6. **リアルタイムモニタリング**: pt-query-digestとnetdataで継続的な性能監視

これらの最適化により、典型的なISUCONシナリオで2-5倍の性能向上が期待でき、特定の最適化では10倍以上の改善も可能です。
