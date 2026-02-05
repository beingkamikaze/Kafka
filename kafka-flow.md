```mermaid
sequenceDiagram
    autonumber

    %% ===== PRODUCER =====
    participant P as Producer
    participant K as Kafka Cluster
    participant B1 as Broker 1 (Leader)
    participant B2 as Broker 2 (Follower)
    participant B3 as Broker 3 (Follower)
    participant Z as Controller / Metadata
    participant C1 as Consumer 1
    participant C2 as Consumer 2

    %% ===== METADATA FETCH =====
    P->>K: Fetch Metadata (topics, partitions, leaders)
    K->>P: Partition → Leader mapping

    %% ===== PRODUCER SEND =====
    P->>P: Serialize message (key, value)
    P->>P: Choose partition\n(hash(key) % partitions)
    P->>B1: Send message to partition leader

    %% ===== WRITE TO LOG =====
    B1->>B1: Append message to log segment
    B1->>B2: Replicate message
    B1->>B3: Replicate message

    %% ===== ACK HANDLING =====
    alt acks = 0
        B1-->>P: No ACK (fire & forget)
    else acks = 1
        B1-->>P: ACK after leader write
    else acks = all
        B2-->>B1: Replica ACK
        B3-->>B1: Replica ACK
        B1-->>P: ACK after ISR sync
    end

    %% ===== OFFSET ASSIGNMENT =====
    B1->>B1: Assign offset (monotonic per partition)

    %% ===== CONSUMER GROUP JOIN =====
    C1->>K: Join consumer group
    C2->>K: Join consumer group
    K->>K: Trigger rebalance
    K->>C1: Assign partitions (P0, P1)
    K->>C2: Assign partitions (P2)

    %% ===== CONSUME DATA =====
    C1->>B1: Poll messages from assigned partitions
    B1-->>C1: Batch of records with offsets

    %% ===== PROCESSING =====
    C1->>C1: Process message (business logic)

    %% ===== OFFSET COMMIT =====
    alt Auto Commit
        C1->>K: Commit offset periodically
    else Manual Commit
        C1->>K: Commit offset after processing
    end

    %% ===== FAILURE SCENARIOS =====
    opt Consumer Crash
        C1--x C1: Consumer dies
        K->>K: Detect heartbeat failure
        K->>K: Trigger rebalance
        K->>C2: Assign C1 partitions to C2
    end

    opt Broker Failure
        B1--x B1: Leader broker dies
        Z->>Z: Elect new leader from ISR
        Z->>B2: Promote to leader
    end
