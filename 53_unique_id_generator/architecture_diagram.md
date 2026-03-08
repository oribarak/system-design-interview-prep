# Unique ID Generator (Snowflake) — Architecture Diagrams

## 1. High-Level Architecture

```mermaid
flowchart TB
    subgraph Applications[Application Services]
        SVC_A[Service A<br/>Embedded Generator]
        SVC_B[Service B<br/>Embedded Generator]
        SVC_C[Service C<br/>Embedded Generator]
    end

    subgraph Coordination[Worker ID Coordination]
        ZK[ZooKeeper Cluster<br/>3 or 5 nodes]
        ZK_NODE_A[/Ephemeral Node<br/>dc:5 worker:1/]
        ZK_NODE_B[/Ephemeral Node<br/>dc:5 worker:2/]
        ZK_NODE_C[/Ephemeral Node<br/>dc:5 worker:3/]
    end

    subgraph Generator[Snowflake Generator - In Process]
        CLOCK[System Clock<br/>currentTimeMillis]
        BIT_LAYOUT[Bit Layout Engine<br/>41 + 5 + 5 + 12 bits]
        SEQ[Sequence Counter<br/>0 - 4095 per ms]
        SKEW[Clock Skew Handler<br/>Wait / Logical Clock / Halt]
    end

    subgraph ID_Consumer[ID Consumers]
        DB[(Database<br/>Primary Keys)]
        EVENTS[Event Stream<br/>Kafka Message Keys]
        TRACE[Distributed Tracing<br/>Trace IDs]
    end

    subgraph Utilities
        PARSER[ID Parser Service<br/>Decompose ID to parts]
        MONITOR[Monitoring<br/>Generation rate, skew events]
    end

    SVC_A -->|startup: register| ZK
    SVC_B -->|startup: register| ZK
    SVC_C -->|startup: register| ZK
    ZK --> ZK_NODE_A
    ZK --> ZK_NODE_B
    ZK --> ZK_NODE_C
    ZK_NODE_A -->|worker_id=1| SVC_A
    ZK_NODE_B -->|worker_id=2| SVC_B
    ZK_NODE_C -->|worker_id=3| SVC_C

    CLOCK --> BIT_LAYOUT
    SEQ --> BIT_LAYOUT
    SKEW --> CLOCK

    SVC_A -->|generateId()| BIT_LAYOUT
    SVC_B -->|generateId()| BIT_LAYOUT
    SVC_C -->|generateId()| BIT_LAYOUT

    BIT_LAYOUT -->|64-bit ID| DB
    BIT_LAYOUT -->|64-bit ID| EVENTS
    BIT_LAYOUT -->|64-bit ID| TRACE

    BIT_LAYOUT --> PARSER
    BIT_LAYOUT --> MONITOR
```

## 2. Deep-Dive: Snowflake ID Bit Layout and Generation

