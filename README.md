# 🏆 Blinkoff IoT Platform (Current State)

**Статус:** Event-Driven Stabilization (Kafka + Watchdog).
**Тип:** Secure Modular Monolith для управления IoT-устройствами с гибридным хранением (RAM + DB) и реактивной системой уведомлений.

## 🏛 Архитектура

* **Modular Monolith:** Приложение разделено на модули `telemetry`, `device_management` и `notification`.
* **Hybrid State Management:**
* **Hot Data (RAM):** Текущие параметры, статус `Online/Offline`, `LastSeen` и токены привязки живут в памяти (`ConcurrentHashMap`). Чтение/Запись — O(1).
* **Cold Data (DB):** Асинхронная синхронизация (Write-Behind) в **PostgreSQL** раз в 10 сек. Восстановление состояния (Rehydration) при рестарте сервера.


* **Event-Driven Core:**
* Все критические изменения (ошибки устройств, потеря связи, подключение) публикуются в **Apache Kafka**.
* Внешние сервисы (Telegram Bot) подписываются на топик и реагируют независимо.


* **Security First:**
* **Traffic:** Шифрование **AES-128 (GCM)** для всего трафика устройств.
* **Network:** `DeviceHandshakeInterceptor` блокирует неавторизованные устройства на уровне TCP-рукопожатия.
* **Admin API:** Защита через `X-Admin-Key`.



---

## 🛠 Технологический стек

* **Core:** Java 21, Spring Boot 3.4.x.
* **Message Broker:** Apache Kafka (Topic: `iot-alarms`).
* **Database:** PostgreSQL 15 (`JSONB` для гибкости).
* **Protocols:**
* **WebSocket (Secure):** Двусторонний обмен зашифрованными данными.
* **HTTP REST:** API управления.


* **Testing:** Testcontainers (Postgres, Kafka), Mockito.

---

## 🔄 Сценарии Взаимодействия (Interaction Flows)

### 1. Подключение и Телеметрия (Device Lifecycle)

1. **Handshake:** ESP32 стучится в `ws://host/ws/device` с хедером `X-Chip-Id`. Сервер сверяет ID с `DeviceAuthCache` (RAM).
2. **Connection Event:** При успехе сервер шлет событие `CONNECTION_ESTABLISHED` в Kafka.
3. **Payload Processing:**
* Устройство шлет AES-шифрованный JSON.
* Сервер дешифрует, обновляет `lastSeen` и параметры в RAM.
* **Анализ:** Если в JSON есть поля `errors` или `events`, сервер генерирует соответствующие события (`DEVICE_ERROR`, `DEVICE_EVENT`) в Kafka.



### 2. Мониторинг Связи (Watchdog & Dead Man's Switch)

1. **Polling:** Фоновый процесс (`DeviceConnectivityWatchdog`) каждые 10 сек сканирует все активные устройства.
2. **Detection:**
* Если устройство `Online`, но молчит > 60 сек -> Перевод в `Offline`, событие `CONNECTION_LOST_TIMEOUT` (переход состояния).
* Если устройство уже `Offline` и продолжает молчать -> Периодическое событие `CONNECTION_NOT_FOUND` (Heartbeat of failure) для напоминания.


3. **Socket Close:** Если сокет закрыт явно (RST/FIN), генерируется событие `CONNECTION_BROKEN`.

### 3. Маршрутизация Уведомлений (Bot Interaction)

1. **Consumer:** Бот читает сообщение из Kafka.
2. **Enrichment:** Бот запрашивает у Core API список владельцев для этого `chipId` (`GET /owners`).
3. **Delivery:** Бот отправляет сообщение в Telegram конкретным пользователям.

---

## 📡 API Контракт (Server Interface)

Все HTTP-запросы возвращают JSON. Ошибки возвращаются в формате `{ "status": "ERROR", "message": "...", "timestamp": "..." }`.

### 🔐 Security Headers

* **Admin/System:** Требуют `X-Admin-Key: <SECRET_KEY>`.
* **Public/Bot:** Используют логику токенов в теле запроса.

