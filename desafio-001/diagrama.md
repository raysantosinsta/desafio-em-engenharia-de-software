# Diagrama de Arquitetura — Desafio 001
**Cenário:** Processamento Assíncrono e Controle de Concorrência com Filas

---

## 1. Comparativo: Antes vs. Depois

```mermaid
flowchart LR
    subgraph Antes["❌ ANTES: Arquitetura Acoplada (Gargalo)"]
        direction TB
        C1["Cliente"] -->|"POST (síncrono)"| A1["Node.js API"]
        A1 -->|"Threads / Processos Filhos"| P1["Puppeteer / Playwright<br/>(mesmo container)"]
        P1 -->|"Consumo extremo de recursos"| OOM["💥 Estouro de RAM (OOM)<br/>💥 Timeout em rotas HTTP<br/>💥 Perda de Jobs ao reiniciar"]
    end

    subgraph Depois["✅ DEPOIS: Arquitetura Desacoplada e Resiliente"]
        direction TB
        C2["Cliente"] -->|"1. POST /coleta"| A2["Node.js API"]
        A2 -->|"2. Resposta rápida: HTTP 202"| C2
        A2 -->|"3. Job enfileirado"| Q["Redis / BullMQ"]
        Q -->|"4. Concorrência controlada"| W["Workers Python (Docker)"]
        W -->|"5. Retry + Backoff"| EXT["Fonte Externa"]
        W -->|"6. Salva resultado"| PG[("PostgreSQL")]
        W -->|"7. Evento de conclusão"| WS["WebSocket / Webhook"]
    end
```

---

## 2. Visão Detalhada da Nova Arquitetura

```mermaid
flowchart TD
    subgraph ClientLayer["Camada do Cliente"]
        User["Cliente / Frontend"]
    end

    subgraph IngestionLayer["Camada de Ingestão (Node.js)"]
        API["API REST<br/>• Validação de payload<br/>• Latência &lt; 50ms<br/>• Retorna HTTP 202 + jobId"]
    end

    subgraph QueueLayer["Mensageria & Filas (Redis)"]
        BullMQ["Fila Principal (BullMQ)<br/>Status: pending"]
        DLQ["Dead Letter Queue (DLQ)<br/>Jobs com erro após 3 retries"]
    end

    subgraph WorkerPool["Pool de Workers Dedicados (Python em Docker)"]
        W1["Worker 1<br/>(Concorrência máx: 3)"]
        W2["Worker 2<br/>(Concorrência máx: 3)"]
        Engine["Puppeteer / Playwright<br/>Automação & Scraping"]
        W1 --> Engine
        W2 --> Engine
    end

    subgraph External["Alvo Externo"]
        Target["Fontes Externas / APIs<br/>(Protegido contra Rate Limit)"]
    end

    subgraph StorageNotify["Persistência & Notificação"]
        DB[("PostgreSQL<br/>Dados Estruturados")]
        Notifier["Serviço de Notificação<br/>(WebSocket / Webhook)"]
    end

    %% Conexões do Fluxo
    User -->|"1. POST /coleta"| API
    API -->|"2. Adiciona Job"| BullMQ
    API -->|"3. HTTP 202 Accepted"| User

    BullMQ -->|"4. Consumo de Jobs"| W1
    BullMQ -->|"4. Consumo de Jobs"| W2

    Engine -->|"5. Executa Coleta"| Target

    %% Tratamento de Falhas
    W1 -.->|"Falha após 3x retries<br/>(Exponential Backoff)"| DLQ
    W2 -.->|"Falha após 3x retries<br/>(Exponential Backoff)"| DLQ

    %% Sucesso
    W1 -->|"6. Persiste resultado"| DB
    W2 -->|"6. Persiste resultado"| DB
    W1 -->|"7. Notifica conclusão"| Notifier
    W2 -->|"7. Notifica conclusão"| Notifier
    Notifier -->|"8. Push status concluído"| User
```

---

## 3. Diagrama de Sequência (Ciclo de Vida da Requisição)

```mermaid
sequenceDiagram
    autonumber
    actor Cliente as Cliente / Frontend
    participant API as API Node.js
    participant Queue as BullMQ (Redis)
    participant Worker as Worker Python (Docker)
    participant Target as Fonte Externa
    participant DB as PostgreSQL
    participant DLQ as Dead Letter Queue (DLQ)

    Cliente->>API: POST /coleta (Payload)
    Note over API: Valida payload rapidamente
    API->>Queue: Enfileira Job (status: pending)
    Queue-->>API: Confirmado (jobId gerado)
    API-->>Cliente: HTTP 202 Accepted { jobId: "123", status: "pending" } (latência &lt; 50ms)

    Note over Worker,Queue: Worker consome respeitando limite de concorrência (ex: 3)
    Queue->>Worker: Entrega próximo Job disponível

    rect rgb(240, 248, 255)
        Note over Worker,Target: Execução da Automação (Puppeteer / Playwright)
        Worker->>Target: Inicia raspagem / requisição
        alt Erro transitório / Rate Limit
            Worker->>Worker: Exponential Backoff (espera progressiva e retenta)
        else Falha persistente (3 retentativas esgotadas)
            Worker->>DLQ: Desvia Job para DLQ (inspeção manual / alertas)
        else Sucesso
            Target-->>Worker: Dados brutos obtidos
        end
    end

    Worker->>DB: Salva dados estruturados processados
    Worker->>Cliente: Envia notificação de conclusão (WebSocket / Webhook)
```
