# Software-Design-and-Documentation_lab-1.1|||


```mermaid
graph LR
    Client -- POST --> API[Backend API]
    API -- 2. Process --> MS[Message Service]
    MS -- 3. Store --> DB[(Messages DB)]
    MS -- 5. Push --> Queue[Message Queue]
    Queue -- 6. Trigger --> DS[Delivery Service]
    DS -- 7. Notify Status --> Client
    DS -- 8. Update State --> MS
```

```mermaid
sequenceDiagram
  participant A as User A
  participant ClientA as Client A
  participant API
  participant Msg as Message Service
  participant DB
  participant Queue
  participant ClientB as Client B

  A->>ClientA: Send message
  ClientA->>API: POST /messages
  API->>Msg: createMessage()
  Msg->>DB: save(message, status="sent")
  Msg->>Queue: enqueue delivery
  API-->>ClientA: 202 Accepted

  Queue->>ClientB: push message
  ClientB->>API: POST /messages/{id}/delivered
  API->>Msg: markDelivered(id)
  Msg->>DB: update status="delivered"
  API-->>ClientB: 200 OK
  Msg-->>ClientA: notifyDelivered(id)

  ClientB->>API: POST /messages/{id}/read
  API->>Msg: markRead(id)
  Msg->>DB: update status="read"
  API-->>ClientB: 200 OK
  Msg-->>ClientA: notifyRead(id)
```
