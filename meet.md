## WebRTC Architecture with MediaSoup and GStreamer

### Overall System Architecture

```mermaid
graph TB
    subgraph "Client Layer"
        C1[React Client 1]
        C2[React Client 2]
        C3[React Client N]
    end

    subgraph "Edge Layer"
        LB[Load Balancer<br/>HAProxy/Nginx]
        WS[WebSocket Gateway<br/>FastAPI]
    end

    subgraph "Media Processing Layer"
        MS1[MediaSoup Worker 1]
        MS2[MediaSoup Worker 2]
        MSN[MediaSoup Worker N]
        GS[GStreamer Pipeline<br/>Recording/Processing]
    end

    subgraph "Control Layer"
        API[FastAPI REST API]
        SIG[Signaling Service]
        ROOM[Room Manager]
        MED[Media Controller]
    end

    subgraph "Data Layer"
        REDIS[(Redis Cluster<br/>PubSub/State)]
        PG[(PostgreSQL<br/>Persistent Data)]
        S3[(S3/MinIO<br/>Media Storage)]
    end

    C1 & C2 & C3 -.->|HTTPS/WSS| LB
    LB -->|Route| WS
    LB -->|Route| API
    
    WS <-->|Control| SIG
    SIG <--> ROOM
    ROOM <--> MED
    
    MED <-->|IPC/Pipe| MS1 & MS2 & MSN
    MS1 & MS2 & MSN <-->|RTP/RTCP| C1 & C2 & C3
    
    MS1 --> GS
    GS --> S3
    
    SIG & ROOM & MED <--> REDIS
    API <--> PG
```

### MediaSoup Architecture Detail

```mermaid
graph TB
    subgraph "MediaSoup Cluster"
        subgraph "MediaSoup Router 1"
            R1[Router]
            PT1[Producer Transport]
            CT1[Consumer Transport]
            P1[Audio Producer]
            C1[Audio Consumer]
        end
        
        subgraph "MediaSoup Router 2"
            R2[Router]
            PT2[Producer Transport]
            CT2[Consumer Transport]
            P2[Audio Producer]
            C2[Audio Consumer]
        end
        
        subgraph "Pipe Transports"
            PIPE[Pipe Transport<br/>Inter-Router]
        end
    end
    
    CLIENT1[Client 1] -->|Produce| PT1
    PT1 --> P1
    P1 --> R1
    R1 --> C2
    C2 --> CT2
    CT2 -->|Consume| CLIENT2[Client 2]
    
    R1 <-->|Pipe| PIPE
    PIPE <-->|Pipe| R2
    
    R1 -->|PlainTransport| GST[GStreamer<br/>Recording]
```

### Signaling Flow Sequence

```mermaid
sequenceDiagram
    participant C as Client
    participant WS as WebSocket Gateway
    participant RM as Room Manager
    participant MC as Media Controller
    participant MS as MediaSoup
    participant Redis as Redis PubSub

    C->>WS: Connect WebSocket
    WS->>RM: Join Room Request
    RM->>Redis: Check Room State
    Redis-->>RM: Room Info
    RM->>MC: Create Transport
    MC->>MS: router.createWebRtcTransport()
    MS-->>MC: Transport Details
    MC-->>RM: Transport Created
    RM-->>WS: Transport Parameters
    WS-->>C: Transport Config
    
    C->>WS: Connect Transport
    WS->>MC: Connect Transport
    MC->>MS: transport.connect()
    MS-->>MC: Connected
    MC->>Redis: Publish: User Connected
    Redis-->>RM: Broadcast to Room
    RM-->>WS: Notify Other Clients
    
    C->>WS: Produce Audio
    WS->>MC: Create Producer
    MC->>MS: transport.produce()
    MS-->>MC: Producer Created
    MC->>Redis: Publish: New Producer
    Redis-->>RM: Broadcast Producer
    RM-->>WS: Notify Consumers
```

### GStreamer Integration Architecture

