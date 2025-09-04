# pgx

`pgx` — это нативный клиент для PostgreSQL на Go с собственным протоколом (без database/sql). Его сила — в полном доступе к возможностям PostgreSQL: типы, пачки запросов, COPY, эффективный пул, транзакции, слушатели уведомлений и пр.

`ДОКУМЕНТАЦИЯ:` <https://pkg.go.dev/github.com/jackc/pgx/v5>
`ПУЛ СОЕДИНЕНИЙ:` <https://pkg.go.dev/github.com/jackc/pgx/v5/pgxpool>

- [pgx](#pgx)
  - [Инициализация соединения](#инициализация-соединения)
    - [Хуки пула pgxpool: BeforeAcquire, AfterRelease, AfterConnect](#хуки-пула-pgxpool-beforeacquire-afterrelease-afterconnect)
  - [Базовые операции: Exec / Query / QueryRow](#базовые-операции-exec--query--queryrow)
    - [Работа со строками результата (pgx.Rows и pgx.Row)](#работа-со-строками-результата-pgxrows-и-pgxrow)
  - [Коллекторы и маппинг в структуры](#коллекторы-и-маппинг-в-структуры)
  - [Транзакции](#транзакции)
    - [DeferrableMode](#deferrablemode)
  - [pgtype — работа с типами PostgreSQL](#pgtype--работа-с-типами-postgresql)
    - [Основные типы pgtype](#основные-типы-pgtype)
  - [Пакетные запросы (SendBatch)](#пакетные-запросы-sendbatch)
  - [Массовая загрузка — CopyFrom](#массовая-загрузка--copyfrom)
  - [Подготовленные выражения (prepared)](#подготовленные-выражения-prepared)
  - [Работа с ошибками](#работа-с-ошибками)
    - [Основные коды ошибок PostgreSQL (SQLSTATE)](#основные-коды-ошибок-postgresql-sqlstate)

## Инициализация соединения

**Особенности pgxpool:**

- Автоматический statement cache внутри соединения.
- Хуки жизненного цикла (AfterConnect, BeforeAcquire), удобно для SET search_path, SET TIME ZONE, регистрации типов.
- TTL/idle, health-check и пр. как в «взрослых» пулах JDBC.

```go
    import (
        "context"
        "time"

        "github.com/jackc/pgx/v5/pgxpool"
    )

    // Подготовка конфигурации пула из DSN.
    cfg, err := pgxpool.ParseConfig(dsn string) // Разбирает DSN/URL в *pgxpool.Config (max conns, timeouts, health-checks).
    // Настройка пула
    cfg.MaxConns = int32(20) // Максимум соединений в пуле.
    cfg.MinConns = int32(2) // Минимум "тёплых" соединений.
    cfg.MaxConnLifetime = 30 * time.Minute // TTL соединения.
    cfg.MaxConnIdleTime = 5 * time.Minute // Idle TTL.
    cfg.HealthCheckPeriod = 1 * time.Minute // Периодический health-check.
    // Хуки:
    // cfg.BeforeAcquire / AfterRelease / AfterConnect — точки расширения (метрики, prepare, SET ...)

    ctx := context.Background()
    pool, err := pgxpool.NewWithConfig(ctx, cfg) // Создаёт пул и устанавливает соединения по мере нужды.
    defer pool.Close() // Важно закрыть при завершении приложения.
```

### Хуки пула pgxpool: BeforeAcquire, AfterRelease, AfterConnect

`Хуки пула` — это функции-обработчики жизненного цикла соединений в pgxpool, которые позволяют внедрять свою логику при создании соединения, при выдаче его из пула и при возврате обратно. Используются для настройки сессии (SET search_path, SET ROLE, SET timezone, application_name), регистрации кодеков/типов, прогрева prepared statements, метрик/трейсинга и мандатного контекста (например, tenant-id для RLS). Хуки задаются в *pgxpool.Config

```go
    cfg.AfterConnect = func(ctx context.Context, c *pgx.Conn) error { /* init session */ }
    // Вызывается ОДИН РАЗ для каждого нового физического соединения, сразу после установления коннекта,
    // но до помещения его в пул. Идеальное место для one-time init: SET-ы по умолчанию, регистрация типов,
    // подготовка SQL по имени (Prepare), включение расширений сессии.

    cfg.BeforeAcquire = func(ctx context.Context, c *pgx.Conn) bool { /* gate */ return true }
    // Вызывается КАЖДЫЙ РАЗ перед тем, как выдать соединение из пула вызову .Query/.Exec/... .
    // Верните true — соединение годно; false — соединение будет уничтожено и будет взято другое.
    // Удобно для установки контекстных параметров сессии (tenant, locale) перед использованием.

    cfg.AfterRelease = func(c *pgx.Conn) bool { /* cleanup */ return true }
    // Вызывается КАЖДЫЙ РАЗ при возврате соединения в пул (после завершения запроса/транзакции).
    // Верните true — вернуть соединение в пул; false — закрыть его (например, если состояние «грязное»).
    // Обратите внимание: контекста здесь НЕТ (дизайн pgx). Используйте безопасные и быстрые операции.

```

## Базовые операции: Exec / Query / QueryRow

```go
    import (
        "context"

        "github.com/jackc/pgx/v5"
        "github.com/jackc/pgx/v5/pgxpool"
    )

    tag, err := pool.Exec(ctx context.Context, sql string, args ...any) // Выполняет команду без набора строк (INSERT/UPDATE/DELETE).
    // CommandTag хранит инфо о кол-ве затронутых строк: tag.RowsAffected()

    rows, err := pool.Query(ctx context.Context, sql string, args ...any) // Возвращает pgx.Rows (итерируемый курсор).
    // Не забудьте rows.Close()

    row := pool.QueryRow(ctx context.Context, sql string, args ...any)    // Удобно, когда ожидается 1 строка.
    // row.Scan(&dest1, &dest2, ...) — сразу сканируем единственную строку.
```

**Пример:**

```go
    import (
        "context"

        "github.com/jackc/pgx/v5"
        "github.com/jackc/pgx/v5/pgxpool"
    )

    // Чтение одной строки.
    var name string
    var count int64
    err := pool.QueryRow(ctx, `
        SELECT current_user, (SELECT COUNT(*) FROM pg_catalog.pg_tables)
    `).Scan(&name, &count) // .Scan — сканирует первую (и единственную) строку.
    // Обработка err: pgx.ErrNoRows — аналог java.sql.SQLException с кодом "no rows".

    // Обход нескольких строк.
    rows, err := pool.Query(ctx, `SELECT id, email FROM users WHERE active = $1`, true)
    defer rows.Close()

    for rows.Next() { // Итерируем как ResultSet.next() в JDBC
        var id int64
        var email string
        if err := rows.Scan(&id, &email); err != nil {
            return err
        }
        // обработка
    }
    if err := rows.Err(); err != nil { // Итоговая ошибка итерации
        return err
    }
```

### Работа со строками результата (pgx.Rows и pgx.Row)

```go
    import "github.com/jackc/pgx/v5"

    ok := rows.Next() // Переходит к следующей строке. false при окончании/ошибке.
    err := rows.Scan(dest...) // Копирует значения текущей строки в указатели.
    err := rows.Err() // Возвращает ошибку итерации.
    rows.Close() // Освобождает ресурсы курсора (важно, если выход раньше конца).

    err := row.Scan(dest...) // Сканирует единственную строку, иначе вернёт pgx.ErrNoRows.
```

## Коллекторы и маппинг в структуры

`Коллекторы (collectors) в pgx/v5` — это набор универсальных функций и «адаптеров строк», которые превращают результат запроса (pgx.Rows) в Go-значения: слайсы, структуры, карты и указатели. Они упрощают итерацию по rows.Next()/Scan() и уменьшают boilerplate

**Особенности:**

- `pgx делает case-insensitive сопоставление`: full_name в SQL = FullName в Go. Но для сложных случаев (например, если колонка называется e_mail), лучше всегда указывать тег.

```go
    import "github.com/jackc/pgx/v5"

    // В pgx используется только ключ:
    // * db:"col_name" — имя колонки 
    // * db:"-" — исключить поле
    type User struct {
        ID       int64  `db:"id"`         // Явное имя колонки: поле ID будет маппиться на колонку "id"
        Email    string `db:"email"`      // Поле Email → колонка "email"
        FullName string `db:"full_name"`  // snake_case колонка → CamelCase поле
        Ignored  string `db:"-"`          // "-" — игнорировать поле, оно не будет участвовать в маппинге
        Description *string `db:"description"` // Если колонка может быть NULL, то через указатель (*string) можно отличить «нет значения» от пустой строки.
    }
    

    items, err := pgx.CollectRows(rows, pgx.RowToStructByName[User]) // Собирает все строки результата в слайс []User.
    // Работает через указанный адаптер (RowToStructByName), который превращает каждую строку в структуру.
    // Удобно, когда нужно загрузить весь набор данных в память (например, список пользователей).

    one, err := pgx.CollectOneRow(rows, pgx.RowToStructByName[User]) // Возвращает только одну строку результата. 
    // Если строк нет — вернёт ошибку pgx.ErrNoRows. Если строк больше — возьмёт первую.
    // Подходит для выборки одиночных объектов, где отсутствие результата допустимо.

    exact, err := pgx.CollectExactlyOneRow(rows, pgx.RowToStructByName[User]) // Требует, чтобы результат содержал ровно одну строку. 
    // Если строк нет или больше одной — вернёт ошибку.
    // Используется для случаев, где по бизнес-логике ожидается уникальный результат (например, поиск по PRIMARY KEY).

    var id int64
    var email string
    // ForEachRow будет вызывать колбэк для каждой строки
    tag, err = pgx.ForEachRow(rows, []any{&id, &email, new(any)}, func() error {
        // Здесь можно делать любые действия с данными без накопления в память
        fmt.Printf("User: id=%d, email=%s\n", id, email)
        return nil
    })
    // В массив scans ([]any{...}) нужно передать указатели на переменные для сканирования. Количество аргументов в []any{} должно совпадать с числом колонок, либо для «лишних» нужно явно передать new(any)
    // Колбэк выполняется после заполнения этих переменных для каждой строки.
    // Метод НЕ собирает данные в память, а обрабатывает их "на лету". 
    // Идеален для стриминговой обработки больших выборок (экспорт в файл, потоковый расчёт).


    val, err := pgx.RowTo[int64](row) // Используется, когда в запросе одна колонка и нужно считать её в простое значение.
    // Например, SELECT COUNT(*) → int64. Удобно для агрегаций или выборки single-value.

    ptr, err := pgx.RowToAddrOf[int64](row) // То же самое, но возвращает *int64. 
    // Полезно, когда колонка может быть NULL — указатель позволит отличить NULL (nil) от "0".

    m, err := pgx.RowToMap(row) // Сканирует строку в map[string]any, где ключ — имя колонки, а значение — её значение.
    // Удобно для динамических запросов, когда структура данных заранее неизвестна (например, админские тулзы).

    s, err := pgx.RowToStructByName[User](row) // Маппит строку в структуру User по имени колонок (snake_case → CamelCase).
    // Требует строгого совпадения: каждая колонка должна найти соответствующее поле в структуре.

    ps, err := pgx.RowToAddrOfStructByName[User](row) // То же самое, что и выше, но возвращает *User.
    // Удобно, если структура может быть nil или её нужно модифицировать по ссылке.

    sL, err := pgx.RowToStructByNameLax[User](row) // "Lax"-вариант: допускает, что у структуры есть поля, которых нет в запросе.
    // Такие поля останутся с нулевыми значениями. Удобно, если SELECT не возвращает полный набор колонок.

    psL, err := pgx.RowToAddrOfStructByNameLax[User](row) // Комбинация "Lax" + возвращение указателя (*User).
    // Часто применяют, когда структура большая и хочется избежать копирования, или при работе с NULL-полями.

    sP, err := pgx.RowToStructByPos[Pair](row) // Маппит по позиции колонок: первая колонка → первое поле, вторая → второе.
    // Требует полного совпадения числа колонок и полей. Удобно для простых SELECT без алиасов.

    psP, err := pgx.RowToAddrOfStructByPos[Pair](row) // То же самое, что RowToStructByPos, но возвращает *Pair.
    // Подходит, когда результат может быть nil или когда нужно работать с указателями на объекты.

    err := pgx.ScanRow(typeMap, fds, row, &id, &email) // Низкоуровневое API: ручное сканирование строки с использованием карты типов (pgtype.Map) и FieldDescription.
    // Применяется для нестандартных типов, кастомного контроля за преобразованиями или собственной логики маппинга.

    buf, err := pgx.AppendRows(buf, rows, pgx.RowToStructByName[User]) // Собирает строки в уже существующий слайс (buf). 
    // В отличие от CollectRows, позволяет контролировать аллокации: можно заранее выделить буфер.
    // Оптимально для больших выборок, где важна производительность.
```

## Транзакции

```go
    import "github.com/jackc/pgx/v5"

    tx, err := pool.Begin(ctx) // Начинает транзакцию (READ COMMITTED по умолчанию).
    defer tx.Rollback(ctx) // Безопасно откатит, если не был Commit.

    err = tx.Commit(ctx) // Фиксирует транзакцию.
    // tx.Rollback(ctx) // Явный откат.

    tx, err = pool.BeginTx(ctx, pgx.TxOptions{
        IsoLevel:   pgx.Serializable,          // Уровень изоляции: ReadCommitted/RepeatableRead/Serializable.
        AccessMode: pgx.ReadWrite,             // ReadWrite / ReadOnly.
        DeferrableMode: pgx.NotDeferrable,     // Для constraint deferrable.
    })

    // Внутри tx доступны те же методы, что и у pool/conn:
    tag, err := tx.Exec(ctx, `UPDATE ...`)
    rows, err := tx.Query(ctx, `SELECT ...`)
    row := tx.QueryRow(ctx, `SELECT ... WHERE id=$1`, id)

    err := pgx.BeginFunc(ctx, pool, func(tx pgx.Tx) error { // BeginFunc сам решает Commit/Rollback
        if _, err := tx.Exec(ctx, `UPDATE accounts SET balance=balance-100 WHERE id=$1`, from); err != nil {
            return err
        }
        if _, err := tx.Exec(ctx, `UPDATE accounts SET balance=balance+100 WHERE id=$1`, to); err != nil {
            return err
        }
        return nil // Commit произойдёт автоматически; ошибка — Rollback.
    })
   
```

### DeferrableMode

В PostgreSQL у ограничений целостности (CONSTRAINTS) есть свойство DEFERRABLE — откладываемое ограничение.

- По умолчанию проверки (например, внешние ключи, уникальные индексы) выполняются немедленно при выполнении SQL-оператора.
- Если ограничение объявлено как DEFERRABLE, то его можно сделать DEFERRED (отложенным) и проверка будет происходить только в момент COMMIT.

pgx.TxOptions.DeferrableMode позволяет задать, как именно будут работать такие ограничения в транзакции.

**Варианты настроек:**

- `DeferrableMode: pgx.NotDeferrable` - Ограничения не откладываются (стандартное поведение).
- `DeferrableMode: pgx.Deferrable` - Ограничения можно откладывать (разрешает переключение в DEFERRED).
- `DeferrableMode: pgx.InitiallyDeferred` - Сразу делать все откладываемые ограничения DEFERRED (проверяются только при COMMIT).

## pgtype — работа с типами PostgreSQL

`pgtype` — это пакет из экосистемы pgx, предоставляющий Go-структуры для работы с типами PostgreSQL.

`ДОКУМЕНТАЦИЯ:` <https://pkg.go.dev/github.com/jackc/pgx/v5/pgtype>

Для чего нужно

1. Безопасная работа с NULL. В Go-примитивы (int, string) нельзя записать NULL.
    - int → не отличишь 0 от NULL.
    - string → не отличишь "" от NULL.
2. Поддержка PostgreSQL-специфичных типов
    - Go не имеет встроенного аналога для UUID, NUMERIC, CIDR, INET, INTERVAL, массивов (int[], text[]) и т.д.
    - Пакет pgtype реализует их через структуры с методами Scan и Value.
3. Совместимость с драйвером
    - pgx и pgtype тесно интегрированы: если в Scan передать pgtype.Text, pgx сам правильно распарсит значение из БД.

**Пример использования:**

```go
    var age pgtype.Int4

    err := pool.QueryRow(ctx, `SELECT age FROM users WHERE id=$1`, 10).Scan(&age)
    if age.Valid { fmt.Println("Возраст:", age.Int32) } 
    else { fmt.Println("Возраст не указан (NULL)") }

    // Вставка JSON
    import "encoding/json"

    payload := map[string]any{"event": "login", "success": true}
    data, _ := json.Marshal(payload)
    _, err := pool.Exec(ctx,
        `INSERT INTO logs(data) VALUES($1)`,
        pgtype.JSON{Bytes: data, Valid: true},
    )

    // UUID
    import "github.com/google/uuid"

    id := uuid.New()
    _, err := pool.Exec(ctx,
        `INSERT INTO users(id, email) VALUES($1, $2)`,
        pgtype.UUID{Bytes: id, Valid: true},
        "test@example.com",
    )
```

### Основные типы pgtype

```go
// Числовые
pgtype.Int2    // smallint
pgtype.Int4    // integer
pgtype.Int8    // bigint
pgtype.Float4  // real
pgtype.Float8  // double precision
pgtype.Numeric // NUMERIC/DECIMAL (сохраняет точность, хранит big.Int)

// Строки и байты
pgtype.Text     // text
pgtype.Varchar  // varchar
pgtype.BPChar   // char(n)
pgtype.Bytea    // bytea (массив байт)

// Булевы
pgtype.Bool // boolean

// Время и даты
pgtype.Date       // date
pgtype.Time       // time without timezone
pgtype.Timestamp  // timestamp without timezone
pgtype.Timestamptz // timestamp with time zone
pgtype.Interval   // interval

// UUID
pgtype.UUID // uuid

// JSON
pgtype.JSON  // json
pgtype.JSONB // jsonb

// Сетевые типы
pgtype.Inet // inet
pgtype.CIDR // cidr
pgtype.MAC  // macaddr

// Массивы
pgtype.Int4Array
pgtype.TextArray
// и другие массивы для всех базовых типов
```

## Пакетные запросы (SendBatch)

`Батч` — это способ отправить на сервер несколько SQL-команд одним раунд-трипом, а затем последовательно прочитать их результаты. В pgx батч строится через pgx.Batch, отправляется методом SendBatch, а результаты читаются через pgx.BatchResults. Это уменьшает сетевые задержки и даёт контроль порядка выполнения.

```go
    b := &pgx.Batch{} // Создаём пустой батч (контейнер команд)

    b.Queue(`INSERT INTO logs(msg) VALUES($1)`, "a") // Добавить команду №1 (SQL + args)
    b.Queue(`INSERT INTO logs(msg) VALUES($1)`, "b") // Команда №2
    b.Queue(`SELECT COUNT(*) FROM logs`) // Команда №3 (SELECT без аргументов)

    // Можно ставить и подготовленные выражения по имени (если вы их заранее Prepare на conn):
    b.Queue(`selUserByID`, 42)  // Имя prepared-statement вместо SQL

    // Выполнение пакетного запроса
    br := pool.SendBatch(ctx, b) // *pgxpool.Pool
    br := conn.SendBatch(ctx, b) // *pgx.Conn
    br := tx.SendBatch(ctx, b)   // pgx.Tx (внутри транзакции); делает батч атомарным в рамках tx
    defer br.Close() // Обязательно закрываем соединение

    tag,  err := br.Exec() // Прочесть результат следующей команды как CommandTag
    rows, err := br.Query() // Прочесть результат следующей команды как Rows
    row := br.QueryRow() // Прочесть результат следующей команды как Row
    err := br.Close() // Завершить чтение всех результатов и закрыть батч
```

## Массовая загрузка — CopyFrom

`CopyFrom` — высокопроизводительный путь массовой вставки данных в PostgreSQL через протокол COPY ... FROM STDIN. В отличие от пачек INSERT, CopyFrom минимизирует сетевые раунд-трипы и кодировку/декодировку текста, поэтому обычно в разы быстрее для больших объёмов

```go
    import "github.com/jackc/pgx/v5"

    // Где доступен CopyFrom
    n, err := pool.CopyFrom(ctx, table pgx.Identifier, columns []string, src pgx.CopyFromSource) // Через пул
    n, err := conn.CopyFrom(ctx, table pgx.Identifier, columns []string, src pgx.CopyFromSource) // На соединении
    n, err := tx.CopyFrom(ctx,   table pgx.Identifier, columns []string, src pgx.CopyFromSource) // В транзакции
    // Возвращает n — сколько строк реально загружено.
    // Работает только с обычной таблицей (или представлением, поддерживающим COPY). ON CONFLICT тут нельзя — это не INSERT

    // CopyFrom читает данные из объекта, реализующего интерфейс
    type CopyFromSource interface {
        Next() bool                 // перейти к следующей строке
        Values() ([]any, error)     // значения текущей строки (по колонкам)
        Err() error                 // итоговая ошибка итерации
    }

    src := pgx.CopyFromRows(rows [][]any) // Простой способ: уже есть [][]any (сформировано заранее в памяти)
    src := pgx.CopyFromSlice(n int, func(i int) ([]any, error)) // Ленивый генератор строк по индексу [0..n)

    // Настройка назначения (таблица/колонки)
    table := pgx.Identifier{"schema", "table"} // Имя таблицы. Можно без схемы: pgx.Identifier{"table"}
    cols  := []string{"col1", "col2", "col3"}  // Порядок важен: соответствует порядку значений в Values()
    // Если колонке в таблице задан DEFAULT, и вы хотите, чтобы он сработал, не включайте эту колонку в cols. Если включите — придётся явно передавать значение (или nil → это будет именно NULL, а не DEFAULT).
    // nil в []any даёт NULL в целевой ячейке.
    // Типы значений кодируются через типовую карту pgx (pgtype.Map). Простые Go-типы (int, string, time.Time, []byte) и большинство pgtype.* кодируются из коробки

    // Примеры использования

    rows := [][]any{
        {1, "alice@example.com"},
        {2, "bob@example.com"},
    }
    n, err := pool.CopyFrom(ctx,
        pgx.Identifier{"users"},              // таблица
        []string{"id", "email"},              // колонки
        pgx.CopyFromRows(rows),               // источник
    ) // n == 2
    // Быстро и просто, но весь набор в памяти — годится для умеренных объёмов

    // Потоковый сценарий: лениво генерируем строки (CopyFromSlice)
    items := make([]UserDTO, 0, 100_000) // предположим, заполнено выше
    n, err := pool.CopyFrom(ctx,
        pgx.Identifier{"users"},
        []string{"id", "email", "full_name"},
        pgx.CopyFromSlice(len(items), func(i int) ([]any, error) {
            u := items[i]
            return []any{u.ID, u.Email, u.FullName}, nil
        }),
    )
```

## Подготовленные выражения (prepared)

`Prepared statement` — это SQL-выражение, которое один раз отправляется и компилируется PostgreSQL, после чего может многократно выполняться с разными параметрами.

**В pgx начиная с v5:**

- Statement cache включён по умолчанию для всех соединений в пуле.
- Запросы (pool.Query, pool.Exec, ...) автоматически кешируются и повторно используются на этом соединении.
- Поэтому в большинстве случаев ручной Prepare не нужен.

**Когда использовать ручной Prepare, а когда нет:**

**Не нужно:**

- Обычные запросы (даже часто повторяющиеся) → pgx сам закеширует их на соединении.
- Если вы не управляете соединениями вручную (а только пулом).

**Нужно:**

- Хотите заранее прогреть соединение и «пришпилить» набор выражений (через AfterConnect хук пула).
- Критичные по скорости запросы, где вы хотите исключить даже первый «пустой» парсинг.
- Нужен контроль имени/жизни подготовленного выражения (например, для совместимости со сторонними тулзами).

```go
    _, err := conn.Prepare(ctx, name string, sql string) // Подготовить выражение с именем (на conn)
    row := conn.QueryRow(ctx, name, args...) // Выполнить по имени (или SQL напрямую)
    rows, err := conn.Query(ctx, name, args...) // То же, но для множества строк
    ct, err := conn.Exec(ctx, name, args...) // Выполнить без возврата строк


    conn, err := pool.Acquire(ctx) // Берём соединение из пула
    defer conn.Release()

    // Подготовка
    _, err = conn.Conn().Prepare(ctx, "selUser", `SELECT id, email FROM users WHERE id=$1`)

    // Выполнение (имя, args...)
    row := conn.Conn().QueryRow(ctx, "selUser", 42)

```

## Работа с ошибками

В pgx (как и в официальном драйвере lib/pq) ошибки PostgreSQL возвращаются как структура *pgconn.PgError, содержащая все поля серверной ошибки (код, сообщение, детализацию, контекст)

**Основные типы ошибок:**

1. Ошибки SQL-сервера
   - Представлены типом *pgconn.PgError.
   - Возникают при нарушении constraint’ов, синтаксических ошибках, проблемах доступа.
2. Ошибки клиента/драйвера
   - pgx.ErrNoRows — результат пустой выборки при QueryRow.Scan().
   - pgx.ErrTxClosed — попытка использовать закрытую транзакцию.
   - pgx.ErrCopyInProgress — конфликт при использовании CopyFrom.
3. Ошибки контекста/сети
   - context.Canceled, context.DeadlineExceeded — таймауты, отмены.
   - Ошибки подключения (connection refused, broken pipe) приходят как обычные error.

**Структура pgconn.PgError:**

```go
    import "github.com/jackc/pgx/v5/pgconn"

    type PgError struct {
        Severity         string // "ERROR", "FATAL", "PANIC"
        Code             string // SQLSTATE (5-символьный код ошибки)
        Message          string // Основной текст ошибки
        Detail           string // Дополнительные детали
        Hint             string // Подсказка от сервера
        Position         int32  // Позиция ошибки в SQL
        InternalPosition int32
        Where            string
        SchemaName       string
        TableName        string
        ColumnName       string
        DataTypeName     string
        ConstraintName   string
        File             string // имя файла в исходниках PostgreSQL
        Line             int32
        Routine          string // функция PostgreSQL, где ошибка возникла
    }
```

### Основные коды ошибок PostgreSQL (SQLSTATE)

Postgres использует стандарт SQLSTATE (5 символов).

`Все коды ошибок:` <https://github.com/jackc/pgerrcode/blob/master/errcode.go>

**Самые популярные:**

| Код     | Константа в `pgerrcode`             | Смысл                                      |
| ------- | ----------------------------------- | ------------------------------------------ |
| `23505` | `UniqueViolation`                   | Нарушение `UNIQUE` или `PRIMARY KEY`       |
| `23503` | `ForeignKeyViolation`               | Нарушение `FOREIGN KEY`                    |
| `23502` | `NotNullViolation`                  | Нарушение `NOT NULL`                       |
| `23514` | `CheckViolation`                    | Нарушение `CHECK`                          |
| `42P01` | `UndefinedTable`                    | Таблица не найдена                         |
| `42703` | `UndefinedColumn`                   | Колонка не найдена                         |
| `42601` | `SyntaxError`                       | Ошибка синтаксиса                          |
| `40001` | `SerializationFailure`              | Ошибка сериализации (повторить транзакцию) |
| `40P01` | `DeadlockDetected`                  | Deadlock                                   |
| `28P01` | `InvalidPassword`                   | Неверный пароль                            |
| `28000` | `InvalidAuthorizationSpecification` | Ошибка авторизации                         |
