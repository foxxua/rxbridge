# RxBridge: план навчання від простого до складного

Створено 12.10.2026

## Структура репозиторіїв

Головний репозиторій `rxbridge` підключає сервіси як git submodules. Кожен репо створюється тоді, коли до нього доходить етап.

| Репозиторій | Що в ньому | Мова | Етап |
| --- | --- | --- | --- |
| `rxbridge` | Головний: submodules, `compose.yaml`, docs, PLAN, ADR, `CLAUDE.md`, налаштування Keycloak і Grafana | — | 1 |
| `rxbridge-enrollment-service` | Пацієнти, реєстрації | Java | 1 |
| `rxbridge-web-ui` | Thymeleaf + htmx + Web Components, BFF | Java | 2 |
| `rxbridge-adverse-event-service` | Звіти про побічні ефекти | Java | 4 |
| `rxbridge-salesforce-sync-service` | Інтеграція з Salesforce, Pub/Sub API | Java | 5 |
| `rxbridge-events` | Спільна бібліотека контрактів подій, публікується в GitHub Packages | Java | 6 |
| `rxbridge-import-service` | Імпорт CSV, Spring Batch | Java | 7 |
| `rxbridge-config` | Конфігурації всіх сервісів для Config Server | YAML | 8 |
| `rxbridge-config-server` | Spring Cloud Config Server | Java | 8 |
| `rxbridge-discovery-server` | Eureka | Java | 8 |
| `rxbridge-api-gateway` | Spring Cloud Gateway | Java | 8 |
| `rxbridge-notification-service` | Сповіщення, ескалації | Kotlin | 10 |
| `rxbridge-inventory-service` | Склади, резерви, Saga | Kotlin | 11 |

Робота з submodules:
- Клонувати: `git clone --recurse-submodules`
- Після коміту в сервісі — окремий коміт у `rxbridge` з оновленим посиланням на submodule. Так головний репо фіксує сумісний набір версій сервісів
- Claude Code відкривати в корені `rxbridge`

## Як працювати з планом

14 етапів від простого до складного: кожна нова технологія з'являється тоді, коли без неї вже незручно. Так ви розумієте не лише «як», а й «навіщо» — саме про це питають на співбесідах.

- **Інфраструктура відкладається.** Перші 7 етапів — без Eureka, Gateway і Config Server: сервіси бачать один одного за адресами з `application.yml`. Spring Cloud з'являється на етапі 8, коли сервісів уже 4–5 і ручні налаштування стають болем.
- **Тести одразу.** Кожен етап включає тести того, що в ньому зроблено, а не «потім».
- **Кожен етап закінчується робочою версією** і тегом версії в змінених репо (`v0.1.0`, `v0.2.0`…) та в `rxbridge`. Історія видна роботодавцю.
- **Окремий репозиторій на сервіс** (Gradle-проєкт у кожному), зібрані в `rxbridge` через submodules. Новий сервіс — новий репо.
- **ADR** (короткий запис рішення) — коли робите вибір між підходами. 5–10 рядків англійською.
- Секрети тільки в `.env`, який у `.gitignore`.

Складність кожного етапу вказана в заголовку: легко, середньо або складно.

## Етап 1 — Перший сервіс: enrollment-service (легко)

Один сервіс з REST API для пацієнтів і реєстрацій на препарати. Інфраструктура — лише PostgreSQL у `compose.yaml`, який Spring Boot піднімає сам (модуль `spring-boot-docker-compose`).

**Що вивчаєте:** структура Spring Boot 3, шари Controller → Service → Repository, Spring Data JPA, Flyway, Bean Validation, Java 21.

- [ ] Репо `rxbridge` (README-заготовка, `.gitignore`, `docs/PLAN.md`, `CLAUDE.md`)
- [ ] Репо `rxbridge-enrollment-service` (Gradle, Spring Initializr), підключити як submodule
- [ ] Сутності `Patient`, `Enrollment`, `Medication`; міграції Flyway
- [ ] DTO як `record`, статус реєстрації як `sealed interface` або `enum` + `switch` з pattern matching
- [ ] CRUD-ендпоінти, пагінація і сортування
- [ ] Валідація: `@Valid`, власний валідатор формату коду препарату
- [ ] Обробка помилок через `@RestControllerAdvice` і `ProblemDetail`
- [ ] `spring.threads.virtual.enabled=true`
- [ ] Swagger UI через springdoc-openapi
- [ ] Тести: unit (JUnit 5 + Mockito), `@WebMvcTest`, `@DataJpaTest` з Testcontainers

