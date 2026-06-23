
### Зачем эта глава

`static` — одно из самых заюзанных слов в C#: `Math`, `Console`, логгеры, фабрики, синглтоны, контейнеры extension methods. И именно поэтому к нему лёгкое, ложное чувство «да это просто общая глобальная переменная». Эта наивная модель прячет три реальные ловушки, по которым и бьют на собесе: **потокобезопасность** (общее состояние на все потоки), **тайминг инициализации** (когда именно оно создаётся) и **lifetime/утечки** (static живёт весь процесс и держит ссылки). Глава строит правильную модель, из которой все три ловушки выводятся сами.

### Mental model

**`static` = принадлежит типу, а не экземпляру; одна копия на закрытый тип; живёт всё время процесса.**

||static|instance|
|---|---|---|
|Копий|одна на закрытый тип|по одной на объект|
|Доступ|по имени типа (`Counter.Total`)|через ссылку на объект|
|`this`|нет (экземпляра не существует)|есть|
|Lifetime|весь процесс (load context)|пока жив объект|

Из «одна копия» растёт общее состояние → потокобезопасность. Из «живёт всегда» растут утечки. Из «на тип» растут тайминг инициализации и generic-нюанс.

### 1. static-член vs static-класс

**static-член** (поле, свойство, метод, событие, конструктор) принадлежит типу: одна копия, доступ по имени типа, без `this` — потому что инстанса, на который мог бы указывать `this`, просто нет.

**static-класс** — контейнер только из static-членов. Его нельзя инстанцировать, нельзя наследовать и от него нельзя наследоваться, нельзя использовать как тип переменной/параметра, и он не может реализовать интерфейс. Механизм под капотом: в IL static-класс помечен одновременно как `sealed` **и** `abstract` — `abstract` запрещает создание экземпляра, `sealed` запрещает наследование. Отсюда все ограничения разом. Применение — утилиты (`Math`, `Path`), контейнеры extension methods, хелперы.

![[Pasted image 20260623104307.png]]

### 2. Инициализация

**static constructor** — parameterless, без модификатора доступа (`static Counter() { ... }`), вызывается рантаймом **автоматически один раз** на закрытый тип. Ты его не зовёшь — CLR сам триггерит инициализацию типа.

**Field initializers** (`static int x = 5;`) выполняются в **текстовом порядке**. Если у типа есть явный static ctor, инициализаторы полей выполняются в начале этого ctor, перед его телом.

Тонкая ловушка порядка: если static-инициализация типа `A` читает static-поле типа `B`, а `B` ещё не инициализирован, можно получить дефолтное (нулевое) значение — особенно при **циклических** static-зависимостях. Это не гипотетика, а реальный источник «магических нулей» на старте.

> **🔬 Глубже: beforefieldinit.** Если явного static ctor **нет**, компилятор помечает тип флагом `beforefieldinit`, и рантайм волен инициализировать его **раньше/лениво** — в любой момент до первого доступа к static-полю, не обязательно ровно на нём (может и при загрузке типа). Явный static ctor флаг **снимает**: инициализация гарантирована перед **первым обращением к типу** — доступом к static-члену **или** созданием экземпляра. Это меняет только **тайминг** — гарантия «ровно один раз и потокобезопасно» остаётся в обоих случаях. Практический вывод: если паттерн рассчитывает на «инициализация ровно при первом обращении» (ленивый синглтон), нужен явный static ctor. (IL-флаг детальнее — в главе `const`/`readonly`.)

### 3. Thread-safety

Здесь живёт главная путаница темы, и её надо развести надвое.

**static ctor — потокобезопасен.** CLR гарантирует, что инициализатор типа выполнится ровно один раз; если несколько потоков одновременно триггерят тип, все, кроме одного, блокируются до завершения инициализации. Именно на этой гарантии стоит идиома `static readonly`-синглтона.

**Изменяемые static-поля — НЕ потокобезопасны.** Поле общее для всех потоков, поэтому любой read-modify-write (`Total++`, добавление в static-коллекцию) из нескольких потоков — гонка. Нужна синхронизация: `Interlocked`, `lock`, конкурентные коллекции — либо иммутабельность.

Ловушка собеса: «static ctor thread-safe, значит и static-поля thread-safe». Нет. Гарантия касается **однократной инициализации**, а не последующих изменений.

### 4. Lifetime и утечки

static-поле — это **GC root**: всё, на что оно ссылается, остаётся живым, пока загружен тип, то есть фактически весь процесс. Отсюда класс багов, которых нет у обычных объектов.