### 1. Device Provisioning & Management (Admin 🛡️)

| Метод | URL | Параметры / Body | Описание |
| --- | --- | --- | --- |
| **POST** | `/api/devices/provision` | `{ "chipId": "...", "name": "..." }` | Регистрация нового чипа. Ответ: **201 Created**. |
| **POST** | `/api/devices/{id}/block` | - | **БАН**. Разрыв WS, очистка кэша, удаление связей. |
| **POST** | `/api/devices/{id}/unblock` | - | **РАЗБАН**. Разрешает подключение. Юзеры должны привязать заново. |
| **POST** | `/api/devices/{id}/token` | - | Генерация временного токена (TTL 5 мин) для привязки. |
| **GET** | `/api/devices/{id}/owners` | `?platform=TELEGRAM` | **NEW.** Получить список ID пользователей (chat_id), владеющих устройством. Фильтр `platform` опционален. |

### 2. User Binding & Data (Bot / Client)

| Метод | URL | Параметры / Body | Описание |
| --- | --- | --- | --- |
| **POST** | `/api/bindings` | `{ "token": "uuid", "userId": "tg_1", "platform": "TELEGRAM" }` | Привязка устройства по OTP-токену. |
| **DELETE** | `/api/bindings` | `?chipId=...&userId=...` | Отвязка устройства пользователем. |
| **GET** | `/api/devices` | `?userId=...` | Список устройств пользователя (`DeviceSummaryDto`). Имена берутся из RAM (если переопределены) или DB. |
| **GET** | `/api/devices/{id}/telemetry` | `?userId=...` | Полный JSON состояния устройства. |

---

## ⚡ WebSocket Contract (Device Protocol)

Протокол взаимодействия "Устройство <-> Сервер".

* **URL:** `ws://host:8080/ws/device`
* **Header:** `X-Chip-Id: <CHIP_ID>`
* **Формат сообщения:** Строка (Base64), содержащая шифрованный AES-128 JSON.

### Структура JSON (до шифрования)

Устройство может отправлять любые данные, но поля `errors` и `events` имеют специальное значение.

```json
{
  "temp": 37.5,                 // Любые параметры телеметрии
  "hum": 60.0,
  "name": "My Incubator",       // Опционально: переопределение имени в RAM
  
  // 🔥 Специальные поля для алертинга
  "errors": [                   // Если массив не пуст -> летит DEVICE_ERROR
    "Sensor Failure", 
    "Heater Error" 
  ],
  "events": [                   // Если массив не пуст -> летит DEVICE_EVENT
    "Incubation Started",
    "Door Opened"
  ]
}

```

---

## 📣 Kafka Event Contract (Notification System)

Контракт для потребителей (Telegram Bot, Push Service).
**Topic:** `iot-alarms`

### JSON Structure (`DeviceAlarmEvent`)

```json
{
  "chipId": "ESP-TEST-01",
  "type": "DEVICE_ERROR",       // Тип события (см. Enum)
  "message": "Critical failure", // Читаемое описание для логов
  "payload": [                  // Список деталей (ошибки, события или null)
    "Sensor Failure",
    "Overheat"
  ],
  "timestamp": "2025-12-16T12:00:00Z"
}

```

### Event Types (Enum)

| Тип | Источник | Описание |
| --- | --- | --- |
| `CONNECTION_ESTABLISHED` | WebSocket | Устройство успешно подключилось и прошло авторизацию. |
| `CONNECTION_BROKEN` | WebSocket | Соединение разорвано явно (закрыт сокет, ошибка сети). |
| `CONNECTION_LOST_TIMEOUT` | Watchdog | Устройство числилось Online, но перестало слать данные (>60с). Переход в Offline. |
| `CONNECTION_NOT_FOUND` | Watchdog | Устройство долго Offline. Периодическое напоминание (Heartbeat of failure). |
| `DEVICE_ERROR` | Device JSON | В телеметрии пришел непустой массив `"errors": [...]`. |
| `DEVICE_EVENT` | Device JSON | В телеметрии пришел непустой массив `"events": [...]`. |

