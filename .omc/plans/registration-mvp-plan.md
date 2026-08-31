# План: Регистрация (MVP) + обучение основам стека

Статус: **pending approval**
Дата: 2026-08-27
Стек: NestJS 11 (Fastify) + TypeORM + PostgreSQL (репозиторий `B:\converter` — подтверждено, это Nest, не Next.js)

## Requirements Summary

Реализовать эндпоинт регистрации по email+паролю, п.1 из присланного ТЗ (`1.1`–`1.6`), со следующими решениями, принятыми на этапе интервью:

- **Email-подтверждение отключено на первой итерации** (флаг заранее закладываем в архитектуру, чтобы включить позже без переписывания).
- **Регистрация НЕ выдаёт JWT/сессию** — она отдельно стоящая функция; вход (login/JWT) — предмет отдельного будущего раздела «Аутентификация» из вашего списка. Это соответствует тому, что в ТЗ Регистрация, RBAC, Аутентификация, Авторизация перечислены как отдельные пункты.
- **Роль (`role`) закладывается в модель пользователя уже сейчас** (со значением по умолчанию `user`), чтобы следующий раздел RBAC не потребовал ломающей миграции.
- Хеширование пароля — `bcryptjs` (чистый JS, без нативной компиляции — проще ставится на Windows без build tools; `bcrypt` с нативными байндингами как более «production» альтернатива упомянута в рисках).

## Мини-теория: как это всё связано (читать один раз перед стартом)

- **Node.js** — рантайм, который выполняет JS вне браузера (у него нет `window`, зато есть доступ к файловой системе, сети, процессам).
- **NestJS** — фреймворк поверх Node, который структурирует backend через **модули** (`@Module`), **контроллеры** (`@Controller`, принимают HTTP-запрос) и **сервисы** (`@Injectable`, бизнес-логика). Контроллер не должен содержать логику — только принять запрос, вызвать сервис, вернуть ответ.
- **Dependency Injection (DI)** — Nest сам создаёт и передаёт экземпляры сервисов туда, где они нужны (через конструктор). Вам не нужно писать `new UsersService()` вручную.
- **PostgreSQL** — реляционная СУБД (сервер, который хранит данные в таблицах со строгой схемой и связями). В этом проекте поднимается через `docker-compose up -d`.
- **TypeORM** — ORM (Object-Relational Mapper): переводит TS-классы (`@Entity`) в SQL-таблицы и обратно, чтобы вы работали с объектами, а не сырыми SQL-запросами.
- **Миграции** — версионируемые файлы, описывающие изменения схемы БД (создать таблицу, добавить колонку). В этом репо `POSTGRES_SYNCHRONIZE=false`, то есть Nest **не** подгоняет схему автоматически под entity — расхождение entity ↔ БД правится только через миграцию. Это осознанный выбор для production-подобной дисциплины.
- **DTO + class-validator** — классы, описывающие форму входных данных (`RegisterDto`), с декораторами (`@IsEmail()`, `@MinLength()`), которые Nest проверяет автоматически через глобальный `ValidationPipe` (уже включён в `src/main.ts:27-31`).

## Acceptance Criteria

1. `POST /auth/register` с валидными `email` + `password` создаёт запись в таблице `users` и возвращает `201` с телом `{ id, email, role, createdAt }` — **без** `passwordHash`.
2. Повторная регистрация с уже существующим `email` возвращает `409 Conflict` с понятным сообщением, запись не дублируется (проверено на уровне приложения И на уровне уникального индекса в БД).
3. Невалидный `email` (не email-формат) или отсутствующий `password` → `400 Bad Request` со списком ошибок валидации (стандартный формат `class-validator`).
4. Пароль короче 8 символов или без цифры/буквы → `400 Bad Request`.
5. Пароль в БД хранится только как хеш (`bcryptjs.hash`, salt rounds ≥ 10) — в дампе таблицы не должно быть исходного пароля.
6. Миграция создаёт таблицу `users` (`id uuid PK`, `email varchar unique not null`, `password_hash varchar not null`, `role varchar not null default 'user'`, `created_at`, `updated_at`) и применяется через `npm run migration:run` без ошибок на чистой БД.
7. `npm run test` проходит, включая новый `auth.service.spec.ts` (юнит-тест на дублирующийся email и на успешный кейс).
8. `npm run lint` и `npm run build` проходят без новых ошибок.

## Implementation Steps

### Шаг 0 — окружение
**Мини-теория:** приложение при старте валидирует `.env` через Joi (`src/core/config/config.validation.ts`) и падает, если чего-то не хватает — это «fail fast», чтобы не ловить баги конфигурации в рантайме.

