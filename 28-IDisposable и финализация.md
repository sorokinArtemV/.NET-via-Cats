Из главы про GC ты уже знаешь: сборщик сам разбирается с managed-памятью и освобождать её вручную не нужно. Но есть ресурсы, до которых GC не дотягивается, — **unmanaged**: файловые и сетевые дескрипторы, сокеты, подключения к БД, нативные буферы, GDI-объекты. Это не «память в куче», а записи в таблицах ОС, и они **не бесконечны**: дескрипторы и соединения — scarce-ресурс, который легко исчерпать.

Проблема в двух вещах. Во-первых, GC не знает, как закрыть файл или сокет, — он умеет только освобождать managed-память. Во-вторых, даже если бы знал, он делает это **недетерминированно**: сборка случится «когда-нибудь», а файл, который ты держишь открытым лишние полминуты, может заблокировать чужую запись или исчерпать пул соединений.

`IDisposable` решает ровно это: даёт тебе **детерминированную точку освобождения** — ты сам говоришь «ресурс больше не нужен, закрой его сейчас», не дожидаясь GC. А финализатор — это страховочная сетка на случай, если вызвать `Dispose()` забыли. Эта глава — про оба механизма, про канонический Dispose pattern и про то, почему финализатор сегодня почти всегда пишут не руками, а через `SafeHandle`.

> Финализация тесно связана с GC: объект с финализатором проходит через очередь финализации и переживает лишнюю сборку (механику очередей мы заложили в главе «Garbage Collector»). Здесь разбираем это со стороны кода.

---

### 🧠 Mental model

> `IDisposable.Dispose()` — это «освободи unmanaged/scarce-ресурсы **сейчас**», детерминированно, в точке выхода из `using`. Финализатор (`~T()`) — это «освободи их **когда-нибудь**, если про Dispose забыли»: недетерминированная страховка ценой лишнего цикла GC.

Главное, что надо произнести: **Dispose — про детерминизм освобождения ресурсов, а не про память. Память — это GC.**

---

### Как это работает

#### 1. Детерминированно vs недетерминированно

`using` — это синтаксический сахар над `try/finally`: `Dispose()` вызывается на выходе из блока **в любом случае**, в том числе при исключении.

```csharp
// using-statement
using (var file = new FileStream("data.txt", FileMode.Open))
{
    // работаем с file
} // Dispose() вызван здесь, даже если внутри бросили исключение

// using-declaration (C# 8): Dispose в конце текущего scope
using var conn = new SqlConnection(cs);
conn.Open();
// Dispose() вызовется в конце метода
```

Если же `Dispose()` не вызвать, а у объекта есть финализатор, ресурс всё равно когда-нибудь освободится — но только когда до объекта дойдёт GC и отработает finalizer thread. Для редких ресурсов это «поздно».

![[Pasted image 20260630122620.png]]

#### 2. Финализатор и его цена

Финализатор в C# — это `~ClassName()`, который компилятор превращает в override `Object.Finalize()`. Он работает как страховка: если объект с финализатором стал недостижим, GC не собирает его сразу, а ставит в **очередь финализации**, отдаёт отдельному finalizer thread, тот выполняет `~T()`, и только следующая сборка освобождает память.

Отсюда — стоимость финализатора, и её надо уметь назвать:

- **Лишний цикл GC.** Объект не собирается в первую сборку, а доживает до второй → автоматически **продвигается в старшее поколение** (promotion), где сборки реже и дороже.
- **Недетерминизм.** Момент запуска `~T()` не гарантирован; порядок финализации объектов не определён.
- **Опасность в коде финализатора.** К моменту `~T()` другие managed-объекты, на которые ты ссылаешься, **могут быть уже финализированы** — трогать их нельзя.

![[Pasted image 20260630122641.png]]

Поэтому в `Dispose()` после ручной очистки вызывают `GC.SuppressFinalize(this)` — он снимает объект с очереди финализации, и лишний цикл не происходит.

#### 3. Канонический Dispose pattern

Когда класс **сам напрямую владеет unmanaged-ресурсом** и хочет дать и детерминированную очистку, и страховку-финализатор, используют классический паттерн с `Dispose(bool disposing)`:

```csharp
public class NativeResource : IDisposable
{
    private IntPtr _handle;          // unmanaged
    private Stream? _managed = new MemoryStream();  // managed, disposable
    private bool _disposed;

    public void Dispose()
    {
        Dispose(disposing: true);
        GC.SuppressFinalize(this);   // финализатор больше не нужен
    }

    protected virtual void Dispose(bool disposing)
    {
        if (_disposed) return;       // идемпотентность
        if (disposing)
        {
            // вызвано из Dispose(): managed-объекты ещё живы — можно трогать
            _managed?.Dispose();
            _managed = null;
        }
        // unmanaged освобождаем всегда (и из Dispose, и из финализатора)
        Free(_handle);
        _handle = IntPtr.Zero;
        _disposed = true;
    }

    ~NativeResource() => Dispose(disposing: false);  // страховка
}
```

Что здесь важно объяснить на собеседовании:

