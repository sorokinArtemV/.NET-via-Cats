## Exceptions / Ошибки как механизм и как значение

> **Mental model №1 (исключения).** Исключение — это принудительный, неигнорируемый канал для _неожиданного_. Цена за «нельзя проигнорировать» — дорогой бросок и раскрутка стека, поэтому исключения не для control flow.
> 
> **Mental model №2 (Result).** Когда сбой _ожидаем_ (не нашли юзера, провалилась валидация), ошибку возвращают как обычное значение — и компилятор не даёт её проигнорировать, потому что она в типе.

---

### Часть A — Исключения как механизм

#### A1. Зачем исключения вообще

До исключений ошибки возвращали кодами: функция отдаёт `-1` / `null` / `false`, вызывающий обязан проверить.

```c
int rc = parse(input);   // -1 при ошибке
// забыл проверить rc — поехал дальше с мусором
```

Три фундаментальные проблемы этого подхода:

1. **Ошибку легко молча проигнорировать** — компилятор не заставляет смотреть на return value.
2. **Канал ошибки забивает полезный канал.** Если метод возвращает `User`, то «не нашли» приходится кодировать тем же типом (`null`) — теряется разница между «нет юзера» и «нет данных вообще».
3. **Ошибку нельзя дёшево пробросить через слои.** Каждый промежуточный метод обязан проверить код и вручную пробросить выше. Пять слоёв — пять проверок, любая из которых может забыться.

Исключения решают это радикально: ошибка идёт по **отдельному, обязательному к обработке каналу**, который сам раскручивается вверх по стеку, пока кто-то его не поймает. Промежуточные слои о нём знать не обязаны. Цена — канал дорогой (см. A6), поэтому правило: исключения для _исключительного_, не для ожидаемого. Именно эта граница породит Result-паттерн в Части B.

#### A2. try / catch / finally и иерархия


```csharp
try { /* код, который может бросить */ }
catch (InvalidOperationException ex) { /* конкретный тип — ловим узко */ }
catch (Exception ex) { /* широкий перехват — обычно только на границе приложения */ }
finally { /* выполнится при любом исходе — обычно cleanup */ }
```

Базовое правило: **ловить максимально узкий тип**, который реально умеешь обработать. `catch (Exception)` без переброса на середине стека — это проглатывание ошибок, классический источник «невидимых» багов на проде.

Иерархия (сокращённо):

```
System.Object
└── System.Exception
    ├── SystemException        ← бросает рантайм
    │   ├── NullReferenceException
    │   ├── IndexOutOfRangeException
    │   ├── ArgumentException → ArgumentNullException, ArgumentOutOfRangeException
    │   └── InvalidOperationException
    └── ApplicationException    ← историческая ветка, см. ниже
```

> **🔬 Глубже / на синьора.** Деление `SystemException` / `ApplicationException` — **исторический мусор**. В .NET Framework 1.0 задумывалось: рантайм бросает `SystemException`, прикладной код наследует свои от `ApplicationException`. Это правило **отменено** (Framework Design Guidelines): своё исключение наследуют напрямую от `Exception`. Предложение на собесе наследоваться от `ApplicationException` — маркер устаревших знаний.

#### A3. throw vs throw ex vs ExceptionDispatchInfo

```csharp
catch (Exception ex)
{
    throw;       // ✅ сохраняет оригинальный stack trace
    // throw ex; // ❌ затирает trace — точка броска теперь ЗДЕСЬ
}
```

**Механизм.** Stack trace хранится внутри объекта исключения. `throw;` (IL-инструкция `rethrow`) повторно поднимает то же исключение, не трогая сохранённый trace. `throw ex;` (IL `throw`) рантайм трактует как **новый бросок из текущей точки**: trace перезаписывается, оригинальная строка падения теряется. На проде это превращает trace в ложь, указывающую на catch вместо настоящего места бага.

> **🔬 Глубже / на синьора.** Третий случай — пробросить **захваченное** исключение (сохранили в поле/переменную, бросаем позже, после `await`, или из другого метода). Там `throw;` недоступен (мы вне catch), а `throw ex;` затрёт trace. Решение — `ExceptionDispatchInfo`:
> 
> csharp
> 
> ```csharp
> ExceptionDispatchInfo? captured = null;
> try { Risky(); }
> catch (Exception ex) { captured = ExceptionDispatchInfo.Capture(ex); }
> // ... позже, возможно в другом контексте ...
> captured?.Throw();   // оригинальный trace сохранён и дополнен
> ```
> 
> Именно так `await` пробрасывает исключение из `Task`, не теряя исходный стек.

