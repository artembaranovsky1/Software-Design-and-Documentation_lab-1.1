# Software-Design-and-Documentation_lab-1.1|||


```mermaid
graph LR
    %% Клієнтська сторона
    C1[Клієнт-Відправник]
    C2[Клієнт-Отримувач]

    %% Точка входу та сервіси
    API[API Gateway]
    Auth[Auth Service]
    MS[Message Service]
    SS[Status Service]

    %% Черга та надійна доставка
    MQ[Message Queue]
    DS[Delivery Service]
    WS[WebSocket / Push Service]

    %% Зберігання даних
    DB[(Messages DB)]

    %% Основний потік: відправлення
    C1 -- "POST /messages" --> API
    API -- "Validate" --> Auth
    API -- "Create" --> MS
    MS -- "Save" --> DB
    MS -- "Set status 'sent'" --> SS
    MS -- "Publish" --> MQ
    MQ -- "Consume" --> DS
    DS -- "Push via WS" --> WS
    WS -- "Deliver" --> C2

    %% Потік підтверджень: статуси
    C2 -- "ACK delivered" --> WS
    WS -- "Deliver ACK" --> SS
    C2 -- "ACK read" --> WS
    
    %% Оновлення та сповіщення про статус
    SS -- "Update" --> DB
    SS -- "Notify sender" --> WS
    WS -- "Status update" --> C1

    %% Стилізація для виділення Status Service (центр твого завдання)
    style SS fill:#e1f5fe,stroke:#01579b,stroke-width:2px
```

```mermaid
sequenceDiagram
    participant A as User A (Sender)
    participant API as Backend API
    participant MS as Message Service
    participant DB as Database
    participant DS as Delivery Service
    participant B as User B (Recipient)

    A->>API: POST /messages (content)
    API->>MS: processMessage()
    MS->>DB: save(status: "sent")
    MS->>DS: triggerDelivery()
    API-->>A: 202 Accepted (status: sent)

    DS->>B: Deliver via WebSocket
    B-->>API: POST /messages/{id}/ack (delivered)
    API->>MS: updateStatus(delivered)
    MS->>DB: update(status: "delivered")
    MS-->>A: Notify: Status Delivered (SignalR/WS)

    B->>B: User opens chat
    B-->>API: POST /messages/{id}/ack (read)
    API->>MS: updateStatus(read)
    MS->>DB: update(status: "read")
    MS-->>A: Notify: Status Read
```


```mermaid
stateDiagram-v2
    [*] --> Sent: Message saved in DB
    Sent --> Delivered: Client B acknowledges receipt
    Delivered --> Read: Client B opens message
    
    Sent --> Failed: Network timeout / No ACK
    Failed --> Sent: Retry logic
    
    note right of Delivered
        Client sends ACK (delivered) 
        when app receives payload
    end note

    note right of Read
        Client sends ACK (read) 
        when message enters viewport
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