```mermaid
graph LR
    subgraph "MediaSoup Side"
        MS[MediaSoup Router]
        PT[PlainTransport]
        RTP[RTP Stream]
    end
    
    subgraph "GStreamer Pipeline"
        RTPBIN[rtpbin]
        DEPAY[rtpdepay]
        DECODE[opusdec]
        
        subgraph "Processing"
            CONVERT[audioconvert]
            RESAMPLE[audioresample]
            LEVEL[level]
            VOLUME[volume]
        end
        
        subgraph "Output Branches"
            TEE[tee]
            
            subgraph "Recording Branch"
                QUEUE1[queue]
                ENCODE1[opusenc]
                MUX1[webmmux]
                SINK1[filesink]
            end
            
            subgraph "Streaming Branch"
                QUEUE2[queue]
                ENCODE2[opusenc]
                PAY[rtpopuspay]
                SINK2[udpsink]
            end
            
            subgraph "Analysis Branch"
                QUEUE3[queue]
                ANALYZE[audioanalyze]
                FAKESINK[fakesink]
            end
        end
    end
    
    MS -->|RTP| PT
    PT -->|UDP| RTP
    RTP --> RTPBIN
    RTPBIN --> DEPAY
    DEPAY --> DECODE
    DECODE --> CONVERT
    CONVERT --> RESAMPLE
    RESAMPLE --> LEVEL
    LEVEL --> VOLUME
    VOLUME --> TEE
    
    TEE --> QUEUE1
    QUEUE1 --> ENCODE1
    ENCODE1 --> MUX1
    MUX1 --> SINK1
    
    TEE --> QUEUE2
    QUEUE2 --> ENCODE2
    ENCODE2 --> PAY
    PAY --> SINK2
    
    TEE --> QUEUE3
    QUEUE3 --> ANALYZE
    ANALYZE --> FAKESINK
```

### Component Interaction Architecture

```mermaid
graph TB
    subgraph "Python Application Layer"
        FAST[FastAPI App]
        WSM[WebSocket Manager]
        RMG[Room Manager]
        MCT[Media Controller]
        GSC[GStreamer Controller]
        
        subgraph "MediaSoup Python Client"
            MPC[MediaSoup Client]
            IPC[IPC/Pipe Transport]
        end
    end
    
    subgraph "Native Layer"
        subgraph "MediaSoup C++ Workers"
            W1[Worker 1<br/>CPU Core 1]
            W2[Worker 2<br/>CPU Core 2]
            WN[Worker N<br/>CPU Core N]
        end
        
        subgraph "GStreamer Processes"
            GP1[GStreamer Pipeline 1]
            GP2[GStreamer Pipeline 2]
            GPN[GStreamer Pipeline N]
        end
    end
    
    FAST --> WSM
    WSM <--> RMG
    RMG <--> MCT
    MCT <--> MPC
    MPC <--> IPC
    
    IPC <--> W1 & W2 & WN
    MCT --> GSC
    GSC -->|Launch/Control| GP1 & GP2 & GPN
    
    W1 -.->|RTP| GP1
    W2 -.->|RTP| GP2
    WN -.->|RTP| GPN
```

### Scaling and High Availability Architecture

```mermaid
graph TB
    subgraph "Region 1 - US West"
        subgraph "Availability Zone 1"
            LB1[Load Balancer]
            API1[API Servers]
            MS1[MediaSoup Cluster]
            GS1[GStreamer Farm]
        end
        
        subgraph "Availability Zone 2"
            LB2[Load Balancer]
            API2[API Servers]
            MS2[MediaSoup Cluster]
            GS2[GStreamer Farm]
        end
        
        REDIS1[(Redis Cluster)]
        PG1[(PostgreSQL Primary)]
    end
    
    subgraph "Region 2 - EU Central"
        subgraph "Availability Zone 3"
            LB3[Load Balancer]
            API3[API Servers]
            MS3[MediaSoup Cluster]
            GS3[GStreamer Farm]
        end
        
        REDIS2[(Redis Cluster)]
        PG2[(PostgreSQL Replica)]
    end
    
    subgraph "Global Services"
        CDN[CDN<br/>Static Assets]
        S3[S3 Compatible<br/>Object Storage]
        TURN[Global TURN<br/>Infrastructure]
    end
    
    LB1 & LB2 <--> API1 & API2
    API1 & API2 <--> MS1 & MS2
    MS1 & MS2 --> GS1 & GS2
    
    REDIS1 <-.->|Replication| REDIS2
    PG1 -->|Streaming Replication| PG2
    
    GS1 & GS2 & GS3 --> S3
    
    MS1 & MS2 & MS3 <-.->|Cross-Region<br/>Pipe| MS3
```

### Resource Allocation Strategy

