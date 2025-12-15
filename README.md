# 🏆 Blinkoff IoT Platform (Current State)

**Secure Modular Monolith** для управления IoT-устройствами с гибридным хранением данных (RAM + DB).

## 🏛 Архитектура

  * **Modular Monolith:** Приложение едино, но четко разделено на бизнес-модули (`telemetry`, `device_management`), которые общаются через Java-интерфейсы.
  * **Hybrid State Management:**
      * **Горячие данные (Real-time):** Статусы (`online`/`offline`), текущая телеметрия и временные токены привязки хранятся в **RAM** (`ConcurrentHashMap`). Это обеспечивает мгновенный отклик (O(1)).
      * **Холодные данные (Persistence):** Данные синхронизируются в **PostgreSQL** асинхронно (фоновый процесс) или синхронно при критических изменениях (регистрация/блокировка).
  * **Security First:**
      * **Traffic:** Весь трафик устройств шифруется **AES-128 (GCM)**.
      * **Device Auth:** Аутентификация по `ChipId` + Белый список в оперативной памяти (`DeviceAuthCache`).
      * **User Binding:** Привязка через временные одноразовые **OTP-токены** (без передачи ChipId пользователю).
      * **Admin Access:** Защита административных эндпоинтов через API Key (`X-Admin-Key`).

-----

## 🛠 Технологический стек

  * **Core:** Java 21, Spring Boot 3.4.x.
  * **Database:** PostgreSQL 15 (используется `JSONB` для гибкости данных).
  * **Protocols:**
      * **WebSocket (Secure Text):** Обмен зашифрованными Base64-строками с устройствами.
      * **HTTP REST:** API для Telegram-бота и администрирования.
  * **Testing:** JUnit 5, Mockito, Spring Boot Test, Testcontainers.

-----

## 📂 Структура проекта (Актуальная)

```text
src/main/java/com/blinkoff/iot
├── BlinkoffIoTApplication.java         // 🚦 Точка входа, @EnableScheduling
│
├── shared                              // 🧱 ОБЩЕЕ ЯДРО
│   ├── config
│   │   ├── AppConstants.java           // Лимиты (Max 10 устройств/юзер)
│   │   ├── WebConfig.java              // 🔥 Регистрация Interceptors
│   │   └── WebSocketConfig.java        // Настройка /ws/device
│   ├── exception                       // Глобальная обработка (ApiError, InvalidTokenException)
│   └── security
│       ├── AdminAuthInterceptor.java   // 🔥 Защита админских ручек (X-Admin-Key)
│       └── crypto                      // AesCryptoEngine (Логика шифрования)
│
└── modules                             // 📦 БИЗНЕС-МОДУЛИ
    │
    ├── device_management               // 🛠 УПРАВЛЕНИЕ (CRUD, API)
    │   ├── controller                  // REST API (Provisioning, Binding, Data)
    │   ├── service                     // Бизнес-логика
    │   │   ├── DeviceProvisioningService.java  // Регистрация, Блокировка/Разблокировка
    │   │   ├── DeviceBindingService.java       // Привязка (обмен Token -> ChipId)
    │   │   └── DeviceDataService.java          // Сборка DTO (RAM + DB + Name Override)
    │   ├── store
    │   │   └── ProvisioningTokenStore.java     // ⚡ RAM: Временные токены (TTL 5 мин)
    │   ├── dto                         // Request/Response (JSON)
    │   ├── model                       // Entity: Device, DeviceBinding
    │   └── repository                  // Spring Data JPA
    │
    └── telemetry                       // 🌡 ТЕЛЕМЕТРИЯ (Real-time)
        ├── api
        │   └── device_facing           // WebSocket слой
        │       ├── DeviceHandler.java              // Прием, Дешифровка, Обновление RAM
        │       └── DeviceHandshakeInterceptor.java // Проверка Header X-Chip-Id + AuthCache
        ├── service
        │   └── DeviceAuthCache.java    // ⚡ RAM: Белый список (Set<String>)
        ├── store
        │   └── InMemoryStateStore.java // ⚡ RAM: Состояние (params, lastSeen, status)
        ├── engine
        │   └── StateSyncService.java   // 🕰 Фоновая синхронизация RAM -> DB
        ├── model                       // Entity: DeviceState
        └── repository                  // Spring Data JPA
```

-----

## 🧪 Архитектура Тестов