**Готово, коли:** можна створити пацієнта й реєстрацію через Swagger, тести зелені.

## Етап 2 — Веб-інтерфейс web-ui: Thymeleaf + htmx (легко)

Окремий репо `rxbridge-web-ui`, який викликає enrollment-service напряму за адресою з конфігурації. Без безпеки й без Gateway — це прийде пізніше.

**Що вивчаєте:** Thymeleaf (шаблони, layout, фрагменти), htmx, `RestClient` для виклику іншого сервісу.

- [ ] Сторінки: список пацієнтів, картка пацієнта, форма реєстрації
- [ ] Спільний layout (шапка, меню) через фрагменти Thymeleaf
- [ ] htmx: пошук без перезавантаження, пагінація, відправка форми з показом помилок валідації
- [ ] `RestClient` з базовою адресою з `application.yml`; обробка помилок сервісу
- [ ] JS і CSS локально (WebJars або `static/`), без CDN
- [ ] Тести контролерів через `@WebMvcTest` з замоканим клієнтом

**Готово, коли:** через браузер можна знайти пацієнта й зареєструвати його на препарат.

## Етап 3 — Spring Security базово (середньо)

Логін у web-ui через форму і ролі. Користувачі поки в базі даних web-ui, без Keycloak — так ви зрозумієте сам механізм Spring Security, перш ніж його ускладнювати.

**Що вивчаєте:** `SecurityFilterChain`, автентифікація vs авторизація, `UserDetailsService`, `PasswordEncoder`, ролі, CSRF, сесії.

- [ ] Форма логіну, користувачі з BCrypt-паролями в базі
- [ ] Ролі `PHARMACY_OPERATOR` і `ADMIN`; обмеження URL і `@PreAuthorize` на методах
- [ ] Thymeleaf Extras Spring Security: показувати кнопки за роллю
- [ ] CSRF-токен у заголовках запитів htmx
- [ ] Cookie сесії: `HttpOnly`, `Secure`, `SameSite`; заголовок Content Security Policy
- [ ] Тести з `@WithMockUser` і `spring-security-test`

**Готово, коли:** оператор має доступ лише до своїх дій, адмін — до всього; запит без CSRF-токена відхиляється.

## Етап 4 — Другий сервіс і виклики між сервісами (середньо)

Новий `adverse-event-service` зі своєю базою. При створенні звіту він перевіряє в enrollment-service, що пацієнт існує, — перший синхронний виклик між сервісами і перша ситуація «а що, коли інший сервіс впав?».

**Що вивчаєте:** database-per-service, OpenFeign, Resilience4j (timeout, retry, circuit breaker), перший веб-компонент.

- [ ] Репо `rxbridge-adverse-event-service` (submodule), окрема база PostgreSQL
- [ ] Клієнт до enrollment-service через OpenFeign
- [ ] Resilience4j: timeout, retry, circuit breaker; зрозуміла помилка, якщо сервіс недоступний
- [ ] Поле «серйозність» і дедлайн обробки звіту
- [ ] У web-ui: перший веб-компонент (Lit або чистий JS) — таблиця звітів з фільтрами
- [ ] Тести клієнта з WireMock: успіх, таймаут, 500

**Готово, коли:** вимкнули enrollment-service — adverse-event-service відповідає контрольованою помилкою, а не падає.

## Етап 5 — Інтеграція з Salesforce (середньо)

Новий `salesforce-sync-service`. Тут ви сильніші за більшість Java-кандидатів, тому етап іде рано: він дасть впевненість і перший «вау»-пункт для резюме. Поки дані передаються простим HTTP-викликом, без черг.

**Що вивчаєте:** OAuth 2.0 JWT Bearer з боку Java, `RestClient` для зовнішнього API, робота з лімітами.

