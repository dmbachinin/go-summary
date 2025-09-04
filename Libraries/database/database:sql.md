# database/sql

`database/sql` — это стандартный пакет Go для работы с реляционными базами данных. Он определяет **унифицированный интерфейс** для доступа к БД, но не содержит реализации драйверов.  

Для подключения к конкретной СУБД (например PostgreSQL) нужно использовать сторонний драйвер, который реализует интерфейсы `database/sql/driver`.

`ДОКУМЕНТАЦИЯ:` <https://pkg.go.dev/database/sql>

- [database/sql](#databasesql)
  - [Особенности](#особенности)
  - [Инициализация соединения](#инициализация-соединения)
    - [DataSourceName (DSN)](#datasourcename-dsn)
  - [Выполнение запросов](#выполнение-запросов)
    - [\*sql.DB - Пул соединений](#sqldb---пул-соединений)
    - [\*sql.Conn - Закрепленное соединение с БД](#sqlconn---закрепленное-соединение-с-бд)
    - [\*sql.Tx - Транзакции](#sqltx---транзакции)
    - [\*sql.Stmt - Подготовленные выражения](#sqlstmt---подготовленные-выражения)
    - [\*sql.Rows, \*sql.Row sql.Result - Чтение результатов](#sqlrows-sqlrow-sqlresult---чтение-результатов)

## Особенности

- **Абстракция**: работает через интерфейсы, сам пакет не умеет подключаться к БД.
- **Драйверы подключаются через импорт**: `_ "github.com/lib/pq"`.
- **SQL-запросы остаются на стороне разработчика**: Go не скрывает SQL, а лишь управляет подключениями и результатами.
- **Поддержка пула соединений** встроена в `sql.DB`.
- **Гибкая обработка ошибок**: большинство методов возвращают `error`.

## Инициализация соединения

```go
    import (
        "database/sql"
        _ "github.com/lib/pq" // подгружаем драйвер PostgreSQL
    )

    // sql.Open(driverName, dataSourceName) // Создаёт объект подключения к БД (пул соединений).
    db, err := sql.Open("postgres", "host=localhost user=postgres password=123 dbname=test sslmode=disable") // Важно: реальное соединение не устанавливается, пока не будет выполнен первый запрос
    // "postgres" → это имя драйвера (регистрация происходит внутри драйвера)

    db.Ping() // Проверяет соединение с БД

    db.SetMaxOpenConns(10) // Максимум открытых соединений с БД.
    // Если запросов больше — новые будут ждать освобождения соединений.
    // 0 (значение по умолчанию) = без ограничений.

    db.SetMaxIdleConns(5) // Максимум "праздных" (неиспользуемых, но открытых) соединений в пуле.
    // Эти соединения не закрываются, а остаются доступными для повторного использования.
    // Значение по умолчанию = 2.
    // Если поставить 0 — пул не будет хранить неиспользуемые соединения.

    db.SetConnMaxLifetime(time.Hour) // Максимальное "время жизни" соединения.
    // После этого срока соединение закрывается и заменяется новым.
    // Полезно для балансировщиков, ротации и обновлений БД.
    // 0 (по умолчанию) = соединение живёт бесконечно.

    db.SetConnMaxIdleTime(30 * time.Minute) // Максимальное "время простоя" соединения в пуле.
    // Если соединение простаивает дольше — оно закрывается.
    // 0 (по умолчанию) = без ограничений.
```

**Обычно параметры подбирают под нагрузку:**

- **SetMaxOpenConns** ≈ число соединений, допустимое в БД.
- **SetMaxIdleConns** ≈ 10–30% от SetMaxOpenConns.
- **SetConnMaxLifetime** выставляют меньше, чем таймаут у балансировщика/файрвола.

### DataSourceName (DSN)

`dataSourceName` (DSN) — это строка подключения к базе данных, которую использует `database/sql` для передачи параметров драйверу.  
Формат этой строки зависит **от конкретного драйвера**, так как пакет `database/sql` лишь передаёт её драйверу «как есть».

**Общие правила:**

- Формат DSN не стандартизирован в Go.
- Всегда указывается в документации конкретного драйвера.
- Может быть в виде **URI** или в виде **ключ=значение**.
- Порядок и наличие параметров зависят от драйвера.
- Ошибки в DSN приводят к ошибкам при `db.Open` или `db.Ping`.

**Пример:** PostgreSQL (`github.com/lib/pq`)

**Ключи:**

- **host** — адрес сервера
- **port** — порт (по умолчанию 5432)
- **user** — имя пользователя
- **password** — пароль
- **dbname** — база данных
- **sslmode** — режим SSL (disable, require, verify-ca, verify-full)

```go
    // 1. Формат ключ=значение
    dsn := "host=localhost port=5432 user=postgres password=123 dbname=test sslmode=disable"
    db, err := sql.Open("postgres", dsn)

    // 2. Формат URL
    dsn := "postgres://postgres:123@localhost:5432/test?sslmode=disable"
    db, err := sql.Open("postgres", dsn)
```

## Выполнение запросов

**Базовые сущности и когда их использовать:**

- `*sql.DB` — пул соединений. Используйте для «обычных» запросов без явного закрепления за одним TCP-коннектом.
- `*sql.Conn` — закреплённое соединение из пула. Нужен для драйвер-специфичных операций, которые требуют одного и того же соединения (напр., session variables, temp tables).
- `*sql.Tx` — транзакция.
- `*sql.Stmt` — подготовленный запрос (prepared statement).
- `*sql.Rows/*sql.Row` — курсоры для чтения результатов.

**Placeholder’ы (важно при переносе с JDBC)**. **В Go placeholder-ы определяет драйвер**, а не database/sql. Примеры: ? (MySQL, SQLite), $1, $2, ... (PostgreSQL), @p1 (SQL Server). В отличие от JDBC нет именованных параметров в стандартном пакете (они есть в сторонних библиотеках, напр. sqlx).

### *sql.DB - Пул соединений

```go
    import "database/sql"

    res, err := db.Exec(query string, args ...any) // Выполняет команду (INSERT/UPDATE/DELETE/DDL). Возвращает sql.Result. Блокирует до завершения.

    res, err := db.ExecContext(ctx context.Context, query string, args ...any) // То же, но учитывает отмену/таймаут через ctx.

    rows, err := db.Query(query string, args ...any) // SELECT на множество строк; вернёт *sql.Rows. ОБЯЗАТЕЛЬНО rows.Close().

    rows, err := db.QueryContext(ctx context.Context, query string, args ...any) // SELECT с контекстом.

    row := db.QueryRow(query string, args ...any) // SELECT, ожидается максимум 1 строка; возвращает *sql.Row (ленивый Scan).

    row := db.QueryRowContext(ctx context.Context, query string, args ...any) // То же с контекстом.

    stmt, err := db.Prepare(query string) // Готовит выражение для многократного выполнения; вернёт *sql.Stmt. ОБЯЗАТЕЛЬНО stmt.Close().

    stmt, err := db.PrepareContext(ctx context.Context, query string) // Подготовка с контекстом.
```

**Пример использования:**

```go
    // Пример чтения множества строк с правильным управлением ресурсами.
    func LoadUsers(ctx context.Context, db *sql.DB) ([]User, error) {
        rows, err := db.QueryContext(ctx, `SELECT id, name, email, created_at FROM users WHERE active = $1`, true)
        if err != nil {
            return nil, err
        }
        defer rows.Close()

        var out []User
        for rows.Next() {
            var u User
            var email sql.NullString // Для NULL используйте sql.NullString, sql.NullInt64, sql.NullBool, sql.NullTime и т.п.
            if err := rows.Scan(&u.ID, &u.Name, &email, &u.CreatedAt); err != nil { // QueryRow* откладывает ошибку до Scan; обязательно обрабатывайте её
                return nil, err
            }
            if email.Valid { // Valid == true, если в БД не NULL
                u.Email = email.String
            }
            out = append(out, u)
        }
        if err := rows.Err(); err != nil { // ОБЯЗАТЕЛЬНО
            return nil, err
        }
        return out, nil
    }
```

### *sql.Conn - Закрепленное соединение с БД

```go
    import "database/sql"

    conn, err := db.Conn(ctx context.Context) // Выдаёт закреплённое соединение *sql.Conn. ОБЯЗАТЕЛЕН conn.Close().

    err := conn.Raw(func(driverConn any) error { return nil }) // Доступ к нативному соединению драйвера (расширенные возможности). Используйте осторожно.

    res, err := conn.ExecContext(ctx context.Context, query string, args ...any) // Аналог db.ExecContext, но гарантированно в рамках одного соединения.

    rows, err := conn.QueryContext(ctx context.Context, query string, args ...any) // Аналог db.QueryContext.

    row := conn.QueryRowContext(ctx context.Context, query string, args ...any) // Аналог db.QueryRowContext.

    stmt, err := conn.PrepareContext(ctx context.Context, query string) // Подготовка выражения, привязана к этому соединению.

    tx, err := conn.BeginTx(ctx context.Context, opts *sql.TxOptions) // Транзакция поверх конкретного соединения.

```

### *sql.Tx - Транзакции

```go
    import "database/sql"

    tx, err := db.Begin() // Начинает транзакцию с настройками по умолчанию (уровень изоляции — драйвер-зависим).

    tx, err := db.BeginTx(ctx context.Context, opts *sql.TxOptions) // Старт транзакции с контекстом и параметрами (ReadOnly, Isolation).

    res, err := tx.Exec(query string, args ...any) // Выполнение команд в рамках транзакции.

    res, err := tx.ExecContext(ctx context.Context, query string, args ...any) // С контекстом.

    rows, err := tx.Query(query string, args ...any) // SELECT в транзакции.

    rows, err := tx.QueryContext(ctx context.Context, query string, args ...any) // SELECT с контекстом.

    row := tx.QueryRow(query string, args ...any) // Ожидается 1 строка.

    row := tx.QueryRowContext(ctx context.Context, query string, args ...any) // С контекстом.

    stmt, err := tx.Prepare(query string) // Подготовка выражения внутри транзакции.

    stmt := tx.Stmt(stmt *sql.Stmt) // Привязывает ранее подготовленный на db stmt к текущей транзакции (удобно для переиспользования).

    stmt := tx.StmtContext(ctx context.Context, stmt *sql.Stmt) // То же с контекстом.

    err := tx.Commit() // Фиксация транзакции. После Commit/Rollback объект tx становится непригоден.

    err := tx.Rollback() // Откат транзакции. ВАЖНО: делать defer Rollback на случай ошибок/паник.
```

**Пример использования:**

```go
    func DoBusinessOp(ctx context.Context, db *sql.DB) error {
        ctx, cancel := context.WithTimeout(ctx, 5*time.Second) // общий таймаут на всю транзакцию
        defer cancel()

        // Уровень изоляции и ReadOnly зависят от поддержки драйвером.
        tx, err := db.BeginTx(ctx, &sql.TxOptions{
            Isolation: sql.LevelSerializable, // или sql.LevelReadCommitted/sql.LevelRepeatableRead - желаемый уровень изоляции. Фактический уровень — как умеет драйвер/СУБД
            ReadOnly:  false, // Может дать СУБД дополнительные оптимизации
        })
        if err != nil {
            return err
        }
        // Всегда откатываем по умолчанию; Commit вручную на «счастливом пути».
        defer tx.Rollback()

        // ... любые Exec/Query внутри tx ...

        return tx.Commit()
    }
```

### *sql.Stmt - Подготовленные выражения

```go
    import "database/sql"

    res, err := stmt.Exec(args ...any) // Выполняет подготовленное выражение (команда).

    res, err := stmt.ExecContext(ctx context.Context, args ...any) // То же с контекстом.

    rows, err := stmt.Query(args ...any) // Выполняет SELECT, вернёт *sql.Rows.

    rows, err := stmt.QueryContext(ctx context.Context, args ...any) // SELECT с контекстом.

    row := stmt.QueryRow(args ...any) // Ожидается 1 строка.

    row := stmt.QueryRowContext(ctx context.Context, args ...any) // То же с контекстом.

    err := stmt.Close() // Закрывает statement и связанные ресурсы в драйвере.
```

### *sql.Rows, \*sql.Row sql.Result - Чтение результатов

`Scan` — метод *sql.Row и *sql.Rows из пакета database/sql, который копирует значения текущей строки результата в переданные получатели (dest ...any). **Количество получателей должно строго совпадать с количеством выбранных колонок**, а порядок — соответствовать порядку колонок в SELECT. **Ошибки многих операций (включая QueryRow*) откладываются до вызова Scan.**

```go
    import "database/sql"

    // *sql.Rows
    cols, err := rows.Columns() // Имена колонок.
    colTypes, err := rows.ColumnTypes() // Метаинформация по типам колонок (драйвер-зависима).
    ok := rows.Next() // Итерируем строки результата; вернёт false на EOF или ошибке.
    err := rows.Err() // Ошибка, произошедшая при итерации (обязательно проверяйте после цикла).
    err := rows.Close() // Освобождает курсор/ресурсы; ОБЯЗАТЕЛЬНО через defer.

    // *sql.Row
    // 1) Кол-во получателей != кол-ву столбцов → ошибка Scan.
    // 2) Пропуск rows.Next() перед rows.Scan() → ошибка: Scan всегда после Next() на *Rows.
    // 3) DDL/ошибки запроса в QueryRow* всплывут только в Row.Scan.
    // 4) RawBytes: использовать после Next() нельзя — буфер станет недействительным.

    // Базовые типы (NOT NULL столбцы)
    var s string
    var n int64
    var f float64
    var b bool
    var t time.Time
    err := rows.Scan(&s, &n, &f, &b, &t) // порядок как в SELECT

    // Обработка NULL #1: sql.Null* (универсально, явный флаг Valid)
    var ns sql.NullString
    var ni sql.NullInt64
    var nt sql.NullTime
    err := rows.Scan(&ns, &ni, &nt)
    if ns.Valid { _ = ns.String } // иначе значение NULL

    // Обработка NULL #2: "указатель на значение"
    var ps *string
    var pn *int64
    var pt *time.Time
    err := rows.Scan(&ps, &pn, &pt)
    // ps == nil при NULL; иначе *ps содержит значение

    // Получить «как есть» от драйвера (без конверсий)
    var raw any
    err := rows.Scan(&raw) // raw будет int64/float64/bool/[]byte/string/time.Time или nil

    // Большие бинарные данные без лишней копии (жить до Next/Scan/Close!)
    var rb sql.RawBytes
    err := rows.Scan(&rb) // см. ограничения RawBytes

    // sql.Result
    id, err := res.LastInsertId() // Возвращает последнюю сгенерированную ID (если драйвер поддерживает).
    n, err := res.RowsAffected() // Возвращает число затронутых строк (если поддерживается).
```
