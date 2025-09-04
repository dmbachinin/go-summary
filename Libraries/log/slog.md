# log/slog

`slog` — это новый стандартный пакет для структурированного логирования, появившийся в Go 1.21

Его цель — заменить устаревший log и дать единый, быстрый и расширяемый инструмент для работы с логами без сторонних библиотек.

`ДОКУМЕНТАЦИЯ:` <https://pkg.go.dev/log/slog>

## Особенности

- Структурированное логирование (ключ-значение, JSON, Text, кастомные форматы)
- Уровни логирования (Debug, Info, Warn, Error)
- Контекст: можно добавлять поля глобально или на уровень конкретного логгера
- Handler API: позволяет писать свои обработчики (например, отправлять в Elastic, Loki, Kafka)
- Из коробки поддерживает JSON и текстовый вывод

## Настройка и использование

```go
    // slog.New(handler slog.Handler)
    logger := slog.New(slog.NewTextHandler(os.Stdout, nil)) // Создаёт новый логгер с указанным обработчиком

    opts := &slog.HandlerOptions{
        Level:     slog.LevelDebug, // Минимальный уровень логирования
        AddSource: true, // Добавлять информацию о файле и строке
        ReplaceAttr: func(groups []string, a slog.Attr) slog.Attr { // Функция для изменения атрибутов (например, переименовать ключи или отфильтровать данные)
            if a.Key == slog.TimeKey {
                a.Key = "timestamp"
            }
            return a
        },
    }
    logger := slog.New(slog.NewJSONHandler(os.Stdout, opts))

    // Уровни логирования
    logger.Debug("отладка", "step", 1) // Ключ + значение (key-value) — обычные параметры через запятую.
    logger.Info("запуск", "port", 8080) // Cначала попадает как any, и только потом в рантайме превращается в Attr -> log.Attr{Key:"port", Value:slog.IntValue(8080)}
    logger.Warn("предупреждение", "cache", "low")
    logger.Error("ошибка", "err", err)

    userLogger := logger.With("user", "alex", "session", "12345") // Добавление полей, которые будут автоматически прикрепляться к каждому сообщению

    // Глобальный логгер
    slog.SetDefault(slog.New(slog.NewJSONHandler(os.Stdout, nil))) // Позволяет использовать slog.Info, slog.Warn и т. д. напрямую, без явного logger
    slog.Info("глобальный логгер", "env", "dev")
```

## Работа с атрибутами

`slog.Attr` - это базовый строительный блок структурированного логирования в slog. Он представляет собой пару ключ-значение, которая прикрепляется к записи лога

**Для чего нужен:**

- Чтобы передавать в логгер структурированные данные, а не просто строку.
- Чтобы задавать статичные поля (например, service=auth, env=prod).
- Чтобы удобно создавать поля разных типов (string, int, bool, time, any).
- Для кастомизации логов при помощи ReplaceAttr (например, переименовать ключи)

```go
    type Attr struct {
        Key   string
        Value Value
    }

    slog.String("user", "alex") // Строка
    slog.Int("port", 8080) // int
    slog.Int64("size", 123456) // int64
    slog.Float64("ratio", 0.95) // float64
    slog.Bool("active", true) // bool
    slog.Time("start", time.Now()) // Время
    slog.Duration("latency", 150*time.Millisecond) // Длительность
    slog.Any("data", map[string]any{ // Универсальный вариант
        "id": 123,
        "name": "test",
    })

    logger.Info("Запуск сервиса",
        slog.String("env", "prod"),
        slog.Int("port", 8080),
        slog.Time("started_at", time.Now()),
    )

    logger.Info("регистрация пользователя",
        slog.Group("user", // Пример объединения нескольких полей в группу (у них будет groups = []string{"user"})
            slog.String("name", "alex"),
            slog.String("email", "alex@example.com"),
        ),
    )
    // time=2023-08-28T12:00:00Z level=INFO msg="Запуск сервиса" env=prod port=8080 started_at=2023-08-28T12:00:00Z
```

### ReplaceAttr