### 📂 Подробное описание структуры и классов

```text
src/main/java/com/blinkoff/iot
├── BlinkoffIoTApplication.java         // 🚦 Запуск Spring Boot. Включает @EnableScheduling для Watchdog и SyncService.
│
├── shared                              // 🧱 ОБЩЕЕ ЯДРО (Утилиты, Конфигурация, Безопасность)
│   ├── config
│   │   ├── AppConstants.java           // 📏 Глобальные константы (макс. кол-во устройств, таймауты).
│   │   ├── WebConfig.java              // ⚙️ Настройка Spring MVC: здесь мы регистрируем AdminAuthInterceptor.
│   │   ├── WebSocketConfig.java        // 🔌 Настройка WebSocket endpoints (/ws/device) и привязка DeviceHandler.
│   │   └── DataSeeder.java             // 🌱 Загрузка начальных данных в БД при старте (тестовый юзер, устройство).
│   ├── exception                       // ⚠️ Обработка ошибок
│   │   ├── ApiError.java               // DTO для красивого JSON-ответа при ошибке (код, сообщение, время).
│   │   ├── GlobalExceptionHandler.java // @ControllerAdvice: ловит исключения и превращает их в ApiError.
│   │   └── InvalidTokenException.java  // Кастомное исключение для неверных токенов привязки.
│   └── security
│       ├── AdminAuthInterceptor.java   // 🛡️ Перехватчик HTTP-запросов. Проверяет заголовок X-Admin-Key.
│       └── crypto                      // 🔐 Криптография
│           ├── AesCryptoEngine.java    // Реализация AES-GCM (шифрование/дешифрование строк).
│           ├── KeyProvider.java        // Интерфейс для получения секретных ключей.
│           ├── KeyType.java            // Enum типов ключей (DEVICE_TRAFFIC, USER_DATA).
│           └── StaticKeyProvider.java  // Простая реализация: хранит ключи в application.yaml.
│
└── modules                             // 📦 БИЗНЕС-МОДУЛИ
    │
    ├── device_management               // 🛠 УПРАВЛЕНИЕ (CRUD, Метаданные, Права)
    │   ├── controller                  // 🌐 REST Контроллеры (принимают HTTP)
    │   │   ├── DeviceBindingController.java      // API для юзеров: привязать/отвязать устройство.
    │   │   ├── DeviceDataController.java         // API для юзеров: получить список своих устройств и их данные.
    │   │   └── DeviceProvisioningController.java // API для Админа/Бота: создание, блок, генерация токенов, список владельцев.
    │   ├── service                     // 🧠 Бизнес-логика
    │   │   ├── DeviceProvisioningService.java  // Логика завода: создать чип, забанить (active=false), чистка кэшей.
    │   │   ├── DeviceBindingService.java       // Логика связей: проверка токена, создание связи, поиск владельцев по PlatformType.
    │   │   └── DeviceDataService.java          // Логика отображения: сборка DTO, приоритет имени (DB vs RAM).
    │   ├── store
    │   │   └── ProvisioningTokenStore.java     // ⚡ RAM: Хранит короткоживущие UUID-токены для привязки (Map<Token, ChipId>).
    │   ├── dto                         // 📦 Объекты передачи данных (JSON)
    │   │   ├── BindRequest.java        // { token, userId, platform }
    │   │   ├── DeviceSummaryDto.java   // { name, status, temp, ... } (для списков)
    │   │   ├── DeviceTelemetryDto.java // Полный JSON состояния.
    │   │   ├── ProvisionRequest.java   // Данные для регистрации завода.
    │   │   └── ProvisionResponse.java  // Ответ завода.
    │   ├── model                       // 🗄️ Сущности базы данных (Entities)
    │   │   ├── enums/PlatformType.java // Enum: TELEGRAM, ANDROID, WHATSAPP.
    │   │   ├── enums/AccessRole.java   // Enum: OWNER, ADMIN, VIEWER.
    │   │   ├── Device.java             // Таблица devices (chip_id, name, active).
    │   │   └── DeviceBinding.java      // Таблица bindings (связь user_id <-> chip_id).
    │   └── repository                  // 🐢 Доступ к БД (Spring Data JPA)
    │       ├── DeviceBindingRepository.java // SQL-запросы к bindings (поиск по платформе, удаление).
    │       └── DeviceRepository.java        // SQL-запросы к devices.
    │
    ├── notification                    // 🔔 УВЕДОМЛЕНИЯ (Kafka)
    │   ├── event
    │   │   └── DeviceAlarmEvent.java   // 📨 DTO события: ChipID, тип (ERROR/EVENT/LOST), payload (текст).
    │   └── kafka
    │       └── AlarmProducer.java      // 📣 Kafka Producer: отправляет DeviceAlarmEvent в топик 'iot-alarms'.
    │
    └── telemetry                       // 🌡 ТЕЛЕМЕТРИЯ (Real-time, WebSocket)
        ├── api
        │   └── device_facing
        │       ├── DeviceHandler.java              // WebSocket Handler: connection open/close, decrypt msg, detection errors/events.
        │       └── DeviceHandshakeInterceptor.java // Handshake: проверяет X-Chip-Id перед соединением, сверяет с AuthCache.
        ├── service
        │   └── DeviceAuthCache.java    // ⚡ RAM (Set<String>): "Белый список" активных ID. Загружается при старте.
        ├── store
        │   └── InMemoryStateStore.java // ⚡ RAM (ConcurrentMap): Потокобезопасное хранение "params", "lastSeen", "isOnline".
        ├── engine
        │   ├── StateSyncService.java           // 🕰 Scheduler: раз в 10 сек сбрасывает данные из RAM в PostgreSQL. При старте восстанавливает RAM из БД.
        │   └── DeviceConnectivityWatchdog.java // 🐕 Scheduler: ищет устройства, которые "зависли" (Timeout) или давно "Not Found". Шлет алерты.
        ├── model
        │   └── DeviceState.java        // Таблица device_states (JSONB params, last_seen).
        └── repository
            └── DeviceStateRepository.java // SQL-запросы к device_states.
```

