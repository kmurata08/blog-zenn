---
title: "キャッシュドライバの移行を容易にする抽象化レイヤーの実装事例"
emoji: "🚀"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [PHP,CodeIgniter]
published: false
---

# はじめに
最近、PHPで管理されているサーバーのキャッシュドライバをmemcachedからRedisに切り替えるということを行なっています。
この移行では、既存のプロダクトへの影響を最小限に抑えながら、安全にキャッシュシステムを移行する必要がありました。
本記事では、キャッシュドライバの移行における課題と、それを解決するためのアプローチについて、実際の経験に基づいて共有します。

# 移行における要件
1. 既存のキャッシュデータの損失を防ぐ
2. ユーザー体験を損なわない（キャッシュが突然取得できなくなる状況を防ぐ）
3. アプリケーションコードの変更を最小限に抑える

# 実際に直面した課題
## 現状のキャッシュ実装
アプリケーション側で以下のような、フレームワークを使ったキャッシュの取得関数が多くの箇所にありました。

```
// キャッシュの取得
// キャッシュドライバの使用はフレームワーク側で抽象化されており、実際には内部的にはmemcachedに対してアクセスが走るような状態
$cache_value = $this->cache->get($cache_key);

// キャッシュの値が存在すれば返す
```

なお、今回はフレームワークにCodeIgniterを使用しており、 `$this->cache` で自動的に使用されるキャッシュドライバは以下のように指定できるイメージです。
```
// memcachedを使う場合
$this->load->driver('cache', [
    'adapter' => 'memcached', 
    'backup' => 'file'
]);

// Redisを使う場合
$this->load->driver('cache', [
    'adapter' => 'redis', 
    'backup' => 'file'
]);
```

これにより、 `$this->cache` で使われるキャッシュドライバが何なのか意識せずともアプリケーション側でキャッシュを使えるような状態です。
つまり、 `$this->cache` はすでにフレームワーク側で一段階抽象されているような状態です。

一方、 `$this->cache->redis->get(xxx)` のように、キャッシュドライバを指定してアプリケーションから呼び出すこともできます。

## キャッシュの用途による課題
今回遭遇したキャッシュの使い道としてはざっくり以下の2点があるような状態でした。

- 一部のデータへの2回目以降のアクセスを高速化する
- 何らかのデータの一時的な保持

キャッシュの書き込み先をRedisに移すにあたり、前者に関しては一時的に多少のパフォーマンスが落ちるくらいなのでドライバを一気にRedisに変える形でも問題ないと考えました。
一方、後者に関してはキャッシュの永続化、読み込み先を一気にRedisに切り替えると、ユーザーが操作を行う中で「あるはずのデータがない」という状況が起きてしまうため、一気にRedisに切り替えることが難しいと考えました。

# 解決するアプローチ
そこでとったアプローチが、「書き込みはRedis、、読み込みはRedisとmemcachedの両方から行い、値がある方を使う」というアプローチでした。
このアプローチにより、以下のメリットが得られます。

- 新しいデータは全てRedisに保存されるため、徐々に移行が進む
- 既存のmemcachedのデータも読み込めるため、ユーザー体験が維持される
- キャッシュの二重読み込みによる若干のパフォーマンス低下はあるが、移行期間限定の一時的なものとなる

キャッシュの値を取得するコードとしては以下のようになります。

```php
$data_redis = $this->cache->redis->get($key, $data, $ttl);
if ($data_redis) {
    // redisにキャッシュがある場合の処理
}

// デフォルトで使用されるキャッシュドライバにはmemcachedが指定されている
$data_memcached = $this->cache->get($key, $data, $ttl);
if ($data_memcached) {
    // memcachedにキャッシュがある場合の処理
}

// 続きの処理
```

書き込みの処理は以下のような感じになります

```PHP
// 元は $this->cache->save
$this->cache->redis->save($key, $data, $ttl);

// 続きの処理
```