- Убедиться, что `docker compose up -d` поднял Postgres и `.env` заполнен (уже есть — `PORT=3007`, `POSTGRES_*`).
- `npm run start:dev` должен стартовать без ошибок ДО начала работы (baseline).

### Шаг 1 — зависимости
```
npm install bcryptjs
npm install -D @types/bcryptjs
```
**Почему:** `bcryptjs` — единственная новая зависимость, нужная для MVP регистрации (без JWT/почты — они не в этой итерации).

### Шаг 2 — `User` entity
**Мини-теория:** `@Entity()` — класс = таблица, `@Column()` — поле = колонка, `@PrimaryGeneratedColumn('uuid')` — первичный ключ. Любой `*.entity.ts` файл в `src/` подхватывается автоматически (`autoLoadEntities: true`, глоб в `src/core/database/database.module.ts:25`) — регистрировать вручную не нужно, кроме `TypeOrmModule.forFeature([User])` в модуле, который с ним работает.

Файл: `src/modules/users/entities/user.entity.ts`
```ts
@Entity('users')
export class User {
  @PrimaryGeneratedColumn('uuid') id: string;

  @Column({ unique: true }) email: string;

  @Column({ name: 'password_hash' }) passwordHash: string;

  @Column({ type: 'enum', enum: UserRole, default: UserRole.USER })
  role: UserRole;

  @CreateDateColumn({ name: 'created_at' }) createdAt: Date;
  @UpdateDateColumn({ name: 'updated_at' }) updatedAt: Date;
}
```
`UserRole` enum (`user`, `admin`) — отдельный файл `src/modules/users/entities/user-role.enum.ts`, задел под будущий RBAC-раздел.

### Шаг 3 — `UsersModule` / `UsersService`
**Мини-теория:** `TypeOrmModule.forFeature([User])` регистрирует **репозиторий** — объект с методами `find`, `save`, `findOneBy` и т.д. для конкретной entity, который можно внедрить через `@InjectRepository(User)`.

- `src/modules/users/users.module.ts` — `imports: [TypeOrmModule.forFeature([User])]`, `exports: [UsersService]`.
- `src/modules/users/users.service.ts` — методы `findByEmail(email)`, `createUser({ email, passwordHash })`.
- Зарегистрировать `UsersModule` в `src/core/app/app.module.ts` (уже импортирован, модуль сейчас пустой `@Module({})` — наполнить).

### Шаг 4 — DTO валидации
**Мини-теория:** `ValidationPipe({ whitelist: true })` в `src/main.ts:27-31` отбрасывает лишние поля и автоматически бросает `400`, если DTO с декораторами `class-validator` не проходит проверку — контроллеру не нужно писать `if` вручную.

Файл: `src/modules/auth/dto/register.dto.ts`
```ts
export class RegisterDto {
  @IsEmail() email: string;

  @IsString()
  @MinLength(8)
  @Matches(/^(?=.*[A-Za-z])(?=.*\d).+$/, {
    message: 'password must contain at least one letter and one digit',
  })
  password: string;
}
```
Ответ (без `passwordHash`) — простой mapper-объект в сервисе (без глобального `ClassSerializerInterceptor`, чтобы не усложнять — обсудим при желании позже).

### Шаг 5 — `AuthModule` / `AuthService` / `AuthController`
**Мини-теория:** уникальность email проверяется дважды — на уровне приложения (`findByEmail`, дружелюбная ошибка) и на уровне БД (`unique` индекс, ловит race condition, когда два запроса приходят одновременно). Второе — сеть безопасности, а не дублирование логики.

- `src/modules/auth/auth.module.ts` — `imports: [UsersModule]`.
- `src/modules/auth/auth.service.ts`:
  1. `findByEmail` → если найден → `ConflictException('Email already registered')`.
  2. `bcryptjs.hash(password, 10)`.
  3. `usersService.createUser(...)`.
  4. Обернуть `save()` в `try/catch`, поймать код ошибки Postgres `23505` (unique violation) → тоже `ConflictException` (страховка от гонки).
  5. Вернуть DTO-ответ без `passwordHash`.
- `src/modules/auth/auth.controller.ts` — `@Post('register')`, `@HttpCode(201)`.
- Зарегистрировать `AuthModule` в `src/core/app/app.module.ts` (сейчас модуль импортирован в `app.module.ts`? — нет, `auth.module.ts` существует, но НЕ импортирован в `AppModule`, только `UsersModule` есть — добавить `AuthModule` в `imports`).