```mermaid
flowchart TB
    subgraph Input[Input Values]
        TIMESTAMP[Current Timestamp<br/>currentTimeMillis - EPOCH<br/>41 bits: ms since 2020-01-01]
        DC_ID[Datacenter ID<br/>5 bits: 0-31<br/>Configured at deploy]
        WORKER_ID[Worker ID<br/>5 bits: 0-31<br/>Assigned by ZooKeeper]
        SEQUENCE[Sequence Number<br/>12 bits: 0-4095<br/>Reset each ms]
    end

    subgraph Clock_Logic[Clock and Sequence Logic]
        GET_TIME[Get Current Timestamp]
        COMPARE{timestamp vs<br/>lastTimestamp?}
        SAME_MS[Same Millisecond<br/>Increment sequence]
        NEW_MS[New Millisecond<br/>Reset sequence to 0]
        BACKWARD[Clock Backward!]
        SEQ_CHECK{Sequence<br/>overflow?}
        WAIT[Wait for Next ms<br/>Busy loop]
        SMALL_JUMP{Jump<br/>< 5ms?}
        SLEEP[Sleep until<br/>clock catches up]
        ERROR[Throw Exception<br/>Clock moved backward]
    end

    subgraph Bit_Assembly[Bit Assembly]
        SHIFT_TS[timestamp LEFT SHIFT 22]
        SHIFT_DC[datacenter LEFT SHIFT 17]
        SHIFT_WK[worker LEFT SHIFT 12]
        OR_OP[Bitwise OR all parts]
        RESULT[64-bit Snowflake ID]
    end

    subgraph Example[Example ID Decomposition]
        EXAMPLE_ID["ID: 6820884539871875073"]
        BIT_VIEW["Binary: 0 | 10111101... | 00101 | 01010 | 000000000001"]
        DECODED["Timestamp: 2024-03-07T16:00:00.000Z<br/>Datacenter: 5<br/>Worker: 10<br/>Sequence: 1"]
    end

    GET_TIME --> COMPARE
    COMPARE -->|equal| SAME_MS
    COMPARE -->|greater| NEW_MS
    COMPARE -->|less| BACKWARD
    SAME_MS --> SEQ_CHECK
    SEQ_CHECK -->|< 4095| TIMESTAMP
    SEQ_CHECK -->|= 4095| WAIT
    WAIT --> GET_TIME
    BACKWARD --> SMALL_JUMP
    SMALL_JUMP -->|yes| SLEEP
    SMALL_JUMP -->|no| ERROR
    SLEEP --> GET_TIME
    NEW_MS --> TIMESTAMP

    TIMESTAMP --> SHIFT_TS
    DC_ID --> SHIFT_DC
    WORKER_ID --> SHIFT_WK
    SEQUENCE --> OR_OP
    SHIFT_TS --> OR_OP
    SHIFT_DC --> OR_OP
    SHIFT_WK --> OR_OP
    OR_OP --> RESULT

    RESULT --> EXAMPLE_ID
    EXAMPLE_ID --> BIT_VIEW
    BIT_VIEW --> DECODED
```

## 3. Critical Path Sequence: ID Generation with Clock Skew Handling

```mermaid
sequenceDiagram
    participant App as Application
    participant Gen as Snowflake Generator
    participant Clock as System Clock
    participant Seq as Sequence Counter
    participant Skew as Clock Skew Handler
    participant ZK as ZooKeeper

    Note over App,ZK: Startup Phase (once)
    App->>ZK: Register worker (create ephemeral node)
    ZK-->>App: Assigned worker_id=10, datacenter_id=5

    Note over App,ZK: Normal ID Generation (< 1 microsecond)
    App->>Gen: generateId()
    Gen->>Clock: currentTimeMillis()
    Clock-->>Gen: timestamp = 1709827200042

    alt Same millisecond as last call
        Gen->>Seq: increment()
        Seq-->>Gen: sequence = 7
    else New millisecond
        Gen->>Seq: reset to 0
        Seq-->>Gen: sequence = 0
    end

    Gen->>Gen: id = (ts-epoch)<<22 | dc<<17 | wk<<12 | seq
    Gen-->>App: id = 6820884539871875073

    Note over App,ZK: Clock Backward Jump Scenario
    App->>Gen: generateId()
    Gen->>Clock: currentTimeMillis()
    Clock-->>Gen: timestamp = 1709827200038 (backward!)

    Gen->>Skew: Clock went backward by 4ms
    Skew->>Skew: 4ms < 5ms threshold: wait
    Skew->>Clock: sleep(4ms)
    Clock-->>Skew: timestamp = 1709827200043
    Skew-->>Gen: Recovered, proceed

    Gen->>Seq: reset to 0 (new millisecond)
    Gen->>Gen: Assemble ID with new timestamp
    Gen-->>App: id = 6820884539871879168

    Note over App,ZK: Sequence Exhaustion Scenario
    App->>Gen: generateId() [burst: 4096th call in same ms]
    Gen->>Seq: increment()
    Seq-->>Gen: sequence = 4095

    App->>Gen: generateId() [4097th call in same ms]
    Gen->>Seq: increment()
    Seq-->>Gen: sequence = 0 (wrapped!)
    Gen->>Gen: Sequence exhausted, must wait

    loop Wait for next millisecond
        Gen->>Clock: currentTimeMillis()
        Clock-->>Gen: still same millisecond
    end
    Gen->>Clock: currentTimeMillis()
    Clock-->>Gen: next millisecond!
    Gen->>Seq: reset to 0
    Gen->>Gen: Assemble ID
    Gen-->>App: id (in next millisecond)
```
