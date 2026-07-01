
Представь горячий цикл, который разбирает большой текстовый лог или бинарный буфер. Ты режешь строку на куски: `line.Substring(0, i)`, потом ещё, потом парсишь число из подстроки. Каждый `Substring` — это **новый объект в куче** и копия символов. На одной строке незаметно, а на миллионе строк в секунду ты завалил GC мусором, которого могло не быть: сами данные уже лежат в исходной строке, ты просто хотел _посмотреть_ на её кусок.

Хочется вот чего: взять «окно» на участок памяти — часть массива, часть строки, буфер, выделенный на стеке, — **не копируя** и не аллоцируя, работать с ним как с массивом (индекс, `Length`, slice), и чтобы это было одинаково для любого источника. Ровно это даёт `Span<T>`.

Но у такого окна есть опасная сторона. Если Span смотрит на буфер, который живёт только внутри текущего кадра стека (`stackalloc`), а ты умудришься сохранить этот Span в поле объекта или вернуть из метода — кадр снимется, память переиспользуется, а Span продолжит на неё указывать. Это классический **use-after-free**, дыра в безопасности памяти, которой в managed-мире быть не должно.

`ref struct` — это и есть контракт, который делает `Span<T>` безопасным: компилятор гарантирует, что такой тип **никогда не покинет стек**, поэтому указатель внутри него не переживёт свою память. Span — это «зачем», ref struct — это «как сделать безопасно».

### Mental model

- **`Span<T>`** — типобезопасное окно `(managed pointer + length)` на непрерывный участок памяти, где бы тот ни лежал: массив (heap), `stackalloc` (stack), строка, unmanaged. Он ничем не владеет и ничего не аллоцирует — только смотрит и (если не `ReadOnly`) пишет.
- **`ref struct`** — value type, который компилятор прибивает к стеку: он не может попасть в кучу _ни при каких условиях_, поэтому может безопасно хранить внутри managed pointer на короткоживущую память.

### Как это работает

#### Span физически — это два поля

`Span<T>` — это `readonly ref struct` из ровно двух полей: managed pointer на начало окна и `int Length`. На x64 это ≈16 байт, целиком на стеке. Он не хранит данные — он на них указывает. `ReadOnlySpan<T>` устроен так же, но запрещает запись; `Span<T>` неявно приводится к `ReadOnlySpan<T>`, не наоборот.

Ключевое следствие: **один и тот же тип `Span<byte>` оборачивает память из любого источника без копирования.** Slice (`span.Slice(2, 3)` или `span[2..5]`) не аллоцирует — он лишь сдвигает указатель и уменьшает длину.

Здесь диаграмма показывает геометрию, которую прозой не передать чисто:

![[Pasted image 20260701132945.png]]

Один источник, разные бэкенды — Span'у всё равно:

```csharp
byte[] heap = new byte[8];
Span<byte> a = heap;                    // окно на массив в куче
Span<byte> b = stackalloc byte[8];      // окно на буфер в стеке
ReadOnlySpan<char> c = "hello world";   // окно на строку (только чтение)
ReadOnlySpan<byte> d = someNativeBuffer; // окно на unmanaged-память

Span<byte> window = heap.AsSpan(2, 3);  // slice без копии и без аллокации
```

#### `stackalloc` → `Span` без `unsafe`

Раньше `stackalloc` жил в `unsafe`-контексте с сырыми указателями. Начиная с C# 7.2, `stackalloc` можно присвоить в `Span<T>` — компилятор сам следит за безопасностью, `unsafe` не нужен. Это даёт zero-allocation работу с временными буферами:

```csharp
static bool TryParseId(ReadOnlySpan<char> input, out int id)
{
    Span<char> buf = stackalloc char[16];      // на стеке, ноль аллокаций
    int n = 0;
    foreach (var ch in input)
        if (char.IsDigit(ch)) buf[n++] = ch;
    return int.TryParse(buf[..n], out id);      // int.TryParse принимает ReadOnlySpan<char>
}
```

Грабли, которые любят на собесе: **`stackalloc` в цикле** накапливает выделения в кадре метода (память не освобождается на каждой итерации, только при выходе из метода) — легко получить `StackOverflowException`. И вообще размер `stackalloc` должен быть маленьким и ограниченным (десятки-сотни байт); стек — это ~1 МБ на поток. Большой или зависящий от ввода размер выделяй в куче через `ArrayPool<T>`.