-----

### 🔄 Взаимодействие классов (Interaction Flows)

#### Сценарий А: Устройство шлет телеметрию (Happy Path)

1.  **Handshake:** Устройство стучится в `/ws/device`. `DeviceHandshakeInterceptor` проверяет, есть ли ChipID в `DeviceAuthCache` (RAM).
2.  **Message:** `DeviceHandler` получает сообщение.
3.  **Crypto:** `AesCryptoEngine` расшифровывает сообщение.
4.  **Update:** `DeviceHandler` вызывает `InMemoryStateStore.updateParams(...)`, обновляя JSON и `lastSeen`.
5.  **Events:** `DeviceHandler` проверяет JSON на наличие полей `errors` или `events`.
      * Если есть -\> вызывает `AlarmProducer` -\> сообщение улетает в Kafka.
6.  **Persistence:** Параллельно `StateSyncService` (по таймеру) берет снимок из `InMemoryStateStore` и сохраняет его через `DeviceStateRepository` в PostgreSQL.

#### Сценарий Б: Устройство потеряло связь (Watchdog Flow)

1.  **Monitor:** `DeviceConnectivityWatchdog` просыпается каждые 10 сек.
2.  **Check:** Он берет список *всех* разрешенных устройств из `DeviceAuthCache`.
3.  **Verify:** Для каждого ID он спрашивает у `InMemoryStateStore`: "Когда он был в сети (`lastSeen`)?".
4.  **Decision:**
      * Если `isOnline=true`, но прошло \> 60 сек -\> Watchdog ставит статус `false` в Store и вызывает `AlarmProducer` (тип `CONNECTION_LOST_TIMEOUT`).
      * Если `isOnline=false` и продолжает молчать -\> `AlarmProducer` (тип `CONNECTION_NOT_FOUND`) для цикличного напоминания.

