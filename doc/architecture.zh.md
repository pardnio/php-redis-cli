# php-redis-cli - 架構

最後更新：2026-10-25

> 返回 [README](./README.zh.md)

## 概覽

```mermaid
graph TB
    A[呼叫端] --> B[pardnchiu\RDB]
    ENV[REDIS_HOST / PORT / PASSWORD] --> C[getConnection]
    B --> C
    C --> X[exec]
    X -->|有連線| D[select db + 指令]
    X -->|無連線| E[安全預設值]
    D --> F[(Redis)]
    EXT[ext-redis \Redis] --> C
```

## Module: RDB

`pardnchiu\RDB` 所有公開方法都經過 `exec()`，由它確保連線並切換資料庫。

```mermaid
graph TB
    subgraph RDB
        CT[__construct] --> GC[getConnection]
        GC -->|已有 client| SKIP[略過]
        GC --> PC[pconnect host port 2.5s]
        PC -->|失敗| LOG[error_log + client = null]
        PC --> AUTH{有密碼?}
        AUTH -->|是| AU[auth]
        AU -->|失敗| LOG
        AUTH --> S0[select 0]
        EX[exec db callback default] --> GC
        EX -->|client 存在| SEL[select db + callback]
        EX -->|client 為 null| DEF[回傳 default]
        OPS[字串 / 雜湊 / 清單 / 集合方法] --> EX
        TTL[帶 expire 的寫入] --> EXPIRE[expire key]
        DS[__destruct] --> CL[close]
    end
```

## Data Flow

```mermaid
sequenceDiagram
    participant 呼叫端
    participant RDB
    participant Redis
    呼叫端->>RDB: set(1, key, value, 600)
    RDB->>RDB: getConnection()
    alt client 為 null
        RDB->>Redis: pconnect + auth + select 0
        Redis-->>RDB: 成功 / 失敗
    end
    alt 有連線
        RDB->>Redis: SELECT 1
        RDB->>Redis: SET key value
        RDB->>Redis: EXPIRE key 600
        Redis-->>RDB: OK
        RDB-->>呼叫端: true
    else 無連線
        RDB-->>呼叫端: false
    end
```

## State Machine

```mermaid
stateDiagram-v2
    [*] --> 未連線
    未連線 --> 已連線: getConnection 成功
    未連線 --> 未連線: 連線 / 認證失敗（error_log）
    已連線 --> 已連線: exec 執行指令
    已連線 --> [*]: __destruct close
    未連線 --> [*]: __destruct
```

***

©️ 2024 [邱敬幃 Pardn Chiu](https://www.linkedin.com/in/pardnchiu)