### Шаг 6 — миграция
**Мини-теория / важная ловушка репозитория:** скрипт `npm run migration:generate` (`package.json`) вызывает TypeORM CLI с именем `src/database/migrations/Migration`, что генерирует файл `<timestamp>-Migration.ts`. Но и `DatabaseModule` (`src/core/database/database.module.ts:29-31`), и CLI data-source (`src/database/data-source.ts:25`) ищут миграции по маске `*.migration{.ts,.js}` — файл без `.migration.ts` в конце **не будет найден и не применится**. Это несоответствие в шаблоне репозитория, не в вашем коде.

1. Убедиться, что Postgres запущен (`docker compose up -d`).
2. `npm run migration:generate` → сгенерирует `src/database/migrations/<timestamp>-Migration.ts`.
3. **Переименовать** файл в `src/database/migrations/<timestamp>-CreateUsers.migration.ts` (и поправить имя экспортируемого класса на что-то вроде `CreateUsers<timestamp>`, TypeORM это не проверяет по классу, только по содержимому файла/таблице `migrations`, но для читаемости стоит поправить).
4. `npm run migration:run` → проверить `npm run migration:show`.

### Шаг 7 — тесты
**Мини-теория:** юнит-тест мокает репозиторий (не бьёт по реальной БД) и проверяет только логику сервиса; e2e — поднимает всё приложение и реальную БД, проверяет полный HTTP-путь. Для MVP хватит юнит-теста на сервис + один e2e happy-path.

- `src/modules/auth/auth.service.spec.ts` — мок `UsersService`: (a) успешная регистрация хеширует пароль и вызывает `createUser`; (b) существующий email → `ConflictException`.
- (опционально, если останется время) `test/auth.e2e-spec.ts` — реальный POST на `/auth/register`, проверка `201` и `409` при повторе.

### Шаг 8 — верификация вручную
- `npm run start:dev`, затем `curl -X POST http://localhost:3007/auth/register -H "Content-Type: application/json" -d '{"email":"a@a.com","password":"passw0rd"}'` → `201`.
- Повторить тот же запрос → `409`.
- `curl ... -d '{"email":"not-an-email","password":"123"}'` → `400` с деталями по обоим полям.
- Заглянуть в БД (`psql` или любой клиент) — убедиться, что `password_hash` не похож на исходный пароль.

## Risks and Mitigations

| Риск | Митигация |
|---|---|
| Файл миграции не матчит glob `*.migration.ts` (см. Шаг 6) и просто не подхватится — миграция "как будто прошла", а таблицы нет | Явный шаг переименования в плане; проверка `migration:show` перед тем как считать шаг завершённым |
| Race condition: два одновременных запроса с одним email проходят app-level проверку до создания записи | Уникальный индекс в БД + `try/catch` на код ошибки `23505` в сервисе (шаг 5.4) |
| Раскрытие факта существования email через `409` (мелкий security-риск, упомянут в вашем ТЗ п.1.4 как опциональный) | Осознанно принято: для учебного MVP оставляем явную ошибку — это упрощает отладку и обучение; в проде можно заменить на нейтральный ответ. Зафиксировано здесь, чтобы не забыть при финальном security-ревью раздела |
| `bcryptjs` медленнее нативного `bcrypt` под нагрузкой | Не критично для учебного проекта/MVP; если станет узким местом — заменить на `bcrypt` (нативный) без изменения интерфейса сервиса |
| Никогда не было ни одной миграции в этом репо — первый прогон может вскрыть проблемы с `data-source.ts`/env, не связанные с вашим кодом | Шаг 0 — baseline-старт приложения до начала работы, чтобы отделить "сломал я" от "было сломано" |

## Verification Steps
1. `npm run lint` — 0 новых ошибок.
2. `npm run build` — проходит.
3. `npm run test -- auth.service.spec` — зелёный.
4. Ручная проверка из Шага 8 (все 3 сценария: успех, дубликат, невалидные данные).
5. `npm run migration:show` подтверждает применённую миграцию.

## Roadmap (следующие разделы, не в этой итерации)
- **RBAC** — эндпоинты, требующие роль `admin` (`@Roles()` guard), используя уже заложенное поле `role`.
- **Аутентификация** — `POST /auth/login`, выдача JWT (нужен `@nestjs/jwt`, добавление `JWT_SECRET` в `Config`/Joi по образцу `COOKIE_SECRET`).
- **Авторизация** — guards на основе JWT + роли.
- **Email-подтверждение** — включаем через admin-флаг, как заложено в требованиях; понадобится почтовый провайдер (см. раздел «Ресурсы» → «Работа с почтой»).

## Changelog
- v1 (2026-08-27): первичный план по итогам интервью (Nest подтверждён, email-confirm выключен на старте, мини-теория перед каждым шагом).
