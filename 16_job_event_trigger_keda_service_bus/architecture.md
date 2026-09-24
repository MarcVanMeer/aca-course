# Architecture — KEDA Service Bus Event-Triggered Job

This demo provisions a Container App **Job** that is triggered by messages on an Azure
Service Bus queue using a KEDA `azure-servicebus` scale rule.

```mermaid
flowchart TB
    subgraph Azure["Azure Subscription"]
        subgraph RG["Resource Group (rg)"]
            ACR["Azure Container Registry<br/>(Basic, admin disabled)<br/>image: job-python:1.0.0"]

            MI["User-Assigned<br/>Managed Identity<br/>(identity-aca)"]

            subgraph SB["Service Bus Namespace (Standard)"]
                Q["Queue<br/>queue-messages<br/>lock: 5m, maxDelivery: 1"]
            end

            LA["Log Analytics<br/>Workspace<br/>(PerGB2018, 30d)"]

            subgraph ACAENV["Container App Environment"]
                JOB["Container App Job<br/>(aca-job-python)<br/>Event-triggered<br/>parallelism: 1"]
                KEDA["KEDA Scaler<br/>azure-servicebus rule<br/>poll: 30s, msgCount: 1<br/>min:0 / max:1 executions"]
            end
        end
    end

    subgraph Identity["RBAC Roles (on queue)"]
        R1["Service Bus Data Receiver"]
        R2["Service Bus Data Sender"]
        R3["AcrPull"]
    end

    %% Scaling trigger
    Q -. "messages in queue<br/>(connection string secret)" .-> KEDA
    KEDA -- "scales 0→1" --> JOB

    %% Job runtime
    JOB -- "pull image<br/>(via MI)" --> ACR
    JOB -- "receive & complete<br/>messages (DefaultAzureCredential)" --> Q
    JOB -- "uses" --> MI
    JOB -- "logs / metrics" --> LA
    ACAENV --> LA

    %% Identity bindings
    MI --- R1
    MI --- R2
    MI --- R3
    R1 -. grants .-> Q
    R2 -. grants .-> Q
    R3 -. grants .-> ACR

    classDef compute fill:#0072C6,stroke:#fff,color:#fff
    classDef data fill:#7FBA00,stroke:#fff,color:#fff
    classDef identity fill:#F25022,stroke:#fff,color:#fff
    classDef monitor fill:#737373,stroke:#fff,color:#fff

    class JOB,KEDA,ACAENV compute
    class SB,Q,ACR data
    class MI,R1,R2,R3 identity
    class LA monitor
```

## How it works

1. **Trigger** — Messages land in the Service Bus `queue-messages`. The Container App
   Job's KEDA `azure-servicebus` scale rule polls the queue every 30s using a
   connection-string secret.
2. **Scale to job** — When `messageCount >= 1`, KEDA starts a job execution
   (scales `0 -> 1`, max 1 parallel replica).
3. **Process** — The Python container (`processor.py`) pulls its image from ACR (via the
   user-assigned managed identity / `AcrPull`), connects to Service Bus with
   `DefaultAzureCredential`, receives one message, and completes it.
4. **Auth** — The user-assigned managed identity holds **Service Bus Data
   Receiver/Sender** on the queue and **AcrPull** on the registry, so no secrets are
   needed at container runtime. The KEDA scaler itself still uses the namespace
   connection-string secret for polling.
5. **Observability** — The Container App Environment streams logs/metrics to the
   Log Analytics workspace.
