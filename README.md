# Software-Design-and-Documentation_lab-1.1|||


```mermaid
graph TD
    Client[Web / Mobile Client] --> API[Backend API]
    API --> MS[Message Service]
    MS --> DB[(Messages DB)]
    MS --> Queue[Message Queue]
    Queue --> DS[Delivery Service]
    DS --> WS[WebSocket / Push Service]
    WS <--> Client
```

```mermaid
sequenceDiagram
    participant A as User A (Sender)
    participant API as Backend API
    participant Msg as Message Service
    participant DB as Database
    participant B as User B (Recipient)

    A->>API: POST /messages (content)
    API->>Msg: createMessage()
    Msg->>DB: save (status: SENT)
    API-->>A: 202 Accepted (status: SENT)

    Note over Msg, B: Delivery Process
    Msg->>B: Push/WebSocket Delivery
    B->>API: POST /messages/{id}/ack (type: DELIVERED)
    API->>Msg: updateStatus(DELIVERED)
    Msg->>DB: update(status: DELIVERED)
    Msg-->>A: WS Notify: Message Delivered

    B->>API: POST /messages/{id}/ack (type: READ)
    API->>Msg: updateStatus(READ)
    Msg->>DB: update(status: READ)
    Msg-->>A: WS Notify: Message Read
```


```mermaid
stateDiagram-v2
    [*] --> Sent: User sends message
    Sent --> Delivered: Recipient client ACKs receipt
    Delivered --> Read: Recipient opens chat
    Sent --> Failed: Network error / Timeout
    Failed --> Sent: Retry logic
    
    note right of Delivered
        Client-side acknowledgment 
        is required here.
    end note
```


Status
Accepted

Context
Нам потрібно гарантувати точне відображення статусів повідомлень. Сервер може знати, що він відправив повідомлення, але тільки клієнт може підтвердити, що воно було отримане (delivered) або прочитане (read).

Decision
Who updates status: Клієнтська програма ініціює оновлення, надсилаючи короткі повідомлення-підтвердження (ACK) через API або WebSocket.

Missing Acknowledgements: Якщо сервер не отримує delivered протягом певного часу, повідомлення вважається "в черзі". При наступному підключенні клієнта (B), система проводить синхронізацію і повторно надсилає відсутні ACKs.

Alternatives
Server-only tracking: Сервер ставить статус delivered одразу після відправки в сокет. (Відхилено: не гарантує, що додаток на телефоні реально отримав дані).

Polling: Клієнт постійно запитує, чи є нові статуси. (Відхилено: надто велике навантаження на батарею та мережу).

Consequences
+ Висока точність статусів (як у Telegram/WhatsApp).

- Збільшення кількості дрібних запитів до API (необхідна оптимізація через batching підтверджень).