#### Почему Span обязан быть `ref struct`

Вернёмся к опасности из подводки. Если бы Span можно было положить в поле объекта, вернуть в кучу или забоксить — managed pointer внутри него уехал бы туда, где живёт дольше, чем память, на которую он смотрит. Для `stackalloc`-буфера это мгновенный use-after-free. Ограничения `ref struct` — не каприз языка, а именно та стена, которая это перекрывает:

![[Pasted image 20260701133002.png]]

Каждый запрет — это перекрытый маршрут в кучу. Поле объекта, элемент массива, boxing, замыкание, машина состояний `async`/итератора — всё это heap-объекты. Пустить туда managed pointer значит разрешить ему пережить свою память. Компилятор просто не даёт скомпилировать такой код.

#### C# 14 / .NET 10: first-class span

Долго язык держал span-типы «на расстоянии»: чтобы передать массив в метод, принимающий `Span<T>`, нужен был `.AsSpan()`. C# 14 добавляет встроенные неявные преобразования между `Span<T>`, `ReadOnlySpan<T>` и `T[]`, делая работу с этими типами естественнее. Теперь массив и срез конвертируются в span сами: [Microsoft Learn](https://learn.microsoft.com/en-us/dotnet/csharp/whats-new/csharp-14)

```csharp
void Process(ReadOnlySpan<int> data) { /* ... */ }

int[] arr = { 1, 2, 3, 4, 5 };
Process(arr);        // C# 14: неявно, без .AsSpan()
Process(arr[..3]);   // срез массива тоже конвертируется
```

Span-типы также могут выступать receiver'ом для extension-методов и лучше участвуют в выводе типов. Пин версии: до C# 14 (`LangVersion <= 13`) всё это требует явного `.AsSpan()`; поведение включается на .NET 10 / C# 14. Есть и обратная сторона — см. breaking change в 🔬-блоке. [Microsoft Learn](https://learn.microsoft.com/en-us/dotnet/csharp/whats-new/csharp-14)

### 🔬 Глубже / на синьора

#### managed pointer, ref fields, `scoped`

Внутри `Span<T>` лежит **managed pointer** (`ref T`, он же byref, он же interior pointer) — особая ссылка рантайма, доступная с .NET 1.0. В отличие от обычной ссылки на объект, managed pointer может указывать _внутрь_ чего угодно: на элемент массива, на поле объекта, на символ в строке, на стековую переменную, на unmanaged-память. Его отслеживает GC (при компактификации кучи он корректно сдвигается), и он быстрее обычной ссылки за счёт stack-only природы.

До .NET 7 Span хранил это через внутренний тип `ByReference<T>`. **C# 11 / .NET 7** ввёл настоящие `ref fields`, и теперь Span объявлен буквально так:

```csharp
public readonly ref struct Span<T>
{
    internal readonly ref T _reference;  // ref field, C# 11
    private readonly int _length;
}
```

Вместе с ref fields пришёл `scoped` — модификатор, сужающий lifetime ссылки/ref struct до текущего кадра. Компилятор проводит **ref safety analysis**: у каждого ref-значения есть «safe-to-escape» область, и вернуть/сохранить его за её пределы нельзя. `scoped` — способ явно сказать «этот параметр не убегает», чтобы разрешить более агрессивные, но безопасные паттерны.

#### Что изменил C# 13 — здесь устаревают старые знания

Три ограничения `ref struct`, которые ещё вчера были абсолютными, в **C# 13 / .NET 9** ослаблены. Если ответишь по-старому — покажешь, что учил по статьям 2020 года:

- Начиная с C# 13, `ref struct` может реализовывать интерфейсы — но не может быть приведён к типу интерфейса, потому что это boxing-конверсия. Пользы от этого напрямую мало; работает в связке со следующим пунктом. [Microsoft Learn](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/builtin-types/ref-struct)
- C# 13 добавил анти-ограничение `allows ref struct`: параметр-тип с `where T : allows ref struct` разрешает подставлять ref struct как аргумент дженерика, и компилятор применяет к нему все ref safety правила. Это первое анти-ограничение в C# — оно не сужает, а расширяет набор допустимых типов. Именно так `Span<T>` теперь протаскивается в generic-алгоритмы. [Microsoft Learn](https://learn.microsoft.com/en-us/dotnet/csharp/whats-new/csharp-13?WT.mc_id=DT-MVP-4021952)[GitHub](https://github.com/dotnet/runtime/issues/102671)
- До C# 13 методы-итераторы и `async`-методы не могли объявлять ref-локали. В C# 13 `async`-метод может объявить локаль ref struct типа — но к ней нельзя обращаться через `await`. [Microsoft Learn](https://learn.microsoft.com/en-us/dotnet/csharp/whats-new/csharp-13?WT.mc_id=DT-MVP-4021952)