```mermaid
graph LR
    subgraph "Server Node (32 CPU, 128GB RAM)"
        subgraph "CPU Allocation"
            CPU1[CPU 0-1<br/>System/OS]
            CPU2[CPU 2-5<br/>FastAPI/Python]
            CPU3[CPU 6-21<br/>MediaSoup Workers]
            CPU4[CPU 22-31<br/>GStreamer Pipelines]
        end
        
        subgraph "Memory Allocation"
            MEM1[8GB<br/>OS/System]
            MEM2[16GB<br/>Python Apps]
            MEM3[64GB<br/>MediaSoup]
            MEM4[40GB<br/>GStreamer]
        end
        
        subgraph "Network Allocation"
            NET1[eth0<br/>Management]
            NET2[eth1<br/>Media/RTP<br/>10Gbps]
            NET3[eth2<br/>Storage<br/>10Gbps]
        end
    end
```

### Media Flow State Machine

```mermaid
stateDiagram-v2
    [*] --> Disconnected
    Disconnected --> Connecting: User Joins
    Connecting --> Connected: Transport Ready
    Connected --> Producing: Start Audio
    Connected --> Consuming: Others Producing
    
    Producing --> Recording: Enable Recording
    Recording --> Producing: Stop Recording
    
    Producing --> ProcessingAudio: Apply Effects
    ProcessingAudio --> Producing: Effects Applied
    
    Producing --> Failed: Error
    Consuming --> Failed: Error
    Failed --> Reconnecting: Retry
    Reconnecting --> Connected: Success
    Reconnecting --> Disconnected: Max Retries
    
    Connected --> Disconnecting: User Leaves
    Producing --> Disconnecting: User Leaves
    Consuming --> Disconnecting: User Leaves
    Disconnecting --> Disconnected: Cleanup Complete
    Disconnected --> [*]
```

### Database Schema Design

```mermaid
erDiagram
    USERS ||--o{ MEETINGS : creates
    USERS ||--o{ PARTICIPANTS : joins
    MEETINGS ||--o{ PARTICIPANTS : has
    MEETINGS ||--o{ RECORDINGS : contains
    PARTICIPANTS ||--o{ MEDIA_TRACKS : produces
    PARTICIPANTS ||--o{ MEDIA_TRACKS : consumes
    
    USERS {
        uuid id PK
        string email
        string name
        timestamp created_at
        jsonb preferences
    }
    
    MEETINGS {
        uuid id PK
        string meeting_code UK
        uuid creator_id FK
        timestamp started_at
        timestamp ended_at
        jsonb settings
        enum status
    }
    
    PARTICIPANTS {
        uuid id PK
        uuid meeting_id FK
        uuid user_id FK
        string display_name
        timestamp joined_at
        timestamp left_at
        jsonb stats
    }
    
    MEDIA_TRACKS {
        uuid id PK
        uuid participant_id FK
        enum kind
        string codec
        jsonb parameters
        timestamp created_at
    }
    
    RECORDINGS {
        uuid id PK
        uuid meeting_id FK
        string storage_url
        bigint size_bytes
        integer duration_seconds
        timestamp created_at
        jsonb metadata
    }
```

### Monitoring and Observability Architecture

```mermaid
graph TB
    subgraph "Application Metrics"
        APP[Python Apps] -->|Prometheus| PROM[Prometheus]
        MS[MediaSoup] -->|Stats API| COLLECTOR[Metrics Collector]
        GS[GStreamer] -->|Custom Probes| COLLECTOR
        COLLECTOR --> PROM
    end
    
    subgraph "Infrastructure Metrics"
        NODE[Node Exporter] --> PROM
        CADV[cAdvisor] --> PROM
    end
    
    subgraph "Logs"
        APP2[Python Apps] -->|Structured Logs| FLUENTD[Fluentd]
        MS2[MediaSoup] -->|stdout/stderr| FLUENTD
        GS2[GStreamer] -->|GST_DEBUG| FLUENTD
        FLUENTD --> ES[Elasticsearch]
    end
    
    subgraph "Traces"
        APP3[Python Apps] -->|OpenTelemetry| JAEGER[Jaeger]
    end
    
    subgraph "Visualization"
        PROM --> GRAFANA[Grafana]
        ES --> KIBANA[Kibana]
        JAEGER --> JAEGER_UI[Jaeger UI]
    end
    
    subgraph "Alerting"
        PROM --> ALERT[AlertManager]
        ALERT --> SLACK[Slack]
        ALERT --> PAGER[PagerDuty]
    end
```

This architecture leverages MediaSoup's excellent performance for WebRTC media routing and GStreamer's flexibility for media processing, recording, and analysis. The system is designed for horizontal scaling, high availability, and comprehensive observability.
