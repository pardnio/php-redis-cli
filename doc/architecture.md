# php-redis-cli - Architecture

Last updated: 2026-10-25

> Back to [README](../README.md)

## Overview

```mermaid
graph TB
    A[Caller] --> B[pardnchiu\RDB]
    ENV[REDIS_HOST / PORT / PASSWORD] --> C[getConnection]
    B --> C
    C --> X[exec]
    X -->|Connected| D[select db + command]
    X -->|Not connected| E[Safe default]
    D --> F[(Redis)]
    EXT[ext-redis \Redis] --> C
```

## Module: RDB

Every public method of `pardnchiu\RDB` goes through `exec()`, which ensures a connection and switches the database.

```mermaid
graph TB
    subgraph RDB
        CT[__construct] --> GC[getConnection]
        GC -->|client exists| SKIP[Skip]
        GC --> PC[pconnect host port 2.5s]
        PC -->|Fail| LOG[error_log + client = null]
        PC --> AUTH{Password set?}
        AUTH -->|Yes| AU[auth]
        AU -->|Fail| LOG
        AUTH --> S0[select 0]
        EX[exec db callback default] --> GC
        EX -->|client present| SEL[select db + callback]
        EX -->|client null| DEF[Return default]
        OPS[String / hash / list / set methods] --> EX
        TTL[Writes with expire] --> EXPIRE[expire key]
        DS[__destruct] --> CL[close]
    end
```

## Data Flow

```mermaid
sequenceDiagram
    participant Caller
    participant RDB
    participant Redis
    Caller->>RDB: set(1, key, value, 600)
    RDB->>RDB: getConnection()
    alt client is null
        RDB->>Redis: pconnect + auth + select 0
        Redis-->>RDB: success / failure
    end
    alt connected
        RDB->>Redis: SELECT 1
        RDB->>Redis: SET key value
        RDB->>Redis: EXPIRE key 600
        Redis-->>RDB: OK
        RDB-->>Caller: true
    else not connected
        RDB-->>Caller: false
    end
```

## State Machine

```mermaid
stateDiagram-v2
    [*] --> Disconnected
    Disconnected --> Connected: getConnection succeeds
    Disconnected --> Disconnected: connect / auth fails (error_log)
    Connected --> Connected: exec runs command
    Connected --> [*]: __destruct close
    Disconnected --> [*]: __destruct
```

***

©️ 2024 [邱敬幃 Pardn Chiu](https://www.linkedin.com/in/pardnchiu)
