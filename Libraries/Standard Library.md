# Standart Library

`ДОКУМЕНТАЦИЯ`: <https://pkg.go.dev/std>

- [Standart Library](#standart-library)
  - [fmt - реализует форматированный ввод-вывод](#fmt---реализует-форматированный-ввод-вывод)
  - [math - реализует математические функции](#math---реализует-математические-функции)
    - [Константы в math](#константы-в-math)
  - [strconv — преобразование строк и чисел в Go](#strconv--преобразование-строк-и-чисел-в-go)
  - [os](#os)
  - [io](#io)
  - [bufio](#bufio)
  - [time](#time)
  - [Аргументы командной строки](#аргументы-командной-строки)
    - [os.Args](#osargs)
    - [flag](#flag)
  - [Env](#env)
    - [Работа с .env файлами (через godotenv)](#работа-с-env-файлами-через-godotenv)

## fmt - реализует форматированный ввод-вывод

Пакет **fmt** реализует форматированный ввод-вывод с помощью функций, аналогичных printf и scanf в C

`ДОКУМЕНТАЦИЯ`: <https://pkg.go.dev/fmt@go1.24.0#pkg-functions>
`ФОРМАТИРОВАНИЕ СТРОК`: <https://pkg.go.dev/fmt@go1.24.0#hdr-Printing>

```go
    import "fmt"

    // Print выводит аргументы на стандартный вывод.
    fmt.Print("Привет, ", "мир!") 
    // Вывод: Привет, мир!

    // Printf форматирует строку согласно спецификатору и выводит на стандартный вывод.
    name := "Алиса"
    age := 30
    fmt.Printf("Меня зовут %[1]s, мне %[2]d лет.\n", name, age) 
    // [1] - указывает номер переменной, которая будет подставлена
    // Вывод: Меня зовут Алиса, мне 30 лет.

    // Println выводит аргументы на стандартный вывод, добавляя пробелы между ними и завершая новой строкой.
    fmt.Println("Привет,", "мир!")  
    // Вывод: Привет, мир!

    // Fprint выводит аргументы в указанный io.Writer.
    fmt.Fprint(os.Stdout, "Вывод в os.Stdout с помощью Fprint\n")
    // Вывод: Вывод в os.Stdout с помощью Fprint

    // Fprintf форматирует строку и выводит в указанный io.Writer.
    fmt.Fprintf(os.Stdout, "Привет, %s!\n", name)  
    // Вывод: Привет, Алиса!

    // Fprintln выводит аргументы в указанный io.Writer, добавляя пробелы и завершая новой строкой.
    fmt.Fprintln(os.Stdout, "Строка с переводом строки")  
    // Вывод: Строка с переводом строки

    // Sprint возвращает форматированную строку без вывода.
    greeting := fmt.Sprint("Привет, ", name, "!")
    fmt.Println(greeting)
    // Вывод: Привет, Алиса!

    // Sprintf возвращает отформатированную по спецификатору строку.
    formattedString := fmt.Sprintf("Возраст: %d лет", age)
    fmt.Println(formattedString)
    // Вывод: Возраст: 30 лет

    // Sprintln возвращает строку с аргументами, разделенными пробелами, и завершающуюся новой строкой.
    line := fmt.Sprintln("Это", "пример", "строки")
    fmt.Print(line)
    // Вывод: Это пример строки

    // Errorf форматирует строку и возвращает её как ошибку.
    err := fmt.Errorf("ошибка номер %d", 404)
    fmt.Println(err)
    // Вывод: ошибка номер 404

    // Fscan считывает из io.Reader и парсит введенные данные в указанные переменные.
    var input string
    fmt.Fscan(os.Stdin, &input)
    // Ввод: тест
    // Переменная input будет содержать "тест"

    // Fscanf считывает из io.Reader, парсит согласно формату и сохраняет в переменные.
    var number int
    fmt.Fscanf(os.Stdin, "%d", &number)
    // Ввод: 42
    // Переменная number будет содержать 42

    // Fscanln считывает из io.Reader до новой строки и парсит в переменные.
    var word string
    fmt.Fscanln(os.Stdin, &word)
    // Ввод: привет
    // Переменная word будет содержать "привет"

    // Scan считывает из стандартного ввода и парсит в переменные.
    var str string
    fmt.Scan(&str)
    // Ввод: пример
    // Переменная str будет содержать "пример"

    // Scanf считывает из стандартного ввода, парсит по формату и сохраняет в переменные.
    var num int
    fmt.Scanf("%d", &num)
    // Ввод: 25
    // Переменная num будет содержать 25

    // Scanln считывает из стандартного ввода до новой строки и парсит в переменные.
    var lineInput string
    fmt.Scanln(&lineInput)
    // Ввод: строка ввода
    // Переменная lineInput будет содержать "строка ввода"

    // Sscan парсит строку и сохраняет значения в переменные.
    var s1 string
    fmt.Sscan("тестовая строка", &s1)
    fmt.Println(s1)
    // Вывод: тестовая

    // Sscanf парсит строку согласно формату и сохраняет в переменные.
    var v1, v2 int
    fmt.Sscanf("42 56", "%d %d", &v1, &v2)
    fmt.Println(v1, v2)
    // Вывод: 42 56

    // Sscanln парсит строку до новой строки и сохраняет в переменные.
    var w1, w2 string
    fmt.Sscanln("hello world", &w1, &w2)
    fmt.Println(w1, w2)
    // Вывод: hello world
```

## math - реализует математические функции

Пакет **math** предоставляет базовые константы и математические функции.

`ДОКУМЕНТАЦИЯ`: <https://pkg.go.dev/math@go1.24.0>

```go
    import "math"

    // Sqrt возвращает квадратный корень числа.
    math.Sqrt(16) // 4

    // Pow возвращает x, возведенное в степень y.
    math.Pow(2, 3) // 8

    // Abs возвращает абсолютное значение числа.
    math.Abs(-5.5) // 5.5

    // Ceil возвращает наименьшее целое число, не меньшее данного.
    math.Ceil(1.5) // 2

    // Floor возвращает наибольшее целое число, не превышающее данное.
    math.Floor(1.9) // 1

    // Round округляет число до ближайшего целого.
    math.Round(2.7) // 3

    // Max возвращает большее из двух чисел.
    math.Max(1, 2) // 2

    // Min возвращает меньшее из двух чисел.
    math.Min(1, 2) // 1

    // Mod возвращает остаток от деления x на y.
    math.Mod(5, 2) // 1

    // Log возвращает натуральный логарифм числа.
    math.Log(math.E) // 1

    // Log10 возвращает десятичный логарифм числа.
    math.Log10(100) // 2

    // Exp возвращает e, возведенное в степень x.
    math.Exp(1) // 2.718281828459045

    // Sin возвращает синус угла, заданного в радианах.
    math.Sin(math.Pi / 2) // 1

    // Cos возвращает косинус угла, заданного в радианах.
    math.Cos(math.Pi) // -1

    // Tan возвращает тангенс угла, заданного в радианах.
    math.Tan(math.Pi / 4) // 1

    // Asin возвращает арксинус числа в радианах.
    math.Asin(0.5) // 0.5235987755982989

    // Acos возвращает арккосинус числа в радианах.
    math.Acos(0.5) // 1.0471975511965979

    // Atan возвращает арктангенс числа в радианах.
    math.Atan(1) // 0.7853981633974483

    // Hypot возвращает квадратный корень из суммы квадратов двух чисел.
    math.Hypot(3, 4) // 5

    // Copysign возвращает число с величиной первого аргумента и знаком второго.
    math.Copysign(3, -1) // -3

    // Trunc усекает дробную часть числа, оставляя только целую часть.
    math.Trunc(5.9) // 5
```

### Константы в math

```go
    // E представляет основание натурального логарифма.
    const E = 2.718281828459045

    // Pi представляет отношение длины окружности к её диаметру.
    const Pi = 3.141592653589793

    // Phi представляет золотое сечение.
    const Phi = 1.618033988749895

    // Sqrt2 представляет квадратный корень из 2.
    const Sqrt2 = 1.4142135623730951

    // SqrtE представляет квадратный корень из E.
    const SqrtE = 1.6487212707001282

    // SqrtPi представляет квадратный корень из Pi.
    const SqrtPi = 1.7724538509055159

    // SqrtPhi представляет квадратный корень из Phi.
    const SqrtPhi = 1.272019649514069

    // Ln2 представляет натуральный логарифм от 2.
    const Ln2 = 0.6931471805599453

    // Log2E представляет двоичный логарифм от E.
    const Log2E = 1.4426950408889634

    // Ln10 представляет натуральный логарифм от 10.
    const Ln10 = 2.302585092994046

    // Log10E представляет десятичный логарифм от E.
    const Log10E = 0.4342944819032518
    
    const (
        MaxInt    = 1<<(intSize-1) - 1  // MaxInt32 or MaxInt64 depending on intSize.
        MinInt    = -1 << (intSize - 1) // MinInt32 or MinInt64 depending on intSize.
        MaxInt8   = 1<<7 - 1            // 127
        MinInt8   = -1 << 7             // -128
        MaxInt16  = 1<<15 - 1           // 32767
        MinInt16  = -1 << 15            // -32768
        MaxInt32  = 1<<31 - 1           // 2147483647
        MinInt32  = -1 << 31            // -2147483648
        MaxInt64  = 1<<63 - 1           // 9223372036854775807
        MinInt64  = -1 << 63            // -9223372036854775808
        MaxUint   = 1<<intSize - 1      // MaxUint32 or MaxUint64 depending on intSize.
        MaxUint8  = 1<<8 - 1            // 255
        MaxUint16 = 1<<16 - 1           // 65535
        MaxUint32 = 1<<32 - 1           // 4294967295
        MaxUint64 = 1<<64 - 1           // 18446744073709551615
    )
```

## strconv — преобразование строк и чисел в Go

`strconv` — стандартная библиотека Go для преобразования строк в примитивные типы (int, uint, float, bool) и обратно.

`ДОКУМЕНТАЦИЯ`: <https://pkg.go.dev/strconv>

```go
    import "strconv"

    // Atoi преобразует строку в int (base 10).
    strconv.Atoi("42") // 42, nil

    // Itoa преобразует int в строку (base 10).
    strconv.Itoa(42) // "42"

    // ParseInt парсит строку в int64 с указанной системой счисления и разрядностью.
    strconv.ParseInt("2a", 16, 64) // 42, nil

    // ParseUint парсит строку в uint64 с указанной системой счисления и разрядностью.
    strconv.ParseUint("42", 10, 32) // 42, nil

    // ParseFloat парсит строку в float64 или float32.
    strconv.ParseFloat("3.14", 64) // 3.14, nil

    // ParseBool парсит строку в bool ("true", "false", "1", "0").
    strconv.ParseBool("true") // true, nil

    // FormatInt преобразует int64 в строку с указанной системой счисления.
    strconv.FormatInt(42, 16) // "2a"

    // FormatUint преобразует uint64 в строку с указанной системой счисления.
    strconv.FormatUint(42, 10) // "42"

    // FormatFloat преобразует число с плавающей точкой в строку.
    // fmt: 'f' — фиксированный формат, 'e' — экспонента, 'g' — оптимальный.
    // prec — количество знаков после запятой.
    strconv.FormatFloat(3.14159, 'f', 2, 64) // "3.14"

    // FormatBool преобразует bool в строку.
    strconv.FormatBool(true) // "true"

    // AppendInt добавляет строковое представление int64 в срез байт.
    strconv.AppendInt([]byte("num="), 42, 10) // []byte("num=42")

    // AppendUint добавляет строковое представление uint64 в срез байт.
    strconv.AppendUint([]byte{}, 42, 10) // []byte("42")

    // AppendFloat добавляет строковое представление float в срез байт.
    strconv.AppendFloat([]byte{}, 3.14, 'f', 2, 64) // []byte("3.14")

    // AppendBool добавляет строковое представление bool в срез байт.
    strconv.AppendBool([]byte{}, true) // []byte("true")

    // Quote возвращает строку в двойных кавычках с экранированием спецсимволов.
    strconv.Quote("Hello\nGo") // "\"Hello\\nGo\""

    // QuoteToASCII возвращает строку в двойных кавычках с преобразованием не-ASCII символов в \uXXXX.
    strconv.QuoteToASCII("Привет") // "\"\\u041f\\u0440\\u0438\\u0432\\u0435\\u0442\""

    // QuoteToGraphic экранирует только не-графические символы.
    strconv.QuoteToGraphic("Hi\nGo") // "\"Hi\\nGo\""

    // QuoteRune возвращает руну в одинарных кавычках с экранированием.
    strconv.QuoteRune('A') // "'A'"

    // QuoteRuneToASCII возвращает руну с заменой не-ASCII на \uXXXX.
    strconv.QuoteRuneToASCII('Ж') // "'\\u0416'"

    // QuoteRuneToGraphic экранирует только не-графические символы руны.
    strconv.QuoteRuneToGraphic('\n') // "'\\n'"

    // Unquote убирает кавычки и экранирование из строки.
    strconv.Unquote("\"Hello\\nGo\"") // "Hello\nGo", nil

    // UnquoteChar разбирает один символ с экранированием, возвращает руну и остаток строки.
    strconv.UnquoteChar(`\nGo`, 0) // '\n', "Go", nil
```

## os

В Go работа с файловой системой реализуется через стандартный пакет `os`, а также дополнительные пакеты `io`, `bufio`

```go
    import "os"

    // Open открывает файл только для чтения.
    os.Open("file.txt") // *os.File, error

    // Create создает новый файл (перезаписывает, если уже существует).
    os.Create("file.txt") // *os.File, error

    // OpenFile открывает файл с указанными флагами и правами доступа.
    os.OpenFile("file.txt", os.O_RDWR|os.O_CREATE, 0644) // *os.File, error

    // Stat возвращает информацию о файле или директории.
    os.Stat("file.txt") // os.FileInfo, error

    // Lstat аналогична Stat, но не разыменовывает символические ссылки.
    os.Lstat("link.txt") // os.FileInfo, error

    // Remove удаляет файл или пустую директорию.
    os.Remove("file.txt") // error

    // RemoveAll удаляет файл или директорию рекурсивно.
    os.RemoveAll("dir") // error

    // Mkdir создает директорию с указанными правами доступа.
    os.Mkdir("newdir", 0755) // error

    // MkdirAll создает директорию и все недостающие родительские каталоги.
    os.MkdirAll("path/to/dir", 0755) // error

    // ReadDir читает содержимое директории и возвращает список os.DirEntry.
    os.ReadDir("dir") // []os.DirEntry, error

    // Chmod изменяет права доступа к файлу.
    os.Chmod("file.txt", 0644) // error

    // Rename переименовывает или перемещает файл/директорию.
    os.Rename("old.txt", "new.txt") // error

    // Write записывает байты в файл.
    file.Write([]byte("Hello")) // int, error

    // WriteString записывает строку в файл.
    file.WriteString("Hello") // int, error

    // Read читает байты из файла в срез.
    file.Read(buf) // int, error

    // Seek перемещает указатель чтения/записи в файле.
    file.Seek(0, os.SEEK_SET) // int64, error

    // Close закрывает файл.
    file.Close() // error
```

## io

`io` — базовый уровень работы с потоками данных (файлы, сеть, память). Чтение и запись идут напрямую в источник/приемник без промежуточного буфера.

Он определяет интерфейсы (Reader, Writer, Closer, Seeker) и вспомогательные функции для копирования, чтения и записи данных.

- **io.Reader** — источник данных. Читает до len(p) байт в p, возвращает количество считанных байт и ошибку (io.EOF при конце данных)
- **io.Writer** — приемник данных. Записывает байты из p, возвращает количество записанных байт и ошибку
- **io.Closer** — закрытие ресурса
- **io.Seeker** — изменение позиции курсора в потоке

Многие типы из стандартной библиотеки уже реализуют эти интерфейсы:

- **os.File** — Reader, Writer, Seeker, Closer
- **bytes.Buffer** — Reader, Writer
- **strings.Reader** — Reader, Seeker
- **net.Conn** — Reader, Writer, Closer

`ДОКУМЕНТАЦИЯ:` <https://pkg.go.dev/io>

```go
    import "io"

    n, err := io.Copy(dst io.Writer, src io.Reader) // Копирует данные из src в dst, возвращает количество скопированных байт.

    n, err := io.CopyN(dst io.Writer, src io.Reader, count int64) // Копирует ровно count байт из src в dst.

    n, err := io.ReadFull(r io.Reader, buf []byte) // Читает ровно len(buf) байт в buf.

    n, err := io.WriteString(w io.Writer, s string) // Записывает строку s в w.

    n, err := io.ReadAtLeast(r io.Reader, buf []byte, min int) // Читает минимум min байт в buf.

    data, err := io.ReadAll(r io.Reader) // Читает все данные из r в память ([]byte).

    limitReader := io.LimitReader(r io.Reader, n int64) // Возвращает Reader, который читает максимум n байт.

    teeReader := io.TeeReader(r io.Reader, w io.Writer) // Читает из r и одновременно пишет копию в w.

    sectionReader := io.NewSectionReader(r io.ReaderAt, offset int64, length int64) // Reader для части r.

    multiReader := io.MultiReader(r1 io.Reader, r2 io.Reader, ...) // Объединяет несколько Reader в один.

    multiWriter := io.MultiWriter(w1 io.Writer, w2 io.Writer, ...) // Записывает в несколько Writer одновременно.
```

## bufio

`bufio` — надстройка над io, добавляющая буферизацию. Данные копируются в промежуточный буфер в памяти, что уменьшает количество обращений к медленным ресурсам (диск, сеть).

Буферизация ускоряет работу с медленными источниками (диск, сеть) за счет накопления данных в памяти перед фактическим чтением или записью.

`ДОКУМЕНТАЦИЯ:` <https://pkg.go.dev/bufio>

```go
    import "bufio"
    
    reader := bufio.NewReader(r io.Reader) // Создает буферизованный Reader. По умолчанию использует буфер 4096 байт (4 KB)

    readerNewSize := bufio.NewReaderSize(r io.Reader, 8192) // Создаем bufio.Reader с буфером на 8 KB

    line, err := reader.ReadString(delim byte) // Читает строку до символа-разделителя (включая его).

    bytes, err := reader.ReadBytes(delim byte) // Читает байты до символа-разделителя.

    b, err := reader.Peek(n int) // Возвращает следующие n байт без перемещения курсора.

    n, err := reader.Read(buf []byte) // Читает данные в buf.

    writer := bufio.NewWriter(w io.Writer) // Создает буферизованный Writer.

    n, err := writer.WriteString(s string) // Записывает строку в буфер.

    err := writer.Flush() // Сбрасывает содержимое буфера в w.

    scanner := bufio.NewScanner(r io.Reader) // Создает Scanner для построчного чтения.

    ok := scanner.Scan() // Считывает следующую строку (true при успешном чтении).

    text := scanner.Text() // Возвращает текст последней строки.

    err := scanner.Err() // Ошибка, возникшая при сканировании.
```

## time

Для работы с датой и временем в Go используется пакет `time`

`ДОКУМЕНТАЦИЯ:` <https://pkg.go.dev/time>

Go не использует буквенные паттерны (dd, MM, yyyy), как Java.
Вместо этого он смотрит на "магическую дату" 2006-01-02 15:04:05 по которой и строится шаблон парсинга

**Это полный вариант эталонной даты, включающий:**

- 2006 — год
- 01 — месяц
- 02 — день
- 15 — часы (24ч формат)
- 04 — минуты
- 05 — секунды
- -0700 — смещение часового пояса

```go
    import "time"
    
    // Парсинг с учетом часового пояса
    layout := "2006-01-02 15:04:05 -0700"
    t, err = time.Parse(layout, "2025-08-18 14:30:00 +0500")

    // Формат: ГГГГ-ММ-ДД
    t, _ := time.Parse("2006-01-02", "2025-08-18")

    // Формат: ДД.ММ.ГГГГ
    t, _ = time.Parse("02.01.2006", "18.08.2025")

    // Формат: ГГГГ/ММ/ДД ЧЧ:ММ:СС
    t, _ = time.Parse("2006/01/02 15:04:05", "2025/08/18 14:35:59")

    // Формат с часовым поясом
    t, _ = time.Parse("2006-01-02 15:04:05 -0700", "2025-08-18 14:35:59 +0500")

    // Форматирование даты в строку
    now := time.Now()
    formatted := now.Format("02.01.2006 15:04:05") // "18.08.2025 14:30:00"

    // Добавление 24 часов
    tomorrow := now.Add(24 * time.Hour)

    // Разница между датами
    diff := tomorrow.Sub(now) // 24h0m0s
```

## Аргументы командной строки

### os.Args

`os.Args` — массив строк с аргументами запуска

***Особенности:***

- тип []string
- os.Args[0] — всегда путь к исполняемому файлу
- остальные элементы — переданные параметры

```go
    import (
        "fmt"
        "os"
    )
    // Пример команды: go run main.go hello world 123
    // Чтение аргументов напрямую
    args := os.Args
    fmt.Println(args[0]) // путь к бинарнику
    fmt.Println(args[1]) // первый параметр
    fmt.Println(args[2]) // второй параметр
```

### flag

Пакет `flag` — более удобный инструмент для именованных флагов (ключ-значение)

`ДОКУМЕНТАЦИЯ:` <https://pkg.go.dev/flag>

***Особенности:***

- поддерживает именованные параметры (--port=8080, -mode=dev)
- умеет автоматически парсить в разные типы (int, string, bool)
- можно задавать значения по умолчанию
- если флаг задан с неправильным форматом, то программа ляжет с ошибкой 2. Этого можно избежать, если использовать flag.NewFlagSet

```go
    import "flag"

    // Флаг типа bool
    debug := flag.Bool("debug", false, "включить режим отладки")
    // при запуске: go run main.go --debug=true

    // Флаг типа string
    mode := flag.String("mode", "dev", "режим запуска (dev/prod)")
    // при запуске: go run main.go --mode=prod

    // Флаг типа int
    port := flag.Int("port", 8080, "порт сервера")
    // при запуске: go run main.go --port=9000

    // Флаг типа float64
    ratio := flag.Float64("ratio", 0.75, "коэффициент загрузки")
    // при запуске: go run main.go --ratio=0.95

    // Флаг типа time.Duration
    delay := flag.Duration("delay", 5*time.Second, "задержка запуска")
    // при запуске: go run main.go --delay=10s

    // ПАРСИНГ И ДОСТУП К ЗНАЧЕНИЯМ

    flag.Parse() // обязательно вызвать, чтобы распарсить все флаги

    fmt.Println(*debug)   // доступ к значению через разыменование
    fmt.Println(*mode)

    // РАБОТА С НЕФЛАГОВЫМИ АРГУМЕНТАМИ

    args := flag.Args()   // возвращает срез непроанализированных аргументов
    first := flag.Arg(0)  // возвращает первый аргумент после флагов
    count := flag.NArg()  // количество непроанализированных аргументов
    used := flag.NFlag()  // количество успешно установленных флагов

    // ОБХОД ВСЕХ ФЛАГОВ

    // Visit — обходит только флаги, которые реально были заданы
    flag.Visit(func(f *flag.Flag) {
        fmt.Println("Установлен:", f.Name, "=", f.Value)
    })

    // VisitAll — обходит все флаги, включая дефолтные
    flag.VisitAll(func(f *flag.Flag) {
        fmt.Println("Флаг:", f.Name, "=", f.Value)
    })

    // СОЗДАНИЕ СОБСТВЕННОГО НАБОРА ФЛАГОВ

    fs := flag.NewFlagSet("MyApp", flag.ExitOnError) // создаем отдельный набор
    customPort := fs.Int("port", 8080, "порт приложения")
    fs.Parse([]string{"--port=5555"}) // разбираем кастомные аргументы
    fmt.Println(*customPort) // 5555
```

## Env

`Переменные окружения (environment variables)` — это способ передачи конфигурации приложению из операционной системы или внешней среды (docker, kubernetes, CI/CD и т.д.). В Go работа с ними реализована через стандартный пакет os и сторонние библиотеки для удобного управления.

```go
    import "os"

// Получение значения переменной окружения
val := os.Getenv("APP_PORT") // если не задана, будет пустая строка

v, ok := os.LookupEnv("DB_USER");  // Проверка существования переменной

// Установка переменной окружения
_ = os.Setenv("MODE", "production")
fmt.Println(os.Getenv("MODE")) // "production"

// Удаление переменной окружения
_ = os.Unsetenv("MODE")

result := os.ExpandEnv("User=$USER, Home=${HOME}, Path=$PATH") // Это функция проходит по строке и заменяет шаблоны переменных окружения на их реальные значения
// Если заданы переменные окружения, то будет результирующая строка: "User=alex, Home=/home/alex, Path=/usr/bin:/bin:/usr/local/bin"

// Получение всех переменных окружения
for _, e := range os.Environ() {
    fmt.Println(e) // формат "KEY=VALUE"
}
```

### Работа с .env файлами (через godotenv)

`godotenv` — популярная библиотека для Go, которая загружает переменные окружения из .env файла в окружение процесса.
Это удобно для локальной разработки: вместо того чтобы хранить секреты или настройки в коде, мы кладем их в .env и используем через os.Getenv.

`ИСХОДНИКИ`: <https://github.com/joho/godotenv>

**Пример установки:**

```sh
    go get github.com/joho/godotenv
```

```go
    import "github.com/joho/godotenv"

    err := godotenv.Load() // Ищет .env в текущей директории
    _ = godotenv.Load("config/dev.env") // Загрузка конкретного .env
    _ = godotenv.Overload(".env") // Даже если переменная уже установлена в системе, перезапишет её значением из файла
    envMap, err := godotenv.Read(".env") // Чтение в map без применения

    port := os.Getenv("APP_PORT") // Пример получения переменной из .env
```
