# Java Backend Roadmap

Цель: читать и писать код в `backend/core`, `backend/domain-lib` и `backend/postgres-core` на уровне
уверенного junior+/middle. План построен от реального стека этих репозиториев.

Темп: ~1–1.5 часа в день, 5 дней в неделю, итого около 6 месяцев. При большем времени этапы сжимаются.

## Что в стеке (из pom.xml)

| Область | `core` + `domain-lib` (Dossym) | `postgres-core` (Scalend) |
|---|---|---|
| Язык | Java 17 | Java 25 |
| Фреймворк | Spring Boot 3.2 | Spring Boot 4 (новые модули `spring-boot-starter-webmvc`, `spring-boot-flyway`) |
| Хранилище | MongoDB, QueryDSL | PostgreSQL, JPA/Hibernate, Flyway; миграция данных из Mongo |
| Интеграции | OpenFeign, Kafka, Camunda, Temporal | Kafka, Spring Modulith events (outbox) |
| Архитектура | пакеты по фичам, контракты `*Client` в domain-lib | модули Spring Modulith, hexagonal: `adapter / application / domain` |
| Прочее | Spring Statemachine, MapStruct, Lombok, Joda-Time, springdoc | MapStruct, Lombok, Caffeine, iText/POI (PDF, Excel) |
| Наблюдаемость | Micrometer, Prometheus, Zipkin/Brave, Actuator | то же |
| Тесты | JUnit 5, Mockito, AssertJ, standalone MockMvc, mongo-java-server | embedded Postgres, `@DataJpaTest`, Modulith tests |
| Деплой | Docker, Helm, GitLab CI | то же |

## Мосты из TypeScript

| TypeScript / фронт | Java / бэк |
|---|---|
| `package.json`, npm | `pom.xml`, Maven |
| `interface`, `type` | `interface`, `record`, `class` |
| union types, discriminated unions | `sealed interface` + `record` + `switch` с pattern matching |
| `?.`, `??` | `Optional`, `Objects.requireNonNullElse` |
| `array.map/filter/reduce` | Stream API |
| `Promise`, `async/await` | `CompletableFuture`, виртуальные потоки |
| декораторы (`@Component` в Angular/NestJS) | аннотации (`@Service`, `@RestController`) |
| DI в Angular/NestJS | Spring IoC-контейнер |
| zod | Bean Validation (`@NotNull`, `@Valid`) |
| `fetch`/axios-клиент | Feign-интерфейс (`*Client` в domain-lib) |
| Jest | JUnit 5 + AssertJ + Mockito |
| Redux reducer / XState | Spring Statemachine |

Главные отличия, на которых спотыкаются фронтендеры:
- статическая типизация без escape hatch `any`: дженерики стираются в рантайме (type erasure);
- `==` сравнивает ссылки, для объектов нужен `equals`; у `Integer` есть ловушка кеша -128..127;
- checked exceptions: компилятор заставляет обработать или объявить исключение;
- деньги только в `BigDecimal`, никогда в `double`;
- многопоточность реальная: код выполняется параллельно, общий изменяемый state опасен.

---

## Этап 0. Окружение (1–2 дня)

- [x] Установить SDKMAN и через него JDK 17 и 25 (сейчас в системе только JDK 27; `core` на нём не собирается из-за Lombok).
- [ ] IntelliJ IDEA (Community хватит на старте, для Spring удобнее Ultimate).
- [x] Maven через SDKMAN.
- [x] Превратить `learn-java` в Maven-проект с JUnit 5. Убрать `Hello.class` из репозитория, завести git.

## Этап 1. Ядро языка (4 недели)