Канонический пример — **static-событие**. `SomeStatic.Changed += handler;` — делегат держит ссылку на подписчика (через `Target`), и подписчик не собирается GC даже когда давно не нужен. Тот же эффект у растущих без ограничения static-коллекций и кэшей. Лечится отпиской (`-=`), weak-ссылками, ограничением роста / eviction.

### 5. static в generic-типах

static-поле в generic-типе — **по одной копии на закрытый тип**, а не одна на `Foo<T>` в целом. `Foo<int>` и `Foo<string>` — разные типы в рантайме, и `Foo<int>.Count` независим от `Foo<string>.Count`. Частая ошибка ожидания — «один счётчик на все `Foo<T>`»; на деле их столько, сколько различных `T` инстанцировано.

### 6. Singleton через static

Простейший потокобезопасный синглтон опирается ровно на гарантию из секции 3:

```csharp
public sealed class Config
{
    public static readonly Config Instance = new();
    private Config() { }
}
```

Инициализация static-поля потокобезопасна по гарантии CLR — без единого `lock`. Но из-за `beforefieldinit` момент создания может наступить **раньше** первого обращения к `Instance`. Если нужна **истинная ленивость** (создать ровно при первом запросе) плюс потокобезопасность — `Lazy<T>`:

```csharp
public sealed class Config
{
    private static readonly Lazy<Config> _lazy = new(() => new Config());
    public static Config Instance => _lazy.Value;
    private Config() { }
}
```

`Lazy<T>` откладывает создание до первого чтения `.Value` и потокобезопасен по умолчанию (`LazyThreadSafetyMode.ExecutionAndPublication`). Итог: `static readonly` — проще всего и thread-safe, но тайминг зависит от beforefieldinit; `Lazy<T>` — явная ленивость + thread-safe.