#### Сценарий В: Регистрация и Привязка пользователя (Admin & User Flow)

1.  **Provision:** Админ вызывает `DeviceProvisioningController`.
      * Проверка `AdminAuthInterceptor`.
      * `DeviceProvisioningService` сохраняет `Device` в БД и добавляет ID в `DeviceAuthCache`.
2.  **Token:** Админ запрашивает токен. `DeviceProvisioningService` -\> `ProvisioningTokenStore` (создает UUID).
3.  **Bind:** Юзер в Телеграме отправляет токен. Бот вызывает `DeviceBindingController`.
      * `DeviceBindingService` проверяет токен в `ProvisioningTokenStore`.
      * Если ок -\> сохраняет запись в `DeviceBindingRepository` (с указанием `PlatformType.TELEGRAM`).

#### Сценарий Г: Бот получает уведомление (Notification Flow)

1.  **Source:** Происходит событие (ошибка в `DeviceHandler` или таймаут в `Watchdog`).
2.  **Produce:** `AlarmProducer` кидает JSON в Kafka (`iot-alarms`).
3.  **Consume (внешний сервис Бота):** Слушатель читает событие.
4.  **Enrich:** Бот идет в `DeviceProvisioningController` (`/owners?platform=TELEGRAM`).
      * Контроллер вызывает `DeviceBindingService`.
      * Сервис через `DeviceBindingRepository` находит Chat ID всех владельцев.
5.  **Notify:** Бот отправляет сообщение пользователю.

## 🧪 Архитектура Тестов (Актуальная)

```text
src/test/java/com/blinkoff/iot
├── shared
│   └── security
│       ├── AdminAuthInterceptorTest.java       // ✅ UNIT: Проверка доступа по ключу (X-Admin-Key).
│       └── crypto
│           └── AesCryptoEngineTest.java        // ✅ UNIT: AES-GCM шифрование/дешифрование.
│
└── modules
    ├── device_management
    │   ├── service
    │   │   ├── DeviceProvisioningServiceTest.java // ✅ UNIT: Регистрация, Блокировка, Разблокировка.
    │   │   ├── DeviceBindingServiceTest.java      // ✅ UNIT: Обмен токена, Фильтрация по PlatformType.
    │   │   └── DeviceDataServiceTest.java         // ✅ UNIT: Приоритет данных (RAM vs DB).
    │   ├── store
    │   │   └── ProvisioningTokenStoreTest.java    // ✅ UNIT: Генерация и потребление токенов.
    │   ├── controller
    │   │   └── DeviceProvisioningControllerTest.java // ✅ WEB: Эндпоинты (включая новый /owners), Mock сервисов.
    │   ├── repository
    │   │   ├── DeviceRepositoryTest.java          // 🐢 INTEG: Поиск JSONB.
    │   │   └── DeviceBindingRepositoryTest.java   // 🐢 INTEG: Constraints, удаление по ChipId.
    │
    ├── notification                            // 🔔 NOTIFICATION (Kafka)
    │   └── kafka
    │       └── AlarmProducerTest.java             // ✅ UNIT: Проверка отправки DTO в KafkaTemplate.
    │
    └── telemetry
        ├── DeviceIntegrationTest.java             // 🚀 E2E: Интеграционный тест полного цикла (опционально).
        ├── api
        │   └── device_facing
        │       └── DeviceHandlerTest.java         // ✅ UNIT: WebSocket Flow + Отправка событий (Error/Event/Broken).
        ├── engine
        │   ├── StateSyncServiceTest.java          // ✅ UNIT: Прогрев кэша (WarmUp) и восстановление (Restore).
        │   └── DeviceConnectivityWatchdogTest.java// ✅ UNIT: Логика таймаутов и статуса "Not Found".
        └── repository
            └── DeviceStateRepositoryTest.java     // 🐢 INTEG: Сохранение состояний в БД.

```