1. Синтаксис, примитивы и обёртки, `String` (неизменяемость, `StringBuilder`), `var`.
2. ООП: классы, конструкторы, модификаторы доступа, `static`, `final`, наследование и композиция, абстрактные классы, интерфейсы, `default`-методы (они нужны в domain-lib для новых методов `*Client`).
3. `equals`/`hashCode`/`toString`, `record`, `enum` с полями и методами.
4. Дженерики: `List<T>`, bounded types, `? extends` / `? super`.
5. Коллекции: `List`, `Set`, `Map`, их реализации и сложность операций, неизменяемые коллекции (`List.of`).
6. Исключения: checked и unchecked, `try-with-resources`, свои исключения.
7. Лямбды, функциональные интерфейсы, Stream API, `Optional`.
8. Современная Java 17–25: `sealed`, pattern matching в `switch` и `instanceof`, text blocks, record patterns.
9. Даты и деньги: `java.time` (`LocalDate`, `Instant`, `ZoneId`), Joda-Time (есть в domain-lib), `BigDecimal` и `RoundingMode`.

**Проект 1: калькулятор графика платежей (CLI).** Аннуитетный и дифференцированный график по сумме, ставке и сроку, `BigDecimal`, `record` для строки графика, `sealed` для типа графика. Покрыть тестами.

## Этап 2. Инструменты и тесты (1.5 недели)

1. Maven: структура проекта, lifecycle (`compile`, `test`, `package`, `install`), scopes, `dependencyManagement`, BOM, parent POM, SNAPSHOT-версии. Зачем `mvn install` в `domain-lib` перед сборкой `core`.
2. JUnit 5: `@Test`, `@ParameterizedTest`, `@Nested`; AssertJ; Mockito (`@Mock`, `@InjectMocks`, `when/verify`).
3. Lombok: `@Getter`, `@Builder`, `@RequiredArgsConstructor`, `@Slf4j`, и что он генерирует под капотом.
4. Отладка в IntelliJ: breakpoints, evaluate, чтение stack trace.

**Практика:** довести Проект 1 до 80%+ покрытия (JaCoCo), добавить параметризованные тесты.

## Этап 3. JVM и конкурентность (1.5 недели)

1. Как работает JVM: байткод, classpath, heap и stack, GC на уровне понимания.
2. Потоки, `ExecutorService`, `CompletableFuture`, виртуальные потоки (Java 21+).
3. Гонки, `synchronized`, `volatile`, `Atomic*`, `ConcurrentHashMap`, почему неизменяемость решает большую часть проблем.

**Практика:** параллельный расчёт графиков для 10 000 займов: сравнить по времени последовательный вариант, пул потоков и виртуальные потоки.

## Этап 4. HTTP и Spring Boot (5 недель)

1. HTTP-сервер глазами бэкенда: статусы, идемпотентность, REST-дизайн, сериализация JSON (Jackson).
2. Spring Core: IoC, бины, scopes, `@Configuration`/`@Bean`, внедрение через конструктор, профили, `application.yml`, `@ConfigurationProperties`.
3. Spring Web MVC: `@RestController`, `@RequestBody`, `@PathVariable`, `@RequestHeader` (у вас `X-user-id`), `ResponseEntity`.
4. Валидация (`@Valid`, кастомные валидаторы), глобальная обработка ошибок (`@RestControllerAdvice`, `ProblemDetail`).
5. Слои: controller → service → repository, DTO и маппинг через MapStruct.
6. OpenAPI через springdoc, Actuator.
7. HTTP-клиенты: OpenFeign. Паттерн из вашего `core`: Feign-интерфейс в `domain-lib` реализуется контроллером.
8. Как устроена автоконфигурация Spring Boot, что поменялось в Boot 4 по сравнению с 3.

**Проект 2: Loan API.** REST-сервис займов: создать заявку, одобрить, получить график (переиспользовать Проект 1). Валидация, обработка ошибок, OpenAPI, тесты контроллеров через standalone `MockMvc`, как в `core`. Пока хранение в памяти.

## Этап 5. Данные (5 недель)

