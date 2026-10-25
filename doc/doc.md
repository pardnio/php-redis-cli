# php-redis-cli - Documentation

Last updated: 2026-10-25

> Back to [README](../README.md)

## Prerequisites

- PHP 8.0 or higher
- The PHP `redis` extension (`ext-redis`)
- Composer
- A reachable Redis server

## Installation

### Using Composer

```bash
composer require pardnchiu/redis-cli
```

### From Source

```bash
git clone https://github.com/pardnio/php-redis-cli.git
cd php-redis-cli
composer install
```

The package maps `pardnchiu\` to `src/` via PSR-4.

## Configuration

### Environment Variables

| Variable | Required | Default | Description |
|----------|----------|---------|-------------|
| `REDIS_HOST` | No | `localhost` | Redis host |
| `REDIS_PORT` | No | `6379` | Redis port |
| `REDIS_PASSWORD` | No | `""` | Runs `AUTH` when non-empty |

```env
REDIS_HOST=localhost
REDIS_PORT=6379
REDIS_PASSWORD=your_password
```

Fixed behavior: `pconnect` with a 2.5-second timeout, `SELECT 0` after connecting, and `close()` on destruct.

## Usage

### Basic

```php
<?php

require __DIR__ . '/vendor/autoload.php';

use pardnchiu\RDB;

$redis = new RDB();

$redis->set(0, "user:123", "Pardn", 3600);
$name = $redis->get(0, "user:123");

$redis->incr(0, "page_views");
```

### Advanced: Graceful Degradation

Methods never throw on connection failure; use `isConnected()` and the returned defaults to decide when to fall back to another source.

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

### Hashes, Lists, and Sets

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

## API Reference

The first argument `$db` of every method is the database index, selected with `SELECT $db` before the command runs. Without a connection, each method returns the value in the Default column.

### Connection

| Method | Signature | Default | Description |
|--------|-----------|---------|-------------|
| Constructor | `__construct()` | - | Opens a persistent connection from env vars; failures only go to `error_log` |
| `isConnected` | `isConnected(): bool` | - | Whether a connection is held |
| `info` | `info(): ?array` | `null` | `INFO` |

### Strings and Counters

| Method | Signature | Default |
|--------|-----------|---------|
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

### Hashes

| Method | Signature | Default |
|--------|-----------|---------|
| `hget` | `hget($db, $key, $field)` | `null` |
| `hset` | `hset($db, $key, $field, $value, $expire = null)` | `false` |
| `hgetall` | `hgetall($db, $key)` | `[]` |

### Lists

| Method | Signature | Default |
|--------|-----------|---------|
| `lpush` | `lpush($db, $key, $value, $expire = null)` | `null` |
| `rpush` | `rpush($db, $key, $value, $expire = null)` | `null` |
| `lpop` | `lpop($db, $key)` | `null` |
| `rpop` | `rpop($db, $key)` | `null` |
| `llen` | `llen($db, $key)` | `0` |

### Sets

| Method | Signature | Default |
|--------|-----------|---------|
| `sadd` | `sadd($db, $key, $member, $expire = null)` | `null` |
| `srem` | `srem($db, $key, $member)` | `null` |
| `smembers` | `smembers($db, $key)` | `[]` |
| `scard` | `scard($db, $key)` | `0` |
| `sismember` | `sismember($db, $key, $member)` | `false` |
| `sinter` | `sinter($db, $keys)` | `[]` |
| `sunion` | `sunion($db, $keys)` | `[]` |
| `sdiff` | `sdiff($db, $keys)` | `[]` |

Write methods that take `$expire` run `EXPIRE $expire` on the whole key after writing.

***

©️ 2024 [邱敬幃 Pardn Chiu](https://www.linkedin.com/in/pardnchiu)