- **Параметр `disposing`** разделяет два пути входа. `true` — пришли из `Dispose()`, managed-объекты ещё живы, их можно безопасно диспозить. `false` — пришли из финализатора, **managed-объекты трогать нельзя** (могли быть уже финализированы), освобождаем только unmanaged.
- **`_disposed`-флаг** делает `Dispose()` идемпотентным — повторный вызов безопасен (этого требует контракт `IDisposable`).
- **`virtual Dispose(bool)`** — чтобы наследник мог дочистить своё, переопределив один метод. Сам `Dispose()` и финализатор наследник не переопределяет.

#### 4. SafeHandle — почему финализатор почти не пишут руками

Ключевая рекомендация Microsoft: **не реализуй финализатор сам — заверни unmanaged-хендл в `SafeHandle`.** `SafeHandle` уже несёт корректный critical finalizer, защиту от handle-recycling атак и потокобезопасное освобождение. Тогда твоему классу финализатор не нужен вообще — достаточно продиспозить хендл:

```csharp
public class SafeResource : IDisposable
{
    private readonly SafeHandle _handle = new SafeFileHandle(/* ... */);

    public void Dispose() => _handle.Dispose();   // финализатор не нужен
}
```

Правило для интервью: финализатор оправдан, **только если класс держит unmanaged-ресурс напрямую** (голый `IntPtr`). Если ресурсы только managed (`Stream`, `HttpClient`, другой `IDisposable`) — финализатор не пишут вообще, лишь реализуют `IDisposable` и диспозят вложенные объекты.

#### 5. IAsyncDisposable — асинхронное освобождение