1. SQL и PostgreSQL: `JOIN`, агрегации, индексы, `EXPLAIN`, транзакции, уровни изоляции, блокировки (`SELECT ... FOR UPDATE`).
2. JPA/Hibernate: сущности, связи, lazy/eager, проблема N+1, persistence context, dirty checking, optimistic locking (`@Version`).
3. Spring Data JPA: репозитории, derived queries, `@Query`, пагинация, проекции, `@Transactional` и его ловушки (self-invocation, `readOnly`, propagation).
4. Flyway: версионные миграции (`V001__baseline.sql` в `postgres-core`).
5. MongoDB и Spring Data MongoDB: документы, когда документ, а когда таблица. Это нужно для `core` и для понимания миграции Mongo → Postgres в `postgres-core`.
6. QueryDSL: типобезопасные динамические запросы.
7. Тесты слоя данных: `@DataJpaTest`, embedded Postgres / Testcontainers.

**Практика:** перевести Проект 2 на PostgreSQL + Flyway, добавить фильтры и пагинацию, тесты репозиториев на реальной БД в Docker.

## Этап 6. Безопасность (1.5 недели)

1. Spring Security: filter chain, аутентификация и авторизация, `@PreAuthorize`.
2. JWT и OAuth2 Resource Server. Как ваш `cerberus-spring-boot-starter` встраивается в приложение.
3. OWASP-минимум для бэкенда: SQL-инъекции, mass assignment, утечки в сообщениях об ошибках, секреты в конфигах.

**Практика:** закрыть Loan API ролями `ADMIN` и `CLIENT`.

## Этап 7. Интеграции и архитектура (4 недели)

1. Kafka: топики, партиции, consumer groups, offsets, at-least-once, идемпотентные консьюмеры, `@KafkaListener`, `KafkaTemplate`.
2. Transactional outbox и Spring Modulith: модули, `ApplicationModules`-тесты, события, event externalization в Kafka (это ядро `postgres-core`).
3. Hexagonal architecture: ports и adapters на примере `postgres-core/.../provider/loanrequest/{adapter,application,domain}`.
4. Spring Statemachine: жизненный цикл займа и операции в `core`.
5. Оркестрация процессов: Temporal (workflows, activities, детерминизм) и Camunda external tasks, на уровне чтения кода.
6. Кеширование: Caffeine, `@Cacheable`.
7. ADR: как в `core/docs/adr` фиксируют решения.

**Проект 3: развитие Loan API.** Выделить модули через Spring Modulith, публиковать событие «заявка одобрена» в Kafka через outbox, отдельный консьюмер «уведомления». Статусы заявки через state machine.

## Этап 8. Наблюдаемость и деплой (1.5 недели)

1. Логирование: SLF4J + Log4j2, структурированные логи, MDC.
2. Метрики: Micrometer → Prometheus; трейсинг: Brave → Zipkin, trace id через Feign и Kafka.
3. Docker (multi-stage build для Java), docker-compose для Postgres + Kafka.
4. Kubernetes и Helm на уровне чтения чартов в `helm/`, GitLab CI.

**Практика:** упаковать Проект 3 в Docker, поднять всё через compose, посмотреть трейс запроса в Zipkin.

## Этап 9. Боевой код (постоянно, с этапа 4)

- Экскурсии по `core` и `postgres-core`: от HTTP-запроса до БД и Kafka.
- Чтение MR коллег, вопросы ментору по непонятным местам.
- Первые маленькие задачи: тест, `default`-метод в `*Client`, новая миграция, новый эндпоинт.

---

## Как работаем

1. Я объясняю тему через аналогии с TS/React и даю упражнение.
2. Ты пишешь код в этом репозитории сам. За тебя я его не пишу, разве что пример на 5–10 строк для иллюстрации.
3. Я делаю ревью: что хорошо, что не идиоматично для Java и почему.
4. В конце темы короткая проверка вопросами. Отмечаем чекбоксы в этом файле.

Полезные источники:
- dev.java/learn — официальный туториал по языку;
- «Effective Java», Joshua Bloch — после этапа 2;
- docs.spring.io/spring-boot и spring.io/guides;
- «Java Persistence with Spring Data and Hibernate» или блог Vlad Mihalcea — к этапу 5;
- «Designing Data-Intensive Applications», Martin Kleppmann — фоном, к этапам 5 и 7.