#### A4. Exception filters — catch when

```csharp
catch (HttpRequestException ex) when (ex.StatusCode == HttpStatusCode.TooManyRequests)
{
    await RetryWithBackoff();
}
```

Фильтр `when` ловит исключение **только если условие истинно**; иначе оно летит дальше, как будто этого catch нет. Зачем, если есть `catch`-по-типу: фильтр смотрит на **свойства** (статус-код, error number), а не только на тип. Второй приём — логирование без перехвата:


```csharp
catch (Exception ex) when (Log(ex)) { }   // Log всегда возвращает false
```

`Log` отрабатывает, возвращает `false`, исключение продолжает лететь — залогировали, ничего не «съев».

> **🔬 Глубже / на синьора.** Ключевой механизм: **фильтр `when` выполняется ДО раскрутки стека** — на первом проходе обработки (см. A5). В момент работы фильтра стек ещё цел, в дебаггере видна реальная точка броска. Отсюда неинтуитивный порядок: `when` верхнего кадра отрабатывает _раньше_, чем `finally` нижнего.

#### A5. Двухпроходная модель и раскрутка стека

CLR обрабатывает исключение в **два прохода** (модель SEH-типа, на CoreCLR кросс-платформенная):

- **Pass 1 — search/dispatch.** Рантайм идёт вверх по стеку от точки броска, ища `catch` подходящего типа, у которого `when`-фильтр (если есть) вернул `true`. **Стек НЕ раскручивается** — кадры живы, фильтры выполняются здесь.
- **Pass 2 — unwind.** Обработчик найден. Стек раскручивается от точки броска до найденного `catch`, на каждом покидаемом кадре выполняются его `finally`. Затем управление входит в `catch`.

Это объясняет тайминг: `when` (Pass 1) всегда раньше `finally` (Pass 2), и отсюда же берётся цена — два прохода по стеку.

![[Pasted image 20260624112430.png]]

#### A6. Цена исключений + миф «try/catch замедляет код»

**Почему `throw` дорог.** При броске рантайм: захватывает stack trace (проход по кадрам), делает Pass 1 (поиск обработчика), делает Pass 2 (раскрутка + `finally`). В сумме это **на порядки дороже обычного `return`** — микросекунды против наносекунд. Поэтому исключение на _ожидаемой_ ветке (валидация в цикле на миллион записей) — катастрофа по throughput.

Из этого — правило **не использовать исключения как control flow**: не «бросил-поймал, чтобы выйти из цикла», не парсинг через try/catch. Управление потоком — это `if`, `return`, `Result`; исключение — для того, что _не должно_ происходить в норме.

> **🔬 Глубже / на синьора. Миф: «сам `try/catch` замедляет код».** Нет. На современном JIT наличие `try/catch` на happy path (когда исключение **не** бросается) практически бесплатно: EH-регионы не добавляют инструкций на нормальном пути. Платишь только в момент реального `throw`. Оговорка: try-регион может слегка ограничить отдельные оптимизации JIT (кэширование в регистрах вокруг региона) — но это микроэффект, не «замедление». Вывод: бойся `throw` в горячем пути, а не `try`.

#### A7. Когда finally НЕ выполнится

«`finally` выполняется всегда» — верно с оговорками. `finally` **не** выполнится при:

- `StackOverflowException` — неперехватываемо, процесс падает немедленно;
- `Environment.FailFast(...)` — намеренное мгновенное завершение без раскрутки;
- жёстком убийстве процесса извне (`TerminateProcess`, `kill -9`);
- зависании самого блока `try` (бесконечный цикл — до `finally` управление не дойдёт).

Во всех штатных исходах (включая `return` из `try` и проброс исключения) — выполнится.

csharp

```csharp
try { return ComputeA(); }   // значение посчитано...
finally { Cleanup(); }       // ...но Cleanup отработает ДО фактического возврата
```