- [ ] Репо `rxbridge-salesforce-sync-service` (submodule)
- [ ] Salesforce Developer Edition org, об'єкти `Patient_Enrollment__c`, `Adverse_Event__c` з `External_Id__c`
- [ ] Connected App з сертифікатом; отримання токена за JWT Bearer flow, кешування до закінчення строку
- [ ] Upsert за External Id через Composite API (до 25 записів за запит)
- [ ] Обробка помилок Salesforce: часткові помилки в Composite, 401 (оновити токен), `REQUEST_LIMIT_EXCEEDED`
- [ ] enrollment-service після збереження викликає sync-service напряму
- [ ] У web-ui: статус синхронізації запису
- [ ] Тести з WireMock, що імітує відповіді Salesforce

**Готово, коли:** реєстрація з web-ui з'являється в Salesforce org.

**Проблема, яку ви помітите:** якщо sync-service недоступний у момент збереження, дані не дійдуть. Це підводить до етапу 6.

## Етап 6 — Outbox і Kafka (середньо)

Прямий виклик замінюємо надійною доставкою: enrollment-service пише подію в таблицю outbox у тій самій транзакції, а окремий процес публікує її в Kafka. Kafka додається в `compose.yaml` — це перша справжня інфраструктура.

**Що вивчаєте:** транзакційний outbox, Spring Kafka, at-least-once доставка, ідемпотентні споживачі, dead letter topic, версіонування спільної бібліотеки контрактів.

- [ ] Таблиця `outbox_event`; запис у тій самій транзакції, що й бізнес-дані
- [ ] Публікатор: `@Scheduled` читає outbox і відправляє в Kafka
- [ ] Репо `rxbridge-events`: події як `record`, формат JSON; публікація в GitHub Packages з semver-версією
- [ ] Сервіси підключають `rxbridge-events` як залежність з версією; GitHub-токен для читання пакетів — у змінних середовища, не в коді
- [ ] sync-service слухає Kafka; таблиця оброблених подій, щоб дублікат не обробився двічі
- [ ] Retry і dead letter topic для подій, що не пройшли
- [ ] Пачки: накопичувати події і відправляти в Composite API групами
- [ ] Інтеграційні тести з Testcontainers Kafka
- [ ] Перший ADR: «Чому outbox, а не пряма публікація в Kafka»

**Готово, коли:** зупинили sync-service, зберегли 10 реєстрацій, запустили — усі 10 дійшли в Salesforce, без дублікатів.

## Етап 7 — import-service на Spring Batch (середньо)

Масове завантаження CSV від аптек і дистрибуційних центрів. Тепер, коли є Kafka, імпорт просто публікує події, а решта системи вже вміє їх обробляти.

**Що вивчаєте:** Spring Batch (job, step, chunk, reader/processor/writer), skip і retry політики, перезапуск після збою, великі обсяги в Salesforce.

- [ ] Репо `rxbridge-import-service` (submodule), завантаження файлу через web-ui
- [ ] Job: читання CSV → валідація → запис → публікація подій
- [ ] Невалідні рядки пропускаються і потрапляють у звіт помилок, а не валять весь файл
- [ ] Перезапуск job з місця збою
- [ ] У sync-service: для великих пачок Bulk API 2.0 замість Composite (вибір за кількістю записів)
- [ ] Генератор синтетичних тестових даних, файл на 50 000 рядків
- [ ] У web-ui: прогрес імпорту (htmx polling або веб-компонент)

**Готово, коли:** файл на 50 000 рядків імпортується, помилкові рядки видно у звіті, дані в Salesforce.

## Етап 8 — Spring Cloud: Config, Eureka, Gateway (складно)

У вас 5 сервісів, адреси й налаштування розкидані по `application.yml`, порти треба пам'ятати. Тепер Spring Cloud вирішує реальну проблему, і ви зможете пояснити, навіщо він.

**Що вивчаєте:** централізована конфігурація, service discovery, client-side load balancing, API Gateway, профілі середовищ.

