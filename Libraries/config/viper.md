# spf13/viper

`Viper` — это одна из самых популярных библиотек в Go для работы с конфигурацией.
Она позволяет централизованно управлять настройками приложения, поддерживает разные форматы файлов (YAML, JSON, TOML и др.), переменные окружения, флаги командной строки и даже hot-reload при изменении файла

```sh
    go get github.com/spf13/viper
```

`РЕПОЗИТОРИЙ`: <https://github.com/spf13/viper>

- [spf13/viper](#spf13viper)
  - [Основные особенности Viper](#основные-особенности-viper)
  - [Настройка источников файла](#настройка-источников-файла)
  - [Чтение/Запись конфигурации](#чтениезапись-конфигурации)
  - [Работа со значениями](#работа-со-значениями)
  - [Переменные окружения](#переменные-окружения)
  - [Работа с флагами командной строки в Viper](#работа-с-флагами-командной-строки-в-viper)
  - [Маппинг настроек в структуру](#маппинг-настроек-в-структуру)
    - [Основные теги](#основные-теги)

## Основные особенности Viper

1. `Поддерживаемые форматы файлов`: JSON, TOML, YAML, HCL, envfile, Java properties.
2. `Источники данных`: файлы, переменные окружения, флаги командной строки (обычно через cobra), значения по умолчанию.
3. `Приоритет источников`: значения можно переопределять (например, файл → env → флаг).
4. `Hot-reload`: при изменении конфигурационного файла Viper может автоматически обновлять значения.
5. `Гибкий доступ к данным`: значения можно доставать по ключу (Get, GetInt, GetString и т.д.).
6. `Встроенная поддержка nested-структур`: удобно работать с вложенными настройками

## Настройка источников файла

```go
    // Фиксированный путь

    viper.SetConfigFile("path/config.yaml") // Однозначно указывает, какой файл читать. Полезно, если путь заранее известен или задаётся пользователем. Всегда грузит только этот файл. Если файла нет — ошибка
    viper.ReadInConfig()
    
    // Гибкий поиск по директориям

    viper.SetConfigName("config") // Указываем имя без расширения (config), а Viper сам попробует расширения (.yaml, .json, .toml, …)
    viper.AddConfigPath(".") // Добавить директорию поиска (можно вызывать несколько раз). Это та директория, из которой пользователь запустил приложение
    viper.AddConfigPath("/etc/myapp/") // Системная директория
    viper.ReadInConfig() // Viper найдёт первый доступный файл config.* из перечисленных директорий. Удобно для кроссплатформенных приложений

    // Чтение из строки/stream

    data := []byte(`
        server:
        port: 9000
    `)
    viper.SetConfigType("yaml") // Нужен, если читаем конфиг не из файла, а из io.Reader или строки (например, встроенные ресурсы). Используется, если конфигурация встроена в бинарь или приходит по сети
    viper.ReadConfig(bytes.NewBuffer(data))
```

## Чтение/Запись конфигурации

```go
    // Чтение конфигурации

    err := viper.ReadInConfig(); // Загрузить файл из путей поиска
    err := viper.ReadConfig(io.Reader) // Читать из reader (вместе с SetConfigType)

    // Слияние/подмешивание дополнительных конфигов - это аналог ReadInConfig(), только вместо полной перезаписи настроек он подмешивает (merge) новый конфигурационный файл в уже загруженные данные
    // Поведение при совпадении ключей
    // - Новые значения переопределяют старые (приоритет у более позднего мерджа).
    // - Если ключа не было — он добавляется.
    // - Вложенные структуры объединяются рекурсивно (map-ы сливаются).

    err := viper.MergeInConfig() // Добавить второй файл поверх уже загруженного. Сам файл не указывается, он берётся из текущих параметров поиска
    err := viper.MergeConfig(io.Reader) // То же, но из reader
    err := viper.MergeConfigMap(map[string]any) // Смержить данные из map

    // Запись/создание файла конфигурации

    err := viper.WriteConfig() // Записывает конфигурацию в файл, из которого она была прочитана. Если файл не найден — будет ошибка
    err := WriteConfigAs(filename string) // Записывает конфигурацию в указанный файл. Если файл существует — перезапишет его
    err := viper.SafeWriteConfig() // Запишет конфигурацию в файл, который уже указан (через SetConfigFile/ReadInConfig). Если файл существует — ошибка (не перезаписывает)
    err := viper.SafeWriteConfigAs("initial_config.yaml") // То же, что WriteConfigAs, но никогда не перезаписывает существующий файл
```

## Работа со значениями

```go
    // Значения по умолчанию и присвоение в рантайме

    viper.SetDefault("server.port", 8080) // Дефолт, если ключ не найден нигде
    viper.Set("server.port", 9090) // Принудительно задать значение (самый высокий приоритет)
    viper.Reset() // Очистить все установленные значения и состояние

    // Доступ к значениям (универсальный и типизированные геттеры)

    _ = viper.Get("db") // Универсальный интерфейсный доступ
    _ = viper.GetString("db.url")
    _ = viper.GetBool("feature.enabled")
    _ = viper.GetInt("server.port")
    _ = viper.GetFloat64("ratio")
    _ = viper.GetDuration("timeout") // time.Duration, например "500ms" или "2s"
    _ = viper.GetStringSlice("hosts") // []string
    _ = viper.GetIntSlice("ports") // []int
    _ = viper.GetStringMap("labels") // map[string]any
    _ = viper.GetStringMapString("labels") // map[string]string
    _ = viper.GetStringMapStringSlice("x") // map[string][]string
    _ = viper.GetSizeInBytes("cache.size") // Парсит "512k", "1MB" в байты (int64)

    // Проверка наличия/источника ключа

    _ = viper.IsSet("db.url") // bool — установлен ли ключ где-либо (default/файл/ENV/Set и т.д.)
    _ = viper.InConfig("db.url") // bool — ключ действительно присутствует в текущем файле(ах)

    // Подраздел конфигурации

    sub := viper.Sub("server") // Вернуть *Viper, «сфокусированный» на префиксе
    _ = sub.GetInt("port") // Чтение как будто ключ "port" — это "server.port"

    // Перечень ключей/настроек

    _ = viper.AllSettings() // map[string]any — «сплющенные» настройки для сериализации/логирования
    _ = viper.AllKeys() // []string — список всех ключей (плоский)
```

## Переменные окружения

```go
viper.AutomaticEnv() // Автоматически подхватывать переменные окружения. 
// Любой ключ, который будет искаться через viper.Get("KEY"), 
// сначала проверяется в ENV.

viper.SetEnvPrefix("MYAPP") // Добавить префикс для переменных окружения. 
// Например: viper.Get("db.url") будет искать переменную MYAPP_DB_URL.

viper.BindEnv("db.url") // Привязать конкретный ключ конфига к переменной окружения с таким же именем. 
// Ожидает, что переменная называется DB_URL.

viper.BindEnv("db.url", "DATABASE_URL") // Привязать ключ к конкретной переменной окружения с заданным именем. 
// Теперь viper.Get("db.url") вернёт значение $DATABASE_URL.

viper.SetEnvKeyReplacer(strings.NewReplacer(".", "_")) // Установить замену символов в ключах при поиске в ENV. 
// Пример: "db.url" → "DB_URL". 
// Без этого Viper не сможет автоматически сопоставить ключи с точками.

viper.AllowEmptyEnv(true) // Разрешить использовать пустые переменные окружения. 
// По умолчанию Viper игнорирует ENV, если переменная существует, но её значение пустое.

val := viper.Get("SOME_KEY") // Получить значение переменной окружения напрямую. 
// Работает после AutomaticEnv() или BindEnv().
```

## Работа с флагами командной строки в Viper

Viper умеет интегрироваться с библиотеками pflag и cobra для удобного парсинга аргументов командной строки.
Это позволяет задавать значения конфигурации через флаги (--port=8080), автоматически маппить их в конфиг и объединять с файлами и ENV.

```go
    import (
        "fmt"
        "github.com/spf13/pflag"
        "github.com/spf13/viper"
    )

    err := viper.BindPFlag("server.port", pflag.Lookup("port")) // Привязать конкретный ключ конфигурации к одному флагу.
    // Теперь viper.Get("server.port") будет брать значение из --port (если задано).

    err := viper.BindPFlags(pflag.CommandLine)  // Привязать все флаги из CommandLine (глобальный набор pflag). 
    // Теперь все зарегистрированные флаги будут доступны через viper.

    // Определяем флаги
    pflag.Int("port", 8080, "Server port")
    pflag.String("host", "localhost", "Server host")
    pflag.Parse()

    // Привязываем флаги к Viper
    viper.BindPFlag("server.port", pflag.Lookup("port"))
    viper.BindPFlag("server.host", pflag.Lookup("host"))

    // Теперь можно получать через Viper
    fmt.Println("Host:", viper.GetString("server.host"))
    fmt.Println("Port:", viper.GetInt("server.port"))
```

## Маппинг настроек в структуру

`Маппинг (Unmarshal)` — это преобразование агрегированной конфигурации Viper (файлы, ENV, флаги, SetDefault, Set) в твои Go-структуры.

Ключевой механизм — библиотека mapstructure, которую Viper использует под капотом и которую можно тонко настраивать (decode hooks, squash, remain и т. п.).

`Viper Unmarshal`: <https://pkg.go.dev/github.com/spf13/viper#Unmarshal>
`Mapstructure`: <https://pkg.go.dev/github.com/mitchellh/mapstructure>

**Особенности:**

- `Единая точка правды`: маппинг снимает «срез» со всех источников (defaults → файлы → ENV → флаги → Set) с учётом приоритета.
- `Теги и имена ключей`: для Viper важны теги mapstructure, а не yaml/json. Ключи регистронезависимы, вложенность — через точки (server.port).
- `Типизация и конверсия`: через decode hooks легко парсить time.Duration, time.Time, net.IP, url.URL, []string из строк и т. п.
- `Встраивание и плоская схема`: **,squash** «сплющивает» вложенную структуру в родителя; **,remain** собирает лишние/неизвестные поля в map.
- `Значения по умолчанию:` задавай viper.SetDefault(...) до Unmarshal — в структуру придут уже «докомплектованные» значения.
- `Слабая типизация по умолчанию`: Viper обычно позволяет разумные авто-конверсии (строка "42" → int 42). Можно отключить.

```go
    viper.Unmarshal(&out) // Обычный маппинг
    viper.UnmarshalKey("prefix", &out) // Маппинг только поддерева
    viper.UnmarshalExact(&out) // Строгий маппинг, падает на неизвестных полях
```

### Основные теги

```go
    Name string `mapstructure:"name"` 
    // Переопределение имени ключа. 
    // Без тега поле маппится по имени (без учёта регистра).

    Ignore string `mapstructure:"-"` 
    // Игнорировать поле при маппинге. 
    // В конфиге ключ будет проигнорирован, даже если он есть.

    Extra map[string]any `mapstructure:",remain"` 
    // Собрать в map все «лишние» ключи, 
    // которые не имеют соответствия в структуре.

    Logging `mapstructure:",squash"` 
    // Встроить поля вложенной структуры на уровень выше. «Не ожидай отдельной вложенной секции в конфиге, а расплющи поля вложенной структуры прямо на уровень выше»
    // Без squash: "logging.level" → Logging.Level. В конфиге обязательно должна быть вложенная секция
    // Со squash: "level" → Logging.Level. При указании настройки будет корректно произведен маппинг в структуру, но при этом не нужен вложенный блок logging в настройке
```

**Пример конфигурации:**

```go
    type Config struct {
        App struct {
            Name string `mapstructure:"name"`
            Env  string `mapstructure:"env"`
        } `mapstructure:"app"`

        Server ServerCfg `mapstructure:"server"`
        DB     DBCfg     `mapstructure:"db"`

        Features struct {
            Enabled bool     `mapstructure:"enabled"`
            Flags   []string `mapstructure:"flags"`
        } `mapstructure:"features"`

        Extra map[string]any `mapstructure:",remain"` // соберёт все неизвестные поля
    }

    type ServerCfg struct {
        Host        string        `mapstructure:"host"`
        Port        int           `mapstructure:"port"`
        ReadTimeout time.Duration `mapstructure:"read_timeout"` // "2s", "500ms"
    }

    type DBCfg struct {
        URL string `mapstructure:"url"`
    }
```