全ての箇所に対して対応を完了させるために、これらをアプリケーションのあらゆる箇所で書き換えていくようなイメージでした。
元々フレームワーク側でキャッシュドライバを意識せず使えるように抽象化されているとはいえ、キャッシュを安全に載せ替える場合にはあらゆる箇所に手を加える必要があった状態です。

# キャッシュ操作の抽象化による解決
この解決策としてとったのが、「フレームワーク側のキャッシュ操作をさらに隠蔽するアプリケーション固有のレイヤーを設ける」ということです。
今回は関数で実装しました。実装は以下のイメージです。

```PHP
/**
 * キャッシュの値を取得する
 * キャッシュドライバのgetをラップしており、この中でキャッシュドライバを切り替えればアプリケーション側の変更が不要
 */
public function get_cache_value(string $key): mixed
{
    // Redis または Memcached からキャッシュを取得する
    $data_redis = $this->cache->redis->get($key);
    if ($data_redis) {
        return $data_redis;
    }

    $data_memcached = $this->cache->get($key);
    if ($data_memcached) {
        return $data_memcached;
    }

    return false;
}

/**
 * キャッシュの値を保存する
 * キャッシュドライバのsaveをラップしており、この中でキャッシュドライバを切り替えればアプリケーション側の変更が不要
 */
public function save_cache_value(string $key, mixed $data, int $ttl): void
{
    $this->cache->redis->save($key, $data, $ttl);
}
```

アプリケーション側でキャッシュを取得するあらゆる箇所では以下のような形にしておきます。

```PHP
$cache_value = $this->get_cache_value($key);
```

このようにすることで、アプリケーション側は「キャッシュをどのような方法で取得するか」ということを意識せず、キャッシュを取得できます。
また、しばらく経ってmemcachedのキャッシュも全て期限が切れ、Redisにのみ書き込みするよう変更する場合も、 `get_cache_value` 関数の中だけ変更すればよくなります。

```php
/**
 * キャッシュの値を取得する
 * キャッシュドライバのgetをラップしており、この中でキャッシュドライバを切り替えればアプリケーション側の変更が不要
 */
public function get_cache_value(string $key): mixed
{
    // Redis からキャッシュを取得する
    $data_redis = $this->cache->redis->get($key);
    if ($data_redis) {
        return $data_redis;
    }

    return false;
}
```

# 発展的な実装案
より良い実装を目指す場合、以下のようなアプローチも考えられます。
このようにキャッシュを扱うインターフェースを定義して、その中でキャッシュを扱う具体的な処理を書いておくことで、CacheServiceInterfaceに依存するクラスではテストの際にモックを作りやすいといったメリットもあるかと思います。

```php
interface CacheServiceInterface
{
    public function get(string $key): mixed;
    public function save(string $key, mixed $data, int $ttl): void;
}

class HybridCacheService implements CacheServiceInterface
{
    private $redis;
    private $memcached;

    public function __construct($redis, $memcached)
    {
        $this->redis = $redis;
        $this->memcached = $memcached;
    }

    public function get(string $key): mixed
    {
        $data = $this->redis->get($key);
        if ($data !== false) {
            return $data;
        }
        return $this->memcached->get($key);
    }

    public function save(string $key, mixed $data, int $ttl): void
    {
        $this->redis->save($key, $data, $ttl);
    }
}
```

# まとめ
キャッシュドライバの移行という課題に対して、適切な抽象化レイヤーを設けることで、安全かつ効率的な移行を実現することができました。
この実装により、実際に以下のような効果が得られました：

- キャッシュドライバの移行作業が大幅に簡素化
- アプリケーションコードの変更量を最小限に抑制
- ユーザーへの影響をほとんど出すことなく移行を完了

また、フレームワークが提供する抽象化に加えて、アプリケーション固有の抽象化レイヤーを設けることで、より柔軟で保守性の高いシステムを構築することができるかと思います。