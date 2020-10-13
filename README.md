# sidekiqについてまとめる

参考URL
https://github.com/mperham/sidekiq/wiki/API

上記をもとにsidekiqのコマンドについてまとめる

## api

sidekiqにはworker, queue, jobに関するリアルタイム情報へのアクセスを可能にするpublic apiが存在する．

```
require 'sidekiq/api'
```

## Queue

全てのqueueを取得するリクエスト

```
Sidekiq::Queue.all
```

queueの取得

```
Sidekiq::Queue.new              # <=== "default" というqueueを取得
Sidekiq::Queue.new("mailer")    # <=== "mailer" というqueueを取得
```

queueに含まれているjobの数を取得

```
Sidekiq::Queue.new.size # ==> 4
```

queueに含まれる全てのjobを削除する

```
Sidekiq::Queue.new.clear
```

jobのidを指定して削除する

```
queue = Sidekiq::Queue.new("mailer")
queue.each do |job|
    job.klass   # => 'MyWorker'
    job.args    # => [1,2,3]
    job.delete if job.jid == 'abcdef1234567890'
end
```

job idを表示する

```
Sidekiq::Queue.new.each{ |job| puts job.id}
```

queueの待ち時間(今の時間 - 最後にジョブが実行された時間)
```
Sidekiq::Queue.new.latency  # => 14.5(example)
```

job idで検索する
```
Sidekiq::Queue.new.find_job(somejid)
```

## Named Queues
### Scheduled
時系列順にソートされたジョブを取得する
そこから，ジョブを検索することもできる(select, scanを使うと高速化できる)
そしてジョブの削除もできる(delete)

```
ss = Sidekiq::ScheduledSet.new
jobs = ss.scan("SomeWorker").select { |retri| retri.klass == 'SomeWorker'}
jobs.each(&:delete)
```
### Retries

ジョブがraiseした時に，SidekiqはRetrySetにジョブを配置して自動的に再試行する

```
rs = Sidekiq::RetrySet.new
rs.size
rs.clear
```

Sidekiq内で再試行の列挙を許可する
これに基づいて検索/フィルタリングができる．
下記はそれを用いたジョブの削除

```
query = Sidekiq::RetrySet.new
query.select do |job|
    job.klass = 'Sidekiq::Extensions::DelayedClass' &&
        ((klass, method, args) = YAML.load(job.args[0])) &&
        klass == User &&
        method == :setup_new_subscriber
end.map(&:delete)
```

### Dead

DeadSetはSidekiqによって死んだとみなされた全てのジョブを死んだときの順序で保持する

```
ds = Sidekiq::DeadSet.new
ds.size
ds.clear
```

DeadSetに含まれる特定のジョブを再試行することもできる

```
ds.select do |job|
    job.klass == 'FixedWorker' && job.args[0] === 123
end.map(&:retry)
```

### Scan

glob：*,?,[]などのワイルドカードを使ったファイルパス(globパターン)から，それにマッチする実際のファイルパスを展開すること

```
ss = Sidekiq::ScheduledSet.new
ss.scan("\"class\":\"HardWorker\""){ |job| ... }
ss.scan("HardWorker") { |job| ... }
```

ブロックを渡さない場合にはscanが列挙子を返す

### Processes

実行中のSidekiqプロセスの現在の情報にアクセスできる(5秒更新)
プロセスをリモートで制御することもできる

```
ps = Sidekiq::ProcessSet.new
ps.size
ps.each do |process|
    p process['busy']
    p process['hostname']
    p process['pid']
end
ps.each(&:quiet!)
ps.each(&:stop!)
```

### Workers

```
workers = Sidekiq::Workers.new
workers.size
workers.each do |process_id, thread_id, work|

end
```

### Stats

```
stats = Sidekiq::Stats.new
stats.processed
stats.failed
stats.queues
stats.enqueued
```

### Stats History

Sidekiqのプロセスの失敗と処理済みの履歴を見ることができる

```
s = Sidekiq::Stats::History.new(3, Date.parse("2012-12-3"))
s.failed
s.processed
```