- [ ] Репо `rxbridge-config`: налаштування всіх сервісів, профілі `local` і `docker`
- [ ] `rxbridge-config-server` читає конфігурацію з `rxbridge-config`
- [ ] `rxbridge-discovery-server` (Eureka); Feign-клієнти звертаються за іменем сервісу, а не адресою
- [ ] `rxbridge-api-gateway` (Spring Cloud Gateway): маршрути, rate limiting, CORS
- [ ] web-ui ходить до сервісів лише через Gateway
- [ ] Запуск двох екземплярів enrollment-service — перевірити балансування
- [ ] Повний `compose.yaml`: усі сервіси + інфраструктура, healthchecks і порядок старту
- [ ] ADR: «Eureka vs Kubernetes service discovery» (знати різницю для співбесід)

**Готово, коли:** `docker compose up` піднімає всю систему однією командою.

## Етап 9 — Keycloak, OAuth2 і BFF (складно)

Власні користувачі з етапу 3 замінюємо на Keycloak. web-ui стає Backend for Frontend: токени живуть на сервері, браузер має лише cookie сесії.

**Що вивчаєте:** OAuth2 / OpenID Connect, Authorization Code flow, resource server, token relay, ролі з JWT.

- [ ] Keycloak у `compose.yaml`; realm, клієнти й ролі імпортуються з JSON при старті
- [ ] web-ui: `oauth2Login`, вихід з системи, ролі з токена
- [ ] Gateway: token relay — передає токен далі до сервісів
- [ ] Кожен сервіс — resource server, перевіряє JWT і ролі
- [ ] Фонові виклики між сервісами (без користувача): client credentials flow для сервісних акаунтів
- [ ] Тести з `jwt()` у `spring-security-test`
- [ ] ADR: «Чому BFF, а не токен у браузері»

**Готово, коли:** у браузері немає жодного токена (перевірити в DevTools), а сервіс без валідного JWT повертає 401.

## Етап 10 — Kotlin: notification-service (середньо)

Перший сервіс на Kotlin навмисно простий: слухає події з Kafka і надсилає сповіщення. Так ви вчите мову, а не мову й складну логіку одночасно.

**Що вивчаєте:** синтаксис Kotlin, null-safety, data classes, extension functions, Spring Boot на Kotlin, використання Java-бібліотеки `rxbridge-events` з Kotlin.

- [ ] Репо `rxbridge-notification-service` на Kotlin, Gradle Kotlin DSL
- [ ] Слухає події «серйозний побічний ефект» і «реєстрацію схвалено»
- [ ] Відправка email (Mailpit у `compose.yaml` для перегляду листів) і webhook
- [ ] Шаблони листів
- [ ] Ескалація: звіт не оброблено до дедлайну — повторне сповіщення адміну
- [ ] Тести на Kotlin: JUnit 5 + MockK

**Готово, коли:** створили серйозний звіт — лист видно в Mailpit; пропустили дедлайн — прийшла ескалація.

## Етап 11 — Saga: inventory-service на Kotlin (складно)

Залишки препаратів на складах дистрибуційних центрів. Реєстрація стає схваленою лише після резервування препарату; якщо резерву немає — реєстрація відхиляється автоматично. Це розподілена транзакція без спільної бази.

**Що вивчаєте:** Saga pattern (хореографія через події), компенсуючі дії, оптимістичне блокування, Kotlin coroutines (опційно).

- [ ] Репо `rxbridge-inventory-service` на Kotlin: склади, залишки, резерви
- [ ] Ланцюжок: `EnrollmentRequested` → `StockReserved` або `StockReservationFailed` → `EnrollmentApproved` або `EnrollmentRejected`
- [ ] Компенсація: реєстрацію скасовано — резерв повертається на склад
- [ ] Оптимістичне блокування (`@Version`), щоб два запити не забрали одну й ту саму одиницю
- [ ] Стан саги видно в web-ui
- [ ] Тест: 20 паралельних реєстрацій на 10 одиниць — рівно 10 схвалено
- [ ] ADR: «Хореографія vs оркестрація саги»

**Готово, коли:** паралельний тест проходить стабільно, а компенсація повертає залишки.

## Етап 12 — Події з Salesforce через Pub/Sub API (складно)

Двосторонній зв'язок: зміни, зроблені в Salesforce (наприклад, менеджер закрив звіт про побічний ефект), повертаються в систему. Це рідкісна навичка навіть серед Salesforce-розробників.

**Що вивчаєте:** gRPC у Java, Avro-схеми, Change Data Capture, replay ID, відновлення після розриву з'єднання.