`ReplaceAttr` — это колбэк в slog.HandlerOptions, который вызывается для каждого (не-группового) атрибута записи лога непосредственно перед выводом. Он позволяет переименовывать ключи (включая встроенные time, level, msg, source), преобразовывать значения (например, формат времени), редактировать или полностью удалять атрибуты из вывода. Если вернуть «нулевой» slog.Attr{}, атрибут будет отброшен.

Особенности

- `Работает для всех атрибутов, включая встроенные`: time, level, msg, source — их можно переименовать, преобразовать или удалить.
- `Группы не переписываются как единый атрибут`: ReplaceAttr не вызывается для Group целиком — только для её содержимого; при этом вам передаётся срез groups []string с путём текущих групп.
- `Удаление атрибута`: чтобы исключить атрибут из вывода, верните «пустой» slog.Attr{} (это поддерживается встроенными хендлерами TextHandler/JSONHandler).
- `Порядок и производительность`: фильтрация по уровню уже произошла (через Enabled), а ReplaceAttr вызывается внутри форматирующего хендлера для каждого атрибута каждой записи, поэтому колбэк должен быть максимально быстрым и без лишних аллокаций.
- `Ключи по умолчанию и формат источника`: msg — ключ сообщения; source присутствует при AddSource=true и содержит структурированную информацию о файле/строке (в TextHandler выводится как FILE:LINE). Это тоже можно переписать через ReplaceAttr.
- `groups []string` — это срез имён групп, в которые вложен текущий атрибут.Когда вы создаёте лог с **slog.Group**, атрибуты внутри попадают не напрямую в корень записи, а в группу. В ReplaceAttr можно узнать, внутри каких групп находится атрибут, и соответствующим образом его обработать

```go
    opts := &slog.HandlerOptions{
        // Минимальный уровень можно задать отдельно: Level: slog.LevelInfo,
        ReplaceAttr: func(groups []string, a slog.Attr) slog.Attr {
            

            // Пример: убрать timestamp целиком (для тестов или если его проставляет платформа логирования)
            if a.Key == slog.TimeKey {
                return slog.Attr{} // drop
            }

            // Пример: переименовать level → severity и вывести числом
            if a.Key == slog.LevelKey {
                lv := a.Value.Any().(slog.Level)
                a.Key = "severity"
                a.Value = slog.IntValue(int(lv))
                return a
            }

            // Пример: переименовать msg → message
            if a.Key == slog.MessageKey {
                a.Key = "message"
                return a
            }

            // Пример: в группе user.* скрыть email/phone
            if len(groups) > 0 && groups[0] == "user" {
                if a.Key == "email" {
                    a.Value = slog.StringValue(maskEmail(a.Value.String()))
                }
                if a.Key == "phone" {
                    a.Value = slog.StringValue(maskPhone(a.Value.String()))
                }
            }

            return a
        },
    }
    slog.New(slog.NewJSONHandler(w, opts))
```

### Автоматическое форматирование структур в slog через интерфейсы

Чтобы ваши структуры автоматически выводились в нужном формате при логировании, реализуйте интерфейс slog.LogValuer. Он позволяет задать «лог-представление» объекта, которое будет резолвиться перед передачей данных хендлеру (JSON/Text/кастомный). Это работает независимо от выбранного хендлера и даёт полный контроль над структурой/типами полей.

`ДОКУМЕНТАЦИЯ:` <https://pkg.go.dev/log/slog#LogValuer>

```go
    type User struct {
        ID    int
        Name  string
        Email string
        Role  string
    }

    // Реализуем интерфейс slog.LogValuer.
    // Этот метод будет вызван автоматически при slog.Any("user", u) и т.п.
    func (u User) LogValue() slog.Value {
        // Возвращаем строго типизированные атрибуты
        return slog.GroupValue(
            slog.Int("id", u.ID),
            slog.String("name", u.Name),
            slog.String("email", maskEmail(u.Email)), // Маскируем PII: email выводим частично
            slog.String("role", u.Role),
        )
    }
```
