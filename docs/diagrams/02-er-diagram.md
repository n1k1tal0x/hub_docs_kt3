# Диаграмма «сущность — связь» (ER-диаграмма базы данных)

Документ: ФИНУЧЁТ.001-01 · Стадия: эскизный проект
СУБД: SQLite 3.x · ORM: Entity Framework Core 10 · Файл данных: `%LOCALAPPDATA%\FinUchet\finuchet.db`

```mermaid
erDiagram
    ACCOUNTS   ||--o{ OPERATIONS : "списание (AccountId)"
    ACCOUNTS   ||--o{ OPERATIONS : "зачисление (ToAccountId)"
    CATEGORIES ||--o{ OPERATIONS : "классификация"
    CATEGORIES ||--o{ BUDGETS    : "лимит по категории"
    CATEGORIES ||--o{ CATEGORIES : "родитель — подкатегория"

    ACCOUNTS {
        int      Id             PK "Идентификатор"
        string   Name           UK "Наименование, 1..100"
        int      Type              "0 наличные, 1 карта, 2 счёт, 3 вклад"
        string   Currency          "Код валюты по ИСО 4217"
        decimal  InitialBalance    "Начальный остаток"
        string   ColorHex          "Цвет метки счёта"
        bool     IsArchived        "Признак архивности"
        datetime CreatedAt         "Дата создания"
    }

    CATEGORIES {
        int      Id         PK "Идентификатор"
        string   Name          "Наименование"
        int      Kind          "0 доход, 1 расход"
        int      ParentId   FK "Родительская категория (NULL для 1 уровня)"
        string   ColorHex      "Цвет на диаграммах"
        bool     IsPredefined  "Предустановленная категория"
    }

    OPERATIONS {
        int      Id          PK "Идентификатор"
        date     Date           "Дата операции, не позже текущей"
        int      Kind           "0 доход, 1 расход, 2 перевод"
        decimal  Amount         "Сумма, строго больше нуля"
        int      AccountId   FK "Счёт (источник)"
        int      ToAccountId FK "Счёт-получатель, только для перевода"
        int      CategoryId  FK "Категория, NULL для перевода"
        string   Comment        "Комментарий, до 255 символов"
        datetime CreatedAt      "Дата и время регистрации"
    }

    BUDGETS {
        int     Id         PK "Идентификатор"
        int     CategoryId FK "Категория расходов"
        int     Year          "Год"
        int     Month         "Месяц 1..12"
        decimal LimitAmount   "Месячный лимит"
    }

    APPUSER {
        int      Id             PK "Идентификатор (единственная запись)"
        string   PasswordHash      "PBKDF2-HMAC-SHA256"
        string   Salt              "Случайная соль, 16 байт"
        int      Iterations        "Число итераций, не менее 100 000"
        int      FailedAttempts    "Счётчик неверных попыток входа"
        datetime LockedUntil       "Момент окончания блокировки входа"
    }

    SETTINGS {
        string Key   PK "Имя параметра"
        string Value    "Значение параметра"
    }
```

## Ограничения целостности

| Ограничение | Реализация | Требование ТЗ |
|---|---|---|
| Уникальность наименования счёта | Уникальный индекс `Accounts.Name` | ФТ-1 |
| Запрет удаления счёта с операциями | `DeleteBehavior.Restrict` для `Operations.AccountId` | ФТ-1 |
| `Amount > 0` | `CHECK`-ограничение и проверка в слое служб | ФТ-2 |
| `Date <= CURRENT_DATE` | Проверка в слое служб | ФТ-2 |
| `AccountId <> ToAccountId` для перевода | `CHECK`-ограничение и проверка в слое служб | ФТ-2 |
| Иерархия категорий не глубже двух уровней | Проверка в слое служб (`ParentId.ParentId IS NULL`) | ФТ-3 |
| Уникальность лимита на месяц | Уникальный индекс `(CategoryId, Year, Month)` | ФТ-5 |
| Атомарность перевода | Одна транзакция БД на обе стороны перевода | п. 4.2.2 ТЗ |

Текущий остаток по счёту в базе данных не хранится: он вычисляется как
`InitialBalance + Σ(зачисления) − Σ(списания)`, что исключает рассогласование данных.

[← К списку диаграмм](README.md)