- [ ] Увімкнути Change Data Capture для `Adverse_Event__c` в org
- [ ] gRPC-клієнт Pub/Sub API: підписка, отримання й кешування Avro-схеми, декодування подій
- [ ] Зберігати останній replay ID, щоб після перезапуску не втратити й не повторити події
- [ ] Захист від «відлуння»: зміна, яку відправили ви самі, не повертається назад у цикл
- [ ] Події з Salesforce публікуються в Kafka і оновлюють adverse-event-service
- [ ] Перепідключення з backoff при розриві

**Готово, коли:** змінили статус звіту в Salesforce — він оновився в web-ui; перезапуск sync-service не губить події.

## Етап 13 — Спостережуваність: tracing, метрики, логи (середньо)

Коли запит проходить через 5 сервісів і Kafka, без трасування неможливо зрозуміти, де він загубився. На демо це найефектніша частина.

**Що вивчаєте:** Micrometer Tracing, поширення trace ID через HTTP і Kafka, власні метрики, структуровані логи.

- [ ] Micrometer Tracing + Zipkin: один запит видно від web-ui до Salesforce
- [ ] Trace ID у логах кожного сервісу; логи у форматі JSON
- [ ] Prometheus + Grafana у `compose.yaml`
- [ ] Власні метрики: кількість синхронізованих записів, помилки Salesforce, залишок денного ліміту API
- [ ] Дашборд Grafana, експортований у репозиторій
- [ ] Actuator health checks у всіх сервісах, включно з перевіркою зв'язку з Kafka і Salesforce

**Готово, коли:** у Zipkin видно повний шлях реєстрації через усі сервіси, а в Grafana — графік синхронізацій.

## Етап 14 — Фініш: CI/CD, contract tests, README (середньо)

Проєкт, який тімлід може запустити за 5 хвилин і зрозуміти за 10. Від цього етапу залежить, чи відкриють ваш код далі README.

**Що вивчаєте:** GitHub Actions, Spring Cloud Contract, Docker-образи через Buildpacks або Jib.

- [ ] GitHub Actions у кожному сервісі: збірка й тести на кожен push, Testcontainers у CI
- [ ] GitHub Actions у `rxbridge`: підняти всю систему через `compose.yaml` і прогнати наскрізний smoke-тест
- [ ] Docker-образи в GitHub Container Registry
- [ ] Contract tests між enrollment-service і adverse-event-service
- [ ] Покриття тестами (JaCoCo), значок у README
- [ ] README англійською: що це, Mermaid-схема архітектури, запуск однією командою, тестові логіни
- [ ] Папка `docs/adr` у `rxbridge` з усіма рішеннями
- [ ] GIF або 2-хвилинне відео демо
- [ ] Налаштування для запуску без Salesforce org (профіль з WireMock), щоб проєкт працював у будь-кого

**Готово, коли:** людина без вашої допомоги клонує репозиторій і запускає систему за 5 хвилин.

## Коли оновлювати CV і орієнтовні терміни

Оновлюйте CV і починайте відгукуватися після етапу 6 — не чекайте завершення всього проєкту. Терміни нижче — при 3–4 годинах на день; при меншому темпі множте приблизно на 1,5.

| Блок | Етапи | Орієнтовно | Що вже можна сказати роботодавцю |
| --- | --- | --- | --- |
| Основа | 1–3 | 2 тижні | Spring Boot 3, JPA, Thymeleaf, Spring Security |
| Інтеграції | 4–6 | 2–3 тижні | Мікросервіси, Resilience4j, Kafka, outbox, Salesforce API — **оновлюємо CV** |
| Spring Cloud і безпека | 7–9 | 3 тижні | Spring Batch, Config/Eureka/Gateway, OAuth2, Keycloak, BFF |
| Kotlin і складні патерни | 10–12 | 3 тижні | Kotlin, Saga, gRPC, Salesforce Pub/Sub API |
| Фініш | 13–14 | 1–2 тижні | Observability, CI/CD, contract tests |

У README на час пошуку роботи — розділ «Roadmap» з уже зробленим і наступними етапами. Незавершений, але живий проєкт з регулярними комітами виглядає краще, ніж «ідеальний», якого ще немає.