> **🔬 Глубже.** `static abstract` члены интерфейсов (C# 11) — это контракт на _статические_ члены (операторы, фабрики), реализуемый типом и диспетчеризуемый через generic-ограничение; фундамент generic math (`INumber<T>`: обобщённый код пишет `T.Zero`, `a + b` для любого числового `T`). Отдельно: `static` local functions и `static` lambdas (C# 8/9) запрещают захват внешнего состояния — это защита от случайных замыканий и лишних аллокаций, а не про static-члены типа.

### Interview layer

Вопросы по убыванию частоты. Формат: вопрос → likely follow-up → trap; затем shallow-ответ (почему плох) и tight model answer (RU + EN).

**Q1 (очень часто). Что такое `static`, чем static-член отличается от instance?**  
→ FU: сколько копий у static-поля? (одна на закрытый тип)  
→ trap: а в generic-типе `Foo<int>` vs `Foo<string>`? (разные)

- _Shallow:_ «static — общий для всех, как глобальная переменная». Прячет lifetime и thread-safety.
- _RU:_ «static-член принадлежит типу: одна копия на закрытый тип, доступ по имени типа, без `this`. Instance-член — своя копия в каждом объекте. Важные следствия — общее состояние (вопросы потокобезопасности) и время жизни на весь процесс.»
- _EN:_ «A static member belongs to the type: one copy per closed type, accessed via the type name, no `this`. An instance member has its own copy per object. The consequences that matter are shared state (thread-safety) and process-long lifetime.»

**Q2 (часто). static ctor — когда вызывается и потокобезопасен ли?**  
→ FU: какие гарантии даёт CLR?  
→ trap: значит и static-поля потокобезопасны? (нет)

- _Shallow:_ «вызывается при первом обращении». Неточно — без явного ctor тайминг ленивый (beforefieldinit).
- _RU:_ «parameterless, без модификатора, вызывается рантаймом автоматически один раз на закрытый тип. CLR гарантирует ровно один вызов и потокобезопасность: при гонке все потоки, кроме одного, блокируются до завершения. Но эта гарантия — про инициализацию типа, а не про последующие изменения static-полей.»
- _EN:_ «Parameterless, no access modifier, invoked automatically by the runtime once per closed type. The CLR guarantees exactly one run and thread-safety — on a race, all but one thread block until it finishes. But that covers type initialization, not later mutations of static fields.»

**Q3 (часто). static-поля и многопоточность — в чём опасность?**  
→ trap: «static ctor же thread-safe».

- _Shallow:_ «ну, надо лочить». Без разделения ctor vs поля не показывает понимания.
- _RU:_ «static-поля общие для всех потоков, поэтому read-modify-write (`Total++`) из нескольких потоков — гонка; нужна синхронизация (`Interlocked`, `lock`) или иммутабельность. Ключевое: "static ctor потокобезопасен" ≠ "static-поля потокобезопасны" — гарантия только про однократную инициализацию.»
- _EN:_ «Static fields are shared across all threads, so a read-modify-write (`Total++`) from several threads is a race — synchronize (`Interlocked`, `lock`) or use immutability. The key point: "the static ctor is thread-safe" ≠ "static fields are thread-safe" — the guarantee is only about one-time initialization.»

**Q4 (mid). Может ли static-класс реализовать интерфейс или быть унаследован?**

- _RU:_ «Нет. В IL static-класс одновременно `sealed` и `abstract`: `abstract` запрещает инстанцирование, `sealed` — наследование. Интерфейс он тоже не реализует — это контракт, реализуемый типом, который можно использовать как значение, а static-класс таким не является.»
- _EN:_ «No. In IL a static class is both `sealed` and `abstract`: `abstract` forbids instantiation, `sealed` forbids inheritance. It can't implement an interface either — an interface is a contract realized by a type usable as a value, which a static class isn't.»

**Q5 (mid+). Как `static` приводит к утечкам памяти?**  
→ FU: приведи конкретный механизм. (static-событие)

- _RU:_ «static-поле — это GC root: всё, на что оно ссылается, живёт, пока загружен тип, то есть весь процесс. Классика — static-событие: подписчик удерживается делегатом и не собирается даже когда не нужен. Лечится отпиской, weak-ссылками, ограничением роста static-коллекций.»
- _EN:_ «A static field is a GC root: anything it references stays alive while the type is loaded — effectively the whole process. The classic case is a static event: a subscriber is held by the delegate and never collected even when unneeded. Fix with unsubscription, weak references, or bounding static collections.»

**Q6 (mid+). `beforefieldinit` — что это и как влияет на ленивую инициализацию?**

- _RU:_ «Без явного static ctor компилятор ставит флаг `beforefieldinit`, и рантайм может инициализировать тип раньше/лениво — в любой момент до первого доступа к static-полю, не обязательно ровно на нём. Явный static ctor флаг снимает: инициализация гарантирована перед первым обращением к типу — доступом к static-члену или созданием экземпляра. Это влияет на тайминг (важно для ленивых синглтонов), но не на гарантию "один раз и потокобезопасно".»
- _EN:_ «Without an explicit static ctor the compiler sets `beforefieldinit`, and the runtime may initialize the type earlier/lazily — any time before the first static-field access, not necessarily exactly at it. An explicit static ctor removes the flag: init is guaranteed before first use of the type — a static-member access or instance creation. It affects timing (matters for lazy singletons), not the "once and thread-safe" guarantee.»

**Q7 (mid+). Реализуй потокобезопасный Singleton.**  
→ trap: чем `static readonly` отличается от `Lazy<T>` по таймингу?

- _RU:_ «Простейший — `public sealed class S { public static readonly S Instance = new(); private S() {} }`: инициализация static-поля потокобезопасна по гарантии CLR, без `lock`. Но из-за beforefieldinit создание может произойти раньше первого обращения к `Instance`. Для истинной ленивости — `Lazy<T>`: откладывает создание до первого `.Value` и потокобезопасен по умолчанию.»
- _EN:_ «Simplest: `public sealed class S { public static readonly S Instance = new(); private S() {} }` — the static-field init is thread-safe by the CLR guarantee, no `lock`. But beforefieldinit may create it earlier than the first `Instance` access. For true laziness use `Lazy<T>` — it defers creation to the first `.Value` and is thread-safe by default.»

**Q8 (🔬). static-поле в `Foo<int>` и `Foo<string>` — одно или разные?**

- _RU:_ «Разные. static-поле — по одной копии на закрытый тип, а `Foo<int>` и `Foo<string>` — разные типы в рантайме. `Foo<int>.Count` и `Foo<string>.Count` независимы.»
- _EN:_ «Separate. A static field is one copy per closed type, and `Foo<int>` and `Foo<string>` are distinct runtime types. `Foo<int>.Count` and `Foo<string>.Count` are independent.»

**Q9 (🔬). `static abstract` члены интерфейса — зачем?**

- _RU:_ «C# 11: интерфейс объявляет статические члены (операторы, фабрики), которые реализует тип, а вызываются они через generic-ограничение. Фундамент generic math (`INumber<T>`): обобщённый алгоритм пишет `T.Zero` или `a + b` для любого числового `T`.»
- _EN:_ «C# 11: an interface declares static members (operators, factories) implemented by the type and dispatched via a generic constraint. It's the basis of generic math (`INumber<T>`): a generic algorithm can write `T.Zero` or `a + b` for any numeric `T`.»