```text
src/test/java/com/blinkoff/iot
├── shared
│   └── security
│       ├── AdminAuthInterceptorTest.java      // ✅ UNIT: Проверка доступа по ключу.
│       └── crypto
│           └── AesCryptoEngineTest.java       // ✅ UNIT: AES-GCM шифрование.
│
└── modules
    ├── device_management
    │   ├── service
    │   │   ├── DeviceProvisioningServiceTest.java // ✅ UNIT: Provision, Block, Unblock.
    │   │   ├── DeviceBindingServiceTest.java      // ✅ UNIT: Token Exchange, Limits.
    │   │   └── DeviceDataServiceTest.java         // ✅ UNIT: RAM Priority, Name Override.
    │   ├── store
    │   │   └── ProvisioningTokenStoreTest.java    // ✅ UNIT: Token Creation/Consumption.
    │   ├── controller
    │   │   └── DeviceProvisioningControllerTest.java // ✅ WEB: HTTP Codes (201, 200, 403).
    │   ├── repository
    │   │   ├── DeviceRepositoryTest.java          // 🐢 INTEG: JSONB queries.
    │   │   └── DeviceBindingRepositoryTest.java   // 🐢 INTEG: Constraints.
    │
    └── telemetry
        ├── api
        │   └── device_facing
        │       └── DeviceHandlerTest.java         // ✅ UNIT: WS Connect/Disconnect flow.
        ├── engine
        │   └── StateSyncServiceTest.java          // ✅ UNIT: Cache Warmup & Sync.
        └── repository
            └── DeviceStateRepositoryTest.java     // 🐢 INTEG: CRUD operations.
```

-----

## 🔄 Сценарии Взаимодействия (Interaction Flows)

### 1\. Подключение устройства (Device Connection)

1.  **Handshake:** ESP32 подключается к `ws://...` с заголовком `X-Chip-Id`.
2.  **Auth:** `DeviceHandshakeInterceptor` проверяет наличие ID в `DeviceAuthCache` (RAM).
      * *Есть:* Пускает.
      * *Нет (или Blocked):* 403 Forbidden / Разрыв.
3.  **Status:** `DeviceHandler` ставит флаг `isOnline = true` в `InMemoryStateStore`.

### 2\. Телеметрия и Имена (Telemetry & Naming Strategy)

1.  **Payload:** ESP32 шлет `AES({"temp": 24.5, "name": "My Incubator"})`.
2.  **Process:** Сервер дешифрует и кладет JSON в RAM.
3.  **Name Resolution:** При запросе списка устройств (`/api/devices`):
      * Сервис смотрит в RAM. Если в JSON есть поле `name`, используется оно.
      * Если в RAM пусто или нет имени — берется "Заводское имя" из PostgreSQL (`devices` table).

### 3\. Привязка пользователя (User Provisioning)

1.  **Generate:** Админ (с ключом `X-Admin-Key`) вызывает `POST /token`. Сервер генерирует UUID (хранится в RAM 5 мин).
2.  **Input:** Пользователь отправляет UUID боту. Бот вызывает `POST /bindings`.
3.  **Bind:** Сервер:
      * Проверяет UUID в `ProvisioningTokenStore`.
      * Находит скрытый `ChipId`.
      * Проверяет статус (`isActive`) и лимиты.
      * Создает связь в БД.

-----

## 📡 API Контракт (Server Interface)

Все запросы возвращают JSON.

### 🔐 Security Headers

  * **Публичные (Бот):** Без защиты (аутентификация через логику `userId` / `token`).
  * **Административные:** Требуют заголовок `X-Admin-Key: <SECRET>`.

### 1\. Provisioning (Admin Only 🛡️)

| Метод | URL | Header | Описание |
| :--- | :--- | :--- | :--- |
| **POST** | `/api/devices/provision` | `X-Admin-Key` | Регистрация нового устройства. Ответ: **201**. |
| **POST** | `/api/devices/{id}/token` | `X-Admin-Key` | Генерация токена для привязки. Ответ: `{ "token": "..." }`. |
| **POST** | `/api/devices/{id}/block` | `X-Admin-Key` | **BLOCK**. `isActive=false`, кик из WS, удаление связей. |
| **POST** | `/api/devices/{id}/unblock` | `X-Admin-Key` | **UNBLOCK**. `isActive=true`, возврат в WS Whitelist. |

### 2\. User Actions (Bot / Client)

| Метод | URL | Body / Params | Описание |
| :--- | :--- | :--- | :--- |
| **POST** | `/api/bindings` | `{ "token": "uuid", "userId": "tg_1", "platform": "TELEGRAM" }` | Привязка по токену. Ошибки: 404 (Token Invalid), 400 (Limits). |
| **DELETE** | `/api/bindings` | `?chipId=...&userId=...` | Удаление связи пользователя с устройством. |
| **GET** | `/api/devices` | `?userId=...` | Список устройств (`DeviceSummaryDto`) с актуальными именами. |
| **GET** | `/api/devices/{id}/telemetry` | `?userId=...` | Полная телеметрия (`DeviceTelemetryDto`). |

### 3\. WebSocket (Device)

  * **URL:** `/ws/device`
  * **Header:** `X-Chip-Id: <ID>`
  * **Body:** Encrypted Base64 String.|
