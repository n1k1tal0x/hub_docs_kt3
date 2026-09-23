# Диаграмма архитектуры приложения

Документ: ФИНУЧЁТ.001-01 · Стадия: технический проект
Шаблон: MVVM (Model — View — ViewModel) · Требование ТЗ: п. 4.5.3

```mermaid
flowchart TB
    subgraph V["Слой представления — FinUchet.Wpf"]
        direction LR
        V1["Views<br/>XAML-окна и страницы"]
        V2["ViewModels<br/>CommunityToolkit.Mvvm"]
        V3["Converters, Styles,<br/>Resources (ru-RU)"]
        V1 <-->|"Binding, Commands"| V2
        V1 --- V3
    end

    subgraph S["Слой служб (бизнес-логика) — FinUchet.Core"]
        direction LR
        S1["AccountService"]
        S2["OperationService"]
        S3["CategoryService"]
        S4["BudgetService"]
        S5["ReportService"]
        S6["CsvService"]
        S7["BackupService"]
        S8["AuthService"]
        S9["Validation<br/>(правила ФТ-1 — ФТ-10)"]
    end

    subgraph D["Слой доступа к данным — FinUchet.Data"]
        direction LR
        D1["FinUchetDbContext<br/>EF Core 10"]
        D2["Migrations"]
        D3["Entities<br/>Account, Category,<br/>Operation, Budget"]
    end

    subgraph I["Инфраструктура"]
        direction LR
        I1["DI-контейнер<br/>Microsoft.Extensions.DependencyInjection"]
        I2["Конфигурация<br/>appsettings.json"]
        I3["Журналирование<br/>Serilog + Sinks.File"]
        I4["Глобальный обработчик<br/>исключений"]
    end

    DB[("SQLite<br/>finuchet.db")]
    FS[["CSV-файлы,<br/>резервные копии,<br/>журналы"]]

    V2 -->|"интерфейсы служб"| S
    S --> D1
    D1 --> DB
    D1 --- D2
    D1 --- D3
    S6 --> FS
    S7 --> FS
    I3 --> FS
    I1 -.->|"внедрение зависимостей"| V2
    I1 -.-> S
    I1 -.-> D1
    I2 -.-> I1
    I4 -.-> V1
    I4 -.-> I3
```

## Состав проектов решения

| Проект | Тип | Назначение |
|---|---|---|
| `FinUchet.Wpf` | WPF (net10.0-windows) | Окна, страницы, модели представления, ресурсы интерфейса |
| `FinUchet.Core` | Class library (net10.0) | Модели предметной области, службы бизнес-логики, правила контроля данных |
| `FinUchet.Data` | Class library (net10.0) | `DbContext`, конфигурации сущностей, миграции схемы БД |
| `FinUchet.Tests` | xUnit (net10.0) | Модульные и интеграционные тесты, покрытие бизнес-логики ≥ 70 % |

## Правила взаимодействия слоёв

- Слой представления не обращается к `DbContext` напрямую — только через интерфейсы служб.
- Службы не зависят от WPF и от типов `System.Windows`, что делает их пригодными для модульного тестирования.
- Все операции, затрагивающие более одной таблицы (в том числе перевод между счетами),
  выполняются в одной транзакции базы данных (п. 4.2.2 ТЗ).
- Необработанные исключения перехватываются глобальным обработчиком: пользователю выводится
  сообщение на русском языке, полные сведения записываются в журнал (п. 4.2.3 ТЗ).

[← К списку диаграмм](README.md)
