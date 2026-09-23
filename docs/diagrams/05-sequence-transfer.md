# Диаграмма последовательности: регистрация перевода между счетами

Документ: ФИНУЧЁТ.001-01 · Стадия: технический проект
Реализуемое требование: ФТ-2 (регистрация операций), п. 4.2.2 ТЗ (транзакционность)

```mermaid
sequenceDiagram
    autonumber
    actor U as Пользователь
    participant V as OperationDialog<br/>(View)
    participant VM as OperationViewModel
    participant OS as OperationService
    participant VAL as OperationValidator
    participant DB as FinUchetDbContext
    participant SQL as SQLite
    participant LOG as Serilog

    U->>V: Нажимает «Перевод»
    V->>VM: LoadAsync(Kind = Transfer)
    VM->>OS: GetActiveAccountsAsync()
    OS->>DB: Accounts.Where(!IsArchived)
    DB->>SQL: SELECT * FROM Accounts
    SQL-->>DB: строки
    DB-->>OS: коллекция счетов
    OS-->>VM: список счетов
    VM-->>V: заполняет списки выбора

    U->>V: Заполняет дату, сумму, счета, комментарий
    U->>V: Нажимает «Сохранить»
    V->>VM: SaveCommand.Execute()
    VM->>OS: AddTransferAsync(operation)

    OS->>VAL: Validate(operation)
    alt Данные некорректны
        VAL-->>OS: Errors (сумма ≤ 0 / дата в будущем / счета совпадают)
        OS-->>VM: Result.Fail(сообщения)
        VM-->>V: подсветка полей и текст ошибки
        V-->>U: «Сумма операции должна быть больше нуля»
    else Данные корректны
        VAL-->>OS: OK
        OS->>DB: BeginTransactionAsync()
        DB->>SQL: BEGIN TRANSACTION
        OS->>DB: Operations.Add(operation)
        DB->>SQL: INSERT INTO Operations
        alt Ошибка записи
            SQL-->>DB: exception
            DB->>SQL: ROLLBACK
            OS->>LOG: Error(исключение, стек)
            OS-->>VM: Result.Fail("Непредвиденная ошибка…")
            VM-->>V: окно сообщения об ошибке
        else Запись выполнена
            SQL-->>DB: OK
            DB->>SQL: COMMIT
            OS->>LOG: Information("Сохранена операция N …")
            OS-->>VM: Result.Ok()
            VM->>OS: GetBalanceAsync(из счёта), GetBalanceAsync(в счёт)
            OS->>DB: пересчёт остатков
            DB->>SQL: SELECT SUM(...)
            SQL-->>DB: остатки
            DB-->>OS: остатки
            OS-->>VM: новые остатки
            VM-->>V: закрывает форму, обновляет журнал и «Обзор»
            V-->>U: операция в списке, остатки пересчитаны
        end
    end
```

## Ключевые свойства сценария

1. **Атомарность.** Перевод изменяет остатки двух счетов; запись выполняется в одной
   транзакции. При сбое посередине транзакция откатывается полностью, операция в базе
   не появляется, остатки обоих счетов остаются прежними.
2. **Контроль до записи.** Проверка данных выполняется в слое служб до открытия транзакции,
   поэтому ошибочный ввод не приводит к обращению к базе данных.
3. **Остатки не хранятся.** После фиксации транзакции остатки вычисляются запросом
   к таблице операций, а не обновляются отдельным полем, — рассогласование исключено.
4. **Журналирование.** Штатные события записываются с уровнем `INF`, ошибки — с уровнем `ERR`
   вместе с текстом исключения и стеком вызовов (п. 4.2.3 ТЗ).

[← К списку диаграмм](README.md)
