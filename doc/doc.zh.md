# php-redis-cli - 技術文件

最後更新：2026-10-25

> 返回 [README](./README.zh.md)

## 前置需求

- PHP 8.0 以上
- PHP `redis` 擴充（`ext-redis`）
- Composer
- 可連線的 Redis 伺服器

## 安裝

### 使用 Composer

```bash
composer require pardnchiu/redis-cli
```

### 從原始碼

```bash
git clone https://github.com/pardnio/php-redis-cli.git
cd php-redis-cli
composer install
```

套件以 PSR-4 將 `pardnchiu\` 對應至 `src/`。

## 設定

### 環境變數

| 變數 | 必填 | 預設值 | 說明 |
|------|------|--------|------|
| `REDIS_HOST` | 否 | `localhost` | Redis 主機 |
| `REDIS_PORT` | 否 | `6379` | Redis 埠號 |
| `REDIS_PASSWORD` | 否 | `""` | 非空時執行 `AUTH` |

```env
REDIS_HOST=localhost
REDIS_PORT=6379
REDIS_PASSWORD=your_password
```

固定行為：`pconnect` 逾時 2.5 秒、連線後先 `SELECT 0`，物件解構時呼叫 `close()`。

## 使用方式

### 基礎

```php
<?php

require __DIR__ . '/vendor/autoload.php';

use pardnchiu\RDB;

$redis = new RDB();

$redis->set(0, "user:123", "Pardn", 3600);
$name = $redis->get(0, "user:123");

$redis->incr(0, "page_views");
```

### 進階：降級處理

方法不會因連線失敗拋出例外，以 `isConnected()` 與回傳的預設值判斷是否需要改走其他資料來源。

```php
<?php

require __DIR__ . '/vendor/autoload.php';

use pardnchiu\RDB;

function loadProfile(RDB $redis, int $id): array
{
    $key = "profile:$id";

    if ($redis->isConnected()) {
        $cached = $redis->get(1, $key);
        if ($cached !== false && $cached !== null) {
            return json_decode($cached, true);
        }
    }

    $profile = ["id" => $id, "name" => "Pardn"];

    if ($redis->set(1, $key, json_encode($profile), 600) === false) {
        error_log("[Unavailable] redis: profile cache skipped");
    }

    return $profile;
}

print_r(loadProfile(new RDB(), 123));
```

### 雜湊、清單與集合

```php
<?php

require __DIR__ . '/vendor/autoload.php';

use pardnchiu\RDB;

$redis = new RDB();

$redis->hset(0, "user:123", "email", "dev@example.com", 3600);
$user = $redis->hgetall(0, "user:123");

$redis->rpush(0, "jobs", "send-mail", 600);
$job = $redis->lpop(0, "jobs");

$redis->sadd(0, "tags:a", "php");
$redis->sadd(0, "tags:b", "php");
$common = $redis->sinter(0, ["tags:a", "tags:b"]);
```

## API 參考

所有方法第一個參數 `$db` 為資料庫編號，執行前會 `SELECT $db`。未連線時回傳「預設值」欄位的值。

### 連線

| 方法 | 簽章 | 預設值 | 說明 |
|------|------|--------|------|
| 建構子 | `__construct()` | - | 依環境變數建立持久連線，失敗僅寫 `error_log` |
| `isConnected` | `isConnected(): bool` | - | 是否持有連線 |
| `info` | `info(): ?array` | `null` | `INFO` |

### 字串與計數器

| 方法 | 簽章 | 預設值 |
|------|------|--------|
| `get` | `get($db, $key)` | `null` |
| `set` | `set($db, $key, $content, $expire = null)` | `false` |
| `delete` | `delete($db, $key)` | `false` |
| `exists` | `exists($db, $key)` | `false` |
| `ttl` | `ttl($db, $key)` | `-1` |
| `keys` | `keys($db, $pattern = "*")` | `[]` |
| `incr` | `incr($db, $key)` | `null` |
| `decr` | `decr($db, $key)` | `null` |
| `append` | `append($db, $key, $value)` | `null` |
| `mget` | `mget($db, $keys)` | `[]` |
| `mset` | `mset($db, $keyValues)` | `false` |
| `flushdb` | `flushdb($db)` | `false` |

### 雜湊

| 方法 | 簽章 | 預設值 |
|------|------|--------|
| `hget` | `hget($db, $key, $field)` | `null` |
| `hset` | `hset($db, $key, $field, $value, $expire = null)` | `false` |
| `hgetall` | `hgetall($db, $key)` | `[]` |

### 清單

| 方法 | 簽章 | 預設值 |
|------|------|--------|
| `lpush` | `lpush($db, $key, $value, $expire = null)` | `null` |
| `rpush` | `rpush($db, $key, $value, $expire = null)` | `null` |
| `lpop` | `lpop($db, $key)` | `null` |
| `rpop` | `rpop($db, $key)` | `null` |
| `llen` | `llen($db, $key)` | `0` |

### 集合

| 方法 | 簽章 | 預設值 |
|------|------|--------|
| `sadd` | `sadd($db, $key, $member, $expire = null)` | `null` |
| `srem` | `srem($db, $key, $member)` | `null` |
| `smembers` | `smembers($db, $key)` | `[]` |
| `scard` | `scard($db, $key)` | `0` |
| `sismember` | `sismember($db, $key, $member)` | `false` |
| `sinter` | `sinter($db, $keys)` | `[]` |
| `sunion` | `sunion($db, $keys)` | `[]` |
| `sdiff` | `sdiff($db, $keys)` | `[]` |

帶 `$expire` 的寫入方法在寫入後對整個鍵執行 `EXPIRE $expire`。

***

©️ 2024 [邱敬幃 Pardn Chiu](https://www.linkedin.com/in/pardnchiu)