> **🔬 Глубже / на синьора.** `finally` может перекрыть результат, если внутри него есть свой `return` — тогда возвращается значение из `finally` (C# даёт warning, так делать не стоит). `await` внутри `finally` валиден — это часть async state machine, детали → глава **async/await**.

#### A8. Custom exceptions

csharp

```csharp
public class PizzaBurnedException : Exception
{
    public PizzaBurnedException() { }
    public PizzaBurnedException(string message) : base(message) { }
    public PizzaBurnedException(string message, Exception inner) : base(message, inner) { }
}
```

Правила: суффикс `Exception`, наследование от `Exception` (не от `ApplicationException`), три стандартных конструктора (пустой, с сообщением, с `inner` для оборачивания). Заводить свой тип стоит, только если вызывающий код может **по-разному реагировать** на него — иначе хватит встроенного (`InvalidOperationException`, `ArgumentException`).

> **🔬 Глубже / на синьора. Сериализационный конструктор устарел.** Старый «обязательный» паттерн — `[Serializable]` + `protected MyException(SerializationInfo, StreamingContext)` + переопределение `GetObjectData` — с **.NET 8** даёт warning **`SYSLIB0051`**: он завязан на `BinaryFormatter`, который с .NET 8 признан небезопасным и помечен obsolete-as-error. Исторически эта сериализация существовала ради remoting, убранного ещё в .NET Core 1.0. Для нового кастомного исключения **просто не добавляй** `[Serializable]`, сериализационный ctor и `GetObjectData` — хватает трёх обычных конструкторов. (Забавный конфликт: старый анализатор `CA2229` раньше _требовал_ этот ctor — теперь спорит с `SYSLIB0051`.)

#### A9. Unhandled и async-исключения

**Unhandled exception.** Если исключение долетело до верха стека и его никто не поймал — оно _необработанное_. С .NET 2.0+ необработанное исключение **на любом потоке роняет процесс** (раньше рантайм часть проглатывал). Точки наблюдения: `AppDomain.CurrentDomain.UnhandledException` (последний шанс залогировать перед смертью), `AppDomain.CurrentDomain.FirstChanceException` (срабатывает на _каждом_ броске, ещё до поиска обработчика — для диагностики).

> **🔬 Глубже / на синьора. async-исключения (кратко; детали → глава async/await).**
> 
> - Исключение в `async Task`-методе **не летит сразу** — оно «кладётся» в возвращаемый `Task` и перебрасывается в точке `await` (через `ExceptionDispatchInfo`, поэтому стек сохраняется).
> - `async void` — исключению **некуда деться**: оно перебрасывается на `SynchronizationContext` и при отсутствии обработчика **роняет процесс**. Отсюда правило «`async void` только для event handlers».
> - Не дождались `Task` с исключением → unobserved exception (`TaskScheduler.UnobservedTaskException`).

---

### Часть B — Ошибка как значение (Result pattern)

#### B1. Зачем (мост из A6)

Большинство «ошибок» в бизнес-логике **не исключительны**: юзер не найден, email невалиден, недостаточно средств — это нормальные, ожидаемые исходы. Городить на них исключения — дорого (A6) и семантически неверно (исключение кричит «случилось непредвиденное», а тут всё предвиденно). Альтернатива: вернуть ошибку как **обычное значение** — объект `Result`, явно говорящий «успех или провал». Контраст с исключениями резкий: ошибка не прыгает мимо слоёв, а проходит сквозь них и **видна в сигнатуре**, поэтому компилятор не даёт её молча потерять.

#### B2. Result / Result<T> / Error

csharp

```csharp
public enum ErrorType { Failure, Validation, NotFound, Conflict, Problem }

public sealed record Error
{
    public static readonly Error None = new(string.Empty, string.Empty, ErrorType.Failure);

    private Error(string code, string description, ErrorType type)
        => (Code, Description, Type) = (code, description, type);

    public string Code { get; }
    public string Description { get; }
    public ErrorType Type { get; }

    public static Error Failure(string code, string desc)    => new(code, desc, ErrorType.Failure);
    public static Error Validation(string code, string desc) => new(code, desc, ErrorType.Validation);
    public static Error NotFound(string code, string desc)   => new(code, desc, ErrorType.NotFound);
    public static Error Conflict(string code, string desc)   => new(code, desc, ErrorType.Conflict);
    public static Error Problem(string code, string desc)    => new(code, desc, ErrorType.Problem);
}
```

`Error` — **value object**: `record` даёт value-равенство (две ошибки с одинаковыми полями равны), что позволяет сравнивать с `Error.None`. **Приватный ctor + статические фабрики** задают **закрытый набор категорий** — снаружи `Error` нельзя собрать произвольно, только через осмысленные точки входа.

csharp

```csharp
public class Result
{
    protected Result(bool isSuccess, Error error)
    {
        if (isSuccess && error != Error.None || !isSuccess && error == Error.None)
            throw new ArgumentException("Invalid error for the result state.", nameof(error));
        IsSuccess = isSuccess;
        Error = error;
    }
    public bool IsSuccess { get; }
    public bool IsFailure => !IsSuccess;
    public Error Error { get; }

    public static Result Success() => new(true, Error.None);
    public static Result Failure(Error error) => new(false, error);
    public static Result<T> Success<T>(T value) => new(value, true, Error.None);
    public static Result<T> Failure<T>(Error error) => new(default, false, error);
}

public sealed class Result<TValue> : Result
{
    private readonly TValue? _value;
    internal Result(TValue? value, bool isSuccess, Error error) : base(isSuccess, error)
        => _value = value;

    public TValue Value => IsSuccess
        ? _value!
        : throw new InvalidOperationException("Cannot access the value of a failure result.");
}
```

Ключевые решения:

- **`protected` ctor + фабрики** (`Success`/`Failure`) — нельзя создать `Result` напрямую, только через осмысленные входы.
- **Инвариант в конструкторе** — `success` обязан идти с `Error.None`, `failure` — с реальной ошибкой. Нарушение = `ArgumentException`: некорректное состояние невозможно собрать.
- **`Value` бросает при доступе на failure** — fail-fast против неправильного использования: достал value без проверки `IsSuccess` — получи исключение немедленно, а не `null` молча дальше.

Использование:

csharp

```csharp
Result<User> result = userRepository.GetById(id);
if (result.IsFailure)
    return result.Error.Type switch
    {
        ErrorType.NotFound => NotFound(result.Error.Description),
        _                  => Problem(result.Error.Description)
    };
var user = result.Value;   // безопасно: проверили IsFailure выше
```

#### B3. Composition (Map / Bind / Match)

Проверять `IsFailure` после каждого шага вручную — шумно. Композиционные методы превращают цепочку «операций, каждая из которых может упасть» в один поток, где первая ошибка автоматически «съезжает на красный путь» (railway-oriented programming):

- **`Map`** — трансформирует value успеха, ошибку пробрасывает: `Result<User> → Result<UserDto>`.
- **`Bind`** (он же `flatMap`) — для шага, который **сам возвращает `Result`**: `Result<User> → Result<Order>`. Если предыдущий шаг — failure, следующий не выполняется.
- **`Match`** — терминальный: разворачивает оба исхода в обычное значение, заставляя обработать и success, и failure.

![[Pasted image 20260624112449.png]]

Цепочка на практике — три падающих шага без единого `if`:

csharp

```csharp
return userRepository.GetById(id)            // Result<User>
    .Bind(user => order.Place(user))         // Result<Order>   — не выполнится, если выше failure
    .Map(o => new OrderDto(o))               // Result<OrderDto>
    .Match(                                   // разворачиваем оба исхода
        dto => Ok(dto),
        error => Problem(error.Description));
```

> **🔬 Глубже / на синьора.** `Bind` — монадическая операция: `Result<A> → (A → Result<B>) → Result<B>`. Её короткое замыкание на первой ошибке — и есть «красный путь» railway. `Map` — функтор (`A → B`, без знания о Result). Различие важно: если шаг сам может упасть (вернуть `Result`) — нужен `Bind`, иначе получишь вложенный `Result<Result<B>>`; если шаг чистый (`A → B`) — `Map`.

#### B4. Агрегация ошибок (валидация)

`Bind`-цепочка **короткозамыкается на первой ошибке** — отлично для «шаг зависит от предыдущего», плохо для валидации формы, где нужно собрать **все** ошибки сразу. Здесь нужен не монадический, а **applicative**-стиль: применить независимые проверки и аккумулировать ошибки.

csharp

```csharp
public static Result<User> Validate(string email, int age)
{
    var errors = new List<Error>();
    if (!email.Contains('@')) errors.Add(Error.Validation("email", "Invalid email"));
    if (age < 18)             errors.Add(Error.Validation("age", "Must be 18+"));
    return errors.Count == 0
        ? Result.Success(new User(email, age))
        : Result.Failure<User>(errors[0]); // либо расширить Error до коллекции
}
```

> **🔬 Глубже / на синьора.** Монада vs applicative: монада (`Bind`) — шаги **зависимы** и последовательны (второй нужен результат первого), при ошибке нет смысла идти дальше → short-circuit. Applicative — шаги **независимы**, можно прогнать все и собрать ошибки. Библиотеки (FluentValidation, language-ext) реализуют именно applicative-агрегацию; самописный `Result` обычно расширяют до `Error[]`.

#### B5. Честный trade-off — когда что

||Исключения|Result|
|---|---|---|
|**Семантика**|непредвиденное|ожидаемый исход|
|**Цена**|дорого (бросок + раскрутка)|дёшево (обычный return)|
|**Видимость**|невидимо в сигнатуре|в типе → компилятор обяжет|
|**Проброс через слои**|бесплатно (сам летит)|вручную (`Bind`/проброс)|
|**Stack trace**|автоматический|нет (нужно класть в `Error`)|

**Result выигрывает** на ожидаемых доменных сбоях: явность, дешевизна, компилятор-as-guardrail. **Result проигрывает** там, где исключения сильны: нельзя бесплатно пробросить через 5 слоёв, шумит сигнатуры, нет авто-трейса, и он **не для truly exceptional** (потеря соединения с БД, OOM — это исключения). Практика: **Result для предсказуемых бизнес-ошибок, исключения для непредвиденного и границ инфраструктуры.**

---

### Interview layer

**1. `throw` vs `throw ex` — в чём разница?** _(очень частый, mid)_  
→ follow-up: _что именно теряется?_ → trap: _как пробросить уже пойманное исключение из другого метода / после await?_  
Слабый ответ: «`throw ex` тоже бросает, разницы нет» — проваливает trap про stack trace.  
**Модель-ответ:** `throw;` сохраняет оригинальный stack trace, `throw ex;` затирает его — точка броска становится текущей строкой, и на проде trace указывает на catch вместо реального места падения. Для проброса захваченного исключения вне catch — `ExceptionDispatchInfo.Capture(ex).Throw()`, он сохраняет исходный стек.

**2. Почему нельзя использовать исключения для control flow?** _(частый, mid+)_  
→ follow-up: _в чём конкретно цена?_ → trap: _а сам `try/catch` без броска замедляет happy path?_  
Слабый ответ: «потому что это плохая практика» — без механизма не проходит follow-up.  
**Модель-ответ:** бросок дорогой — рантайм захватывает stack trace и делает два прохода по стеку (поиск обработчика + раскрутка с `finally`), это на порядки дороже `return`. На горячем пути убивает throughput. При этом сам `try/catch` без броска почти бесплатен на современном JIT — платишь за `throw`, а не за наличие try.

**3. Когда `finally` НЕ выполнится?** _(mid+/senior)_  
Слабый ответ: «`finally` выполняется всегда» — неверно.  
**Модель-ответ:** при `StackOverflowException`, `Environment.FailFast`, жёстком убийстве процесса и зависании самого `try`. Во всех штатных исходах, включая `return` из `try` и проброс исключения, — выполнится.

**4. Зачем `catch when`, если есть `catch` по типу?** _(mid+)_  
→ trap: _в каком порядке `when` относительно `finally` нижнего кадра?_  
**Модель-ответ:** фильтр смотрит на свойства исключения, не только на тип, и логирует без перехвата (`when (Log(ex))`, возвращающий false). Механически фильтр выполняется на Pass 1 — до раскрутки стека, поэтому `when` верхнего кадра срабатывает раньше `finally` нижнего.

**5. Иерархия `Exception`; наследоваться ли от `ApplicationException`?** _(mid)_  
**Модель-ответ:** нет. Деление `SystemException`/`ApplicationException` историческое, правило про `ApplicationException` отменено. Свои исключения наследуют напрямую от `Exception`, с тремя стандартными конструкторами и суффиксом `Exception`.

**6. Result vs исключения — когда что?** _(senior, архитектурный)_  
→ follow-up: _чем плох `catch (Exception)` везде?_ → trap: _как Result композится через слои без ручных `if`?_  
Слабый ответ: «Result — это просто чище / модно» — без trade-off не проходит.  
**Модель-ответ:** исключения — для непредвиденного и границ инфраструктуры; Result — для ожидаемых доменных сбоев (не найден, валидация), где важна явность и дешевизна. Result виден в типе, поэтому компилятор обязывает обработать ошибку. Композится через `Map` (чистая трансформация), `Bind` (шаг сам возвращает Result, short-circuit на первой ошибке — railway), `Match` (терминальная обработка обоих исходов). Минусы: шумит сигнатуры, нет авто-трейса, нельзя бесплатно пробросить через слои.

**7. (🔬) Что значит «unhandled exception» и что с процессом?** _(senior)_  
**Модель-ответ:** долетело до верха стека без обработчика. С .NET 2.0+ необработанное исключение на любом потоке роняет процесс. Последний шанс залогировать — `AppDomain.UnhandledException`; `async void` роняет процесс именно так, потому что исключению некуда деться.