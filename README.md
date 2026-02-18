# Практична робота: Моделювання інформаційної системи сповіщень
**Предметна область:** Система автоматичного оповіщення про відключення серверів через SIP/RTP.
**Технологічний стек:** Zabbix (Monitoring) -> API -> Asterisk (FreePBX) -> SIP/IP-Phone.

---

## 1. Діаграма варіантів використання (Use Case Diagram)
Відображає ролі учасників та основні функції системи.



```mermaid
useCaseDiagram
    actor "Сервер (Об'єкт)" as Server
    actor "Адміністратор (Клієнт)" as Admin
    actor "Asterisk (SIP-сервер)" as Asterisk

    package "Система моніторингу та зв'язку" {
        usecase "Виявлення недоступності (Trigger)" as UC1
        usecase "Надсилання запиту на дзвінок (API)" as UC2
        usecase "Встановлення SIP-з'єднання" as UC3
        usecase "Відтворення голосового повідомлення (RTP)" as UC4
    }

    Server --> UC1
    UC1 ..> UC2 : <<include>>
    UC2 --> Asterisk
    Asterisk --> UC3
    UC3 ..> UC4 : <<include>>
    UC4 --> Admin

    sequenceDiagram
    autonumber
    participant S as Server (Down)
    participant Z as Zabbix Server
    participant A as Asterisk (FreePBX)
    participant P as IP-Phone (Client)

    S-X S: Сервіс зупинено
    Z->>Z: Trigger: High Severity
    Note over Z,A: Виклик API (AMI Originate)
    Z->>A: POST /api/v1/make_call (Ext: 101)
    
    A->>P: SIP INVITE (Calling...)
    P-->>A: 180 Ringing
    P-->>A: 200 OK (Слухавку піднято)
    
    Note right of A: Встановлення медіа-потоку
    A->>P: RTP Stream (Audio: "Alert! Server 1 is offline")
    
    P-->>A: SIP BYE (Клієнт поклав слухавку)
    A-->>Z: Call Result: Success


    stateDiagram-v2
    [*] --> Monitoring: Zabbix активний
    Monitoring --> IncidentDetected: Сервер не відповідає
    
    IncidentDetected --> SendApiRequest: Формування API запиту
    
    state "Процес виклику" as CallProcess {
        [*] --> InitiatingCall
        InitiatingCall --> CallRinging
        CallRinging --> Connected: Клієнт відповів
        CallRinging --> NoAnswer: Тайм-аут (30 сек)
    }

    Connected --> PlayAudio: Програти запис (RTP)
    PlayAudio --> [*]

    NoAnswer --> RetryLogic
    state RetryLogic <<choice>>
    RetryLogic --> SendApiRequest: Спроба < 3
    RetryLogic --> LogError: Спроби вичерпано
    
    LogError --> [*]