То есть корректная формулировка теперь: «`Span` нельзя _держать через_ `await`/`yield`», а не «`Span` нельзя в `async`». Тонкая, но синьорская граница.

#### `Memory<T>` vs `Span<T>`

Когда буфер должен пережить `await` (типично для асинхронного I/O) — Span не подходит по определению. Для этого есть `Memory<T>`: это **обычный** `readonly struct`, не ref struct. Он может лежать в поле, в куче, проходить через `async`. В точке использования из него достают Span через `.Span`.

||`Span<T>`|`Memory<T>`|
|---|---|---|
|Тип|`ref struct`|обычный `readonly struct`|
|Где живёт|только стек|стек или куча|
|Поле класса|✗|✓|
|Через `await` / `yield`|✗ (нельзя держать)|✓|
|Доступ к данным|напрямую (`span[i]`)|через `.Span`|
|Назначение|синхронный hot-path|асинхронные буферы, хранение|

Правило: **Span — рабочая лошадка синхронного кода; Memory — «конверт», который можно хранить и таскать через async, разворачивая в Span на месте.**

#### Цена и границы — где Span не бесплатен

- **API-виральность.** Раз Span нельзя положить в поле, ты не кэшируешь его в объекте — ты протягиваешь его параметрами через методы. Переход API на Span нередко расходится волной по всей цепочке вызовов.
- **Никакого async напрямую** — только через `Memory<T>`.
- **Bounds-checks.** Индексатор Span проверяет границы. Но в циклах с известной длиной JIT часто их элиминирует (bounds-check elimination), поэтому на практике скорость близка к массиву, а не хуже.
- Span — это _view_, а не хранилище: backing-память всё равно нужно откуда-то взять (массив, стек, пул).

#### C# 14 breaking change — ловушка на синьора

За first-class span есть плата в overload resolution. В C# 14 методы с параметрами `ReadOnlySpan<T>`/`Span<T>` начинают участвовать в выводе типов и как extension-методы в большем числе сценариев. В частности, span-методы вроде `MemoryExtensions.Contains` теперь связываются внутри Expression-лямбд, где при компиляции с интерпретацией они бросают исключение в рантайме. Лечение — привязать не-span перегрузку: например, привести аргумент к `IEnumerable<T>` или вызвать `Enumerable.Contains` явно. Красивый вопрос-ловушка «почему после апгрейда на .NET 10 упал код с Expression-деревьями». [Microsoft Learn + 2](https://learn.microsoft.com/en-us/dotnet/core/compatibility/core-libraries/10.0/csharp-overload-resolution)

### Собеседование

#### Вопросы по частоте (mid+ → синьор)

