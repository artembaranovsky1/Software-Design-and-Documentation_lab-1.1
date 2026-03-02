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
    participant UA as User A (Sender)
    participant C as Client A
    participant API as Backend API
    participant MS as Message Service
    participant Q as Message Queue
    participant DS as Delivery Service
    participant CB as Client B (Recipient)

    UA->>C: Write message
    C->>API: POST /send (status: pending)
    API->>MS: processMessage()
    MS->>Q: enqueue delivery task
    API-->>C: 202 Accepted (status: sent)
    
    Q->>DS: trigger delivery
    DS->>CB: push message via WebSocket
    CB-->>DS: ACK (received)
    DS->>MS: updateStatus(delivered)
    MS-->>C: WebSocket: Notify "Delivered"
```


```mermaid
stateDiagram-v2
    [*] --> Sent: Client sends message
    Sent --> Delivered: Recipient's device ACKs receipt
    Sent --> Failed: Network timeout / Error
    Failed --> Sent: Retry logic
    Delivered --> Read: Recipient opens chat/message
    Read --> [*]
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
