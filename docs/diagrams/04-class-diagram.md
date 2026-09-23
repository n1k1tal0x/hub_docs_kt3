# Диаграмма классов моделей и служб

Документ: ФИНУЧЁТ.001-01 · Стадия: технический проект

```mermaid
classDiagram
    direction LR

    class Account {
        +int Id
        +string Name
        +AccountType Type
        +string Currency
        +decimal InitialBalance
        +string ColorHex
        +bool IsArchived
        +ICollection~Operation~ Operations
    }

    class Category {
        +int Id
        +string Name
        +CategoryKind Kind
        +int? ParentId
        +Category Parent
        +string ColorHex
        +bool IsPredefined
        +bool IsRoot()
    }

    class Operation {
        +int Id
        +DateOnly Date
        +OperationKind Kind
        +decimal Amount
        +int AccountId
        +int? ToAccountId
        +int? CategoryId
        +string Comment
        +decimal SignedAmount(int accountId)
    }

    class Budget {
        +int Id
        +int CategoryId
        +int Year
        +int Month
        +decimal LimitAmount
        +decimal Spent
        +double UsagePercent()
        +BudgetState State()
    }

    class AccountType {
        <<enumeration>>
        Cash
        Card
        BankAccount
        Deposit
    }
    class CategoryKind {
        <<enumeration>>
        Income
        Expense
    }
    class OperationKind {
        <<enumeration>>
        Income
        Expense
        Transfer
    }
    class BudgetState {
        <<enumeration>>
        Normal
        Warning
        Exceeded
    }

    class IAccountService {
        <<interface>>
        +GetAllAsync() Task~IReadOnlyList~Account~~
        +GetBalanceAsync(int id) Task~decimal~
        +CreateAsync(Account a) Task~Result~
        +UpdateAsync(Account a) Task~Result~
        +DeleteAsync(int id) Task~Result~
        +ArchiveAsync(int id) Task~Result~
    }

    class IOperationService {
        <<interface>>
        +FindAsync(OperationFilter f, int page) Task~PagedResult~Operation~~
        +AddAsync(Operation o) Task~Result~
        +UpdateAsync(Operation o) Task~Result~
        +DeleteAsync(int id) Task~Result~
        +AddTransferAsync(Operation o) Task~Result~
    }

    class IBudgetService {
        <<interface>>
        +GetMonthAsync(int y, int m) Task~IReadOnlyList~Budget~~
        +SetLimitAsync(int categoryId, int y, int m, decimal limit) Task~Result~
        +CheckAfterOperationAsync(Operation o) Task~BudgetState~
    }

    class IReportService {
        <<interface>>
        +IncomeExpenseAsync(DateOnly from, DateOnly to) Task~PeriodReport~
        +StructureAsync(DateOnly from, DateOnly to) Task~StructureReport~
        +DynamicsAsync(DateOnly from, DateOnly to) Task~DynamicsReport~
    }

    class ICsvService {
        <<interface>>
        +ExportAsync(IEnumerable~Operation~ ops, string path) Task~Result~
        +ImportAsync(string path) Task~ImportLog~
    }

    class IBackupService {
        <<interface>>
        +CreateAsync(string path) Task~Result~
        +RestoreAsync(string path) Task~Result~
        +CleanupAsync(int keep)  Task
    }

    class IAuthService {
        <<interface>>
        +HasPassword() bool
        +SetPasswordAsync(string pwd) Task~Result~
        +SignInAsync(string pwd) Task~SignInResult~
        +ChangePasswordAsync(string old, string neu) Task~Result~
    }

    class Result {
        +bool IsSuccess
        +string Message
        +IReadOnlyList~string~ Errors
    }

    class ImportLog {
        +int Processed
        +int Imported
        +int Rejected
        +IReadOnlyList~ImportLine~ Lines
    }

    Account "1" --> "0..*" Operation : списание
    Account "0..1" --> "0..*" Operation : зачисление
    Category "1" --> "0..*" Operation
    Category "1" --> "0..*" Budget
    Category "0..1" --> "0..*" Category : подкатегории
    Account ..> AccountType
    Category ..> CategoryKind
    Operation ..> OperationKind
    Budget ..> BudgetState

    IAccountService ..> Account
    IOperationService ..> Operation
    IBudgetService ..> Budget
    IReportService ..> Operation
    ICsvService ..> ImportLog
    IAccountService ..> Result
    IOperationService ..> Result
```

## Примечания

- Все службы возвращают объект `Result` (или его обобщённую форму), содержащий признак успеха
  и перечень сообщений для пользователя, — это позволяет выводить тексты из раздела 4
  руководства пользователя без дублирования логики в слое представления.
- `Budget.State()` возвращает `Normal` (< 80 %), `Warning` (80…100 %) или `Exceeded` (> 100 %),
  что соответствует цветовой индикации, описанной в ФТ-5.
- Перечисления (`enum`) хранятся в базе данных как целые числа.

[← К списку диаграмм](README.md)