1. **Что такое `Span<T>` и зачем он нужен?** → _likely:_ чем отличается от массива? → _trap:_ где именно живут данные Span'а? (где угодно — heap/stack/unmanaged; сам Span лишь указатель + длина).
2. **Почему `Span<T>` — это `ref struct`? Почему его нельзя положить в поле класса или подержать через `await`?** → _likely:_ что конкретно гарантируют эти ограничения? → _trap:_ а как тогда держать буфер через `await`? (`Memory<T>`).
3. **`Span` vs `Memory` — когда что?** → _trap:_ почему `Memory` — не ref struct, а `Span` — да?
4. **Покажи zero-allocation парсинг через `stackalloc` + `Span`. Какие грабли?** → _trap:_ `stackalloc` в цикле; размер, зависящий от ввода → stack overflow.
5. **(синьор) Что Span хранит внутри? Что такое managed pointer?** → ref fields (C# 11), `scoped`, ref safety.
6. **(синьор) Что изменилось для `ref struct` в C# 13?** → интерфейсы, `allows ref struct`, ref-локали в async/итераторах. Ловит устаревшие знания.

#### Частые поверхностные ответы и почему проваливаются

- **«Span — это просто быстрый массив / замена массива».** Нет: Span ничем не владеет и не аллоцирует, это _view_; он не может быть полем, элементом массива или жить через `await`. Массив всего этого не запрещает.
- **«Span всегда лежит на стеке» / «Span хранит данные на стеке».** Миф. На стеке лежит _сам_ Span (16-байтная структура); данные — где угодно, чаще всего в куче. `stackalloc` — лишь один частный источник.
- **«`ref struct` нельзя в `async`, точка».** Устарело. С C# 13 локаль ref struct типа в `async` можно — нельзя лишь держать её _через_ `await`.
- **«`ref struct` не может реализовать интерфейс».** Устарело с C# 13. Может — но привести его к типу интерфейса нельзя (это boxing).
- **«`Memory` — то же самое, что `Span`».** Нет: `Memory` — обычный struct, живёт в куче и в async; Span достаётся из него через `.Span` в точке использования.

#### Модель-ответы

**Q1 — что такое `Span<T>` и зачем.**

**RU.** `Span<T>` — это типобезопасное окно на непрерывный участок памяти: структура из managed pointer и длины (~16 байт), которая не владеет данными и не аллоцирует. Один и тот же `Span<T>` одинаково оборачивает массив в куче, буфер из `stackalloc` на стеке, строку (`ReadOnlySpan<char>`) или unmanaged-память, а slice не копирует, а лишь сдвигает указатель и длину. Нужен для zero-allocation работы с кусками памяти в горячем коде — парсинг, срезы, буферы — там, где раньше плодили `Substring` и промежуточные массивы, нагружая GC.  
_Senior add-on._ Внутри — `ref T` field (настоящие ref fields с C# 11), managed pointer отслеживается GC и может указывать внутрь объекта, строки, стека или нативной памяти.

**EN.** `Span<T>` is a type-safe window over a contiguous region of memory: a struct of a managed pointer plus a length (~16 bytes) that owns nothing and allocates nothing. The same `Span<T>` uniformly wraps a heap array, a `stackalloc` stack buffer, a string (`ReadOnlySpan<char>`), or unmanaged memory, and slicing just adjusts the pointer and length instead of copying. It exists for zero-allocation work over chunks of memory in hot paths — parsing, slicing, buffers — where you'd otherwise spawn `Substring`s and temporary arrays and pressure the GC.  
_Senior add-on._ Internally it's a `ref T` field (real ref fields since C# 11); the managed pointer is GC-tracked and can point inside an object, a string, the stack, or native memory.

**Q2 — почему `Span` — это `ref struct`.**

**RU.** Потому что Span хранит managed pointer, который может указывать на короткоживущую память — например, на `stackalloc`-буфер в текущем кадре. Если бы Span можно было сохранить в поле объекта, забоксить, положить в массив или подержать через `await`, указатель уехал бы в кучу и пережил бы свою память — это use-after-free. `ref struct` — контракт компилятора: тип не может попасть в кучу никаким путём, значит указатель внутри не переживёт кадр. Если буфер нужен через `await`, берут `Memory<T>` — обычный struct, который живёт в куче и разворачивается в Span через `.Span` в точке использования.  
_Senior add-on._ С C# 13 ограничения точечно ослаблены: ref struct может реализовывать интерфейсы (но не приводиться к ним — это boxing), подставляться в дженерики через `allows ref struct`, и объявляться локалью в async/итераторе — просто не жить через `await`/`yield`.

**EN.** Because a Span holds a managed pointer that may point to short-lived memory — e.g. a `stackalloc` buffer in the current frame. If a Span could be stored in a field, boxed, put into an array, or held across an `await`, that pointer would escape to the heap and outlive its memory — a use-after-free. `ref struct` is the compiler's contract: the type can never reach the heap by any route, so the pointer inside can't outlive the frame. If a buffer must survive an `await`, use `Memory<T>` — an ordinary struct that lives on the heap and is unwrapped into a Span via `.Span` at the point of use.  
_Senior add-on._ C# 13 relaxed the rules narrowly: a ref struct can implement interfaces (but not be converted to them — that's boxing), be a generic argument via `allows ref struct`, and be declared as a local in async/iterator methods — it just can't live across `await`/`yield`.