Иногда корректная очистка требует I/O: дослать буфер в сеть, мягко закрыть соединение. Блокировать на этом поток в синхронном `Dispose()` плохо. Для этого есть `IAsyncDisposable` (C# 8 / .NET Core 3.0) с методом `DisposeAsync()`, возвращающим `ValueTask`, и `await using`:

```csharp
await using var conn = new AsyncDbConnection(cs);
await conn.QueryAsync(...);
// DisposeAsync() будет awaited на выходе из scope
```

Канонический async-паттерн выносит асинхронную часть в `DisposeAsyncCore()`:

```csharp
public async ValueTask DisposeAsync()
{
    await DisposeAsyncCore().ConfigureAwait(false);
    Dispose(disposing: false);   // синхронная часть для unmanaged
    GC.SuppressFinalize(this);
}
protected virtual async ValueTask DisposeAsyncCore()
{
    if (_asyncResource is not null)
        await _asyncResource.DisposeAsync().ConfigureAwait(false);
}
```

Если тип владеет async-ресурсами — реализуют `IAsyncDisposable`; часто реализуют **оба** интерфейса, чтобы тип работал и в `using`, и в `await using`.

---

### 🔬 Глубже / на синьора

**`using` — это pattern-based, а не только интерфейс.** Компилятор разворачивает `using` в `try/finally` по _наличию_ подходящего метода `Dispose()`, а не строго по `IDisposable`. Благодаря этому `using` работает с `ref struct` (которые не могли реализовать интерфейсы), если у них есть публичный `Dispose()`. Аналогично `await using` работает по наличию `DisposeAsync()`.

**`Dispose()` не должен бросать.** Исключение из `Dispose()` (особенно из finally при уже летящем исключении) маскирует исходную ошибку и роняет очистку. Гайдлайн: `Dispose()` должен быть безопасным и идемпотентным.

**Диспозить только то, чем владеешь.** Частый баг — продиспозить инъектированную зависимость или общий `HttpClient`, которым владеет кто-то другой → ты ломаешь его остальным потребителям. С `HttpClient` обратная классика: его, наоборот, **нельзя** создавать-и-диспозить на каждый запрос (исчерпание сокетов из-за `TIME_WAIT`) — используют `IHttpClientFactory`/долгоживущий клиент.

**`ConfigureAwait(false)` и стек `await using`.** Внутри `DisposeAsync()` используют `ConfigureAwait(false)`. Но «складывать» несколько `await using` с `.ConfigureAwait(false)` в одну строку рискованно — в редких ошибочных ветках `DisposeAsync` может не вызваться; MS рекомендует не стекать их, а оформлять вложенными блоками. (Async/await-механику закрываем в отдельной главе — кросс-реф.)

**`GC.ReRegisterForFinalize` / воскрешение.** Из финализатора объект можно «воскресить», вернув на него ссылку из корня (object resurrection). Это редкий и опасный приём — на интервью достаточно знать, что он существует и почему его избегают.

**Финализатор не гарантирован вовсе.** При жёстком завершении процесса финализаторы могут не успеть отработать. Опираться на `~T()` как на гарантию освобождения нельзя — это только страховка от забытого `Dispose()`.

---

### 🎤 Interview layer

#### Вопросы по частоте

**Q1 (очень часто). Что такое `IDisposable` и зачем он нужен?**  
Follow-up: _Что делает `using`?_ → Трап: _Освобождает ли `Dispose()` память самого объекта?_ (нет — память это GC; `Dispose` про unmanaged/scarce-ресурсы).

**Q2 (часто). Чем `Dispose` отличается от финализатора? Кто кого вызывает?**  
Follow-up: _Вызовет ли GC `Dispose()`?_ (нет — GC вызывает финализатор, не `Dispose`) → Трап: _Финализатор — это деструктор как в C++?_ (нет: недетерминирован, без гарантии момента, со стоимостью).

**Q3 (часто). Объясни/напиши Dispose pattern.** `Dispose(bool)`, финализатор, `SuppressFinalize`.  
Follow-up: _Зачем параметр `disposing`?_ → Трап: _Почему в финализаторе нельзя трогать managed-объекты?_ (порядок финализации не определён — они могут быть уже финализированы).

**Q4 (часто). Почему финализатор дорог?** Лишний цикл GC, promotion в старшее поколение, недетерминизм.  
Трап: _Как избежать этой цены, если Dispose всё-таки вызвали?_ (`GC.SuppressFinalize`).

**Q5 (средне). Когда нужен `SafeHandle`?** Когда класс владеет unmanaged-хендлом напрямую — `SafeHandle` заменяет ручной финализатор.

**Q6 (средне). `IDisposable` vs `IAsyncDisposable`, когда `await using`?** Когда очистка требует I/O и нельзя блокировать поток.

#### Типичные неглубокие ответы и почему проваливаются

- **«`Dispose()` освобождает память объекта».** Нет. Память освобождает GC. `Dispose` — про unmanaged/scarce-ресурсы (handles, соединения, сокеты). На follow-up про память кандидат поплывёт.
- **«GC вызовет `Dispose()` автоматически».** Нет: GC вызывает **финализатор**, а не `Dispose`. `Dispose` — твоя ответственность (через `using` или вручную).
- **«Финализатор = деструктор C++».** Синтаксис похож (`~T()`), семантика — нет: не детерминирован, момент не гарантирован, стоит лишнего цикла GC.
- **«В Dispose pattern всегда нужен финализатор».** Нет: только при прямом владении unmanaged. Для managed-ресурсов финализатор не пишут; для unmanaged предпочитают `SafeHandle`.
- **«`using` не сработает при исключении».** Сработает: это `try/finally`, `Dispose` вызовется и при исключении.

#### Модельные ответы

**Q1 — «Что такое IDisposable и зачем он нужен?»**

🇷🇺 «`IDisposable` — это контракт с методом `Dispose()` для **детерминированного** освобождения ресурсов, до которых GC не дотягивается: unmanaged-хендлов, файлов, сокетов, соединений с БД. GC управляет только managed-памятью и делает это недетерминированно, поэтому scarce-ресурсы нельзя оставлять на него. Обычно `Dispose()` вызывают через `using`, который разворачивается в `try/finally` и гарантирует вызов даже при исключении. Ключевое: `Dispose` — про ресурсы и про _когда_ их отпустить, а не про память объекта.»

🇬🇧 “`IDisposable` is a contract with a `Dispose()` method for **deterministic** release of resources the GC can’t handle — unmanaged handles, files, sockets, database connections. The GC only manages managed memory and does so non-deterministically, so scarce resources shouldn’t be left to it. You normally call `Dispose()` via `using`, which compiles to `try/finally` and guarantees the call even on exception. The key point: `Dispose` is about _resources_ and _when_ to release them, not about the object’s memory.”

**Q3/Q4 — Dispose pattern и цена финализатора**

🇷🇺 «Канонический паттерн — публичный `Dispose()`, защищённый `virtual Dispose(bool disposing)` и, если есть прямое владение unmanaged, финализатор `~T()`. `Dispose()` зовёт `Dispose(true)` и `GC.SuppressFinalize(this)`; финализатор зовёт `Dispose(false)`. Параметр `disposing` разделяет пути: при `true` (из Dispose) managed-объекты живы и их можно диспозить, при `false` (из финализатора) трогаем только unmanaged, потому что порядок финализации не определён. Финализатор дорог: объект переживает лишнюю сборку и продвигается в старшее поколение — поэтому `SuppressFinalize` после ручного `Dispose` убирает его из очереди. На практике финализатор руками почти не пишут — unmanaged заворачивают в `SafeHandle`.»

🇬🇧 “The canonical pattern is a public `Dispose()`, a protected `virtual Dispose(bool disposing)`, and — only if the class directly owns unmanaged resources — a `~T()` finalizer. `Dispose()` calls `Dispose(true)` and `GC.SuppressFinalize(this)`; the finalizer calls `Dispose(false)`. The `disposing` flag splits the paths: when `true` (from Dispose) managed objects are alive and can be disposed; when `false` (from the finalizer) you touch only unmanaged state, because finalization order is undefined. Finalizers are costly — the object survives an extra GC and gets promoted — so `SuppressFinalize` after a manual `Dispose` removes it from the queue. In practice you rarely hand-write a finalizer; you wrap unmanaged handles in a `SafeHandle` instead.”

_(senior add-on)_ «Если очистка асинхронная — реализуют `IAsyncDisposable.DisposeAsync()` и потребляют через `await using`; часто реализуют оба интерфейса.»