# Capability: api

## Requirements

### Requirement: Интерфейс библиотеки
Система SHALL предоставлять программный API — класс `PhonebookDB`:

| Метод | Сигнатура | Возможные ошибки |
|---|---|---|
| Открыть/создать | `PhonebookDB(path: Path)` | DatabaseLockedError; ValidationError (path — файл, или директория с посторонними файлами) |
| Добавить | `add(name: str, phone: str) -> int` | ValidationError, DuplicateError |
| Телефоны по имени | `find_phones_by_name(name: str, limit: int = 500) -> list[Record]`; префиксный поиск [ADR-9], порядок `(name_key, id)` | — |
| Имена по телефону | `find_names_by_phone(phone: str) -> list[Record]` | — |
| Сменить имя | `update_name(record_id: int, new_name: str) -> None` | ValidationError, DuplicateError, RecordNotFoundError |
| Сменить телефон | `update_phone(record_id: int, new_phone: str) -> None` | ValidationError, DuplicateError, RecordNotFoundError |
| Удалить | `delete(record_id: int) -> None` | RecordNotFoundError |
| Количество записей | `count() -> int` | — |
| Обход всех записей | `iter_all() -> Iterator[Record]`; порядок — по возрастанию id | — |
| Закрыть | `close() -> None`; поддержан контекст-менеджер `with` | — |

Семантика открытия: несуществующая директория создаётся (с родителями);
существующая пустая открывается; path-файл или директория с посторонними
файлами — ValidationError.

`count()` и `iter_all()` — приёмочная поверхность: сверка целостности в
crash-тестах выполняется через них (`specs/performance/`).

#### Scenario: сверка после восстановления
- **GIVEN** N подтверждённых добавлений
- **WHEN** БД открыта после аварийного завершения
- **THEN** `count() == N` и `iter_all()` перечисляет все N записей

### Requirement: Модель ошибок
Система SHALL выбрасывать типизированные исключения, все — подклассы базового
`PhonebaseError`:
- `ValidationError` — ввод не проходит правила `specs/data-model/`;
  некорректный path;
- `DuplicateError` — нарушение уникальности пары [ADR-6], кроме обновления
  записи её же значением;
- `RecordNotFoundError` — операция над несуществующим id;
- `DatabaseLockedError` — директория БД занята живым экземпляром [ADR-5];
- `StorageCorruptedError` — невосстановимое повреждение хранилища
  (`specs/storage/`).

#### Scenario: обновление несуществующей записи
- **GIVEN** в базе 2 записи
- **WHEN** вызывают `update_name(999, "Имя")`
- **THEN** RecordNotFoundError; состояние БД не изменилось

### Requirement: Обработка поисковых запросов
Система SHALL канонизировать поисковый запрос теми же правилами, что и
сохраняемые значения (`specs/data-model/`), но НЕ применять к запросу
валидацию. Семантика поиска — `specs/search/`: префикс по имени (пустой
префикс = обзор с начала, [ADR-9], CUJ-5), точное совпадение по телефону.
Запрос без цифр даёт пустой список; поиск — операция без побочных эффектов.

#### Scenario: невалидный по форме запрос — не ошибка
- **WHEN** ищут имена по телефону `abc`
- **THEN** возвращается пустой список

#### Scenario: пустой префикс имени — обзор
- **WHEN** вызывают `find_phones_by_name("")`
- **THEN** возвращаются первые 500 записей с начала алфавитного порядка

### Requirement: Жизненный цикл
Система SHALL: `close()` идемпотентен (повторный вызов не ошибка); любая
операция после `close()` выбрасывает `PhonebaseError` («база закрыта»);
`close()` выполняет компактификацию при необходимости (`specs/storage/`).

### Requirement: Контракт исполнения
Система SHALL: все методы синхронные; потокобезопасность не гарантируется
(контракт однопоточный) [ADR-5]; валидация и канонизация входа выполняются в
API до передачи в ядро (правило границ — `openspec/project.md`).
