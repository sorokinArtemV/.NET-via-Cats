## const / readonly

### Mental model

**`const` = compile-time literal.** Значение известно компилятору и **копируется** (inline) в каждое место использования — в том числе в чужие сборки. Это не «общая переменная», а запечённый литерал.

**`readonly` = runtime-fixed binding.** Настоящее поле, которое присваивается один раз — в объявлении или в конструкторе — и читается в рантайме.

> Ведущая фраза: _«const компилятор подставляет по месту как литерал; readonly — это поле, зафиксированное после конструктора и читаемое в рантайме»._ Этой строкой разводятся обе оси: _когда известно значение_ и _где оно физически живёт_.

---

### 1. const — compile-time literal

**Как работает**

- Значение должно быть вычислимо на этапе компиляции (constant expression). Поэтому `const DateTime` или `const Guid` невозможны — их значения рождаются только в рантайме.
- `const`-поле **неявно `static`**. Доступ через имя типа, не через экземпляр. Написать `static const` нельзя — ошибка компиляции `CS0504` (модификатор избыточен и запрещён).
- Компилятор **встраивает** значение в IL по месту использования: вместо чтения поля эмитится литерал (`ldc.i4`, `ldstr` и т.д.). Само «поле» в рантайме не читается.
- `const` может быть **локальной переменной** внутри метода (`const int x = 5;`) — в отличие от `readonly`, который только модификатор поля.

**Допустимые типы.** Только то, что выражается compile-time литералом:

|Категория|Что можно|
|---|---|
|Встроенные примитивы|`sbyte byte short ushort int uint long ulong char float double decimal bool`|
|Строка|`string`|
|Перечисление|любой `enum`|
|Ссылочные типы|только значение `null`|

- `decimal` — особый случай: на уровне IL это **не** настоящий литерал. Компилятор сохраняет значение через `DecimalConstantAttribute`, поэтому `const decimal` в C# разрешён, хотя механически это не IL-константа, как у `int`.
- C# 10: разрешены **`const` interpolated strings** — `const string s = $"{A}-{B}";`, если все интерполируемые части тоже `const string`.

**`const` иногда обязателен, а не просто предпочтителен.** В compile-time контекстах `static readonly` не компилируется вовсе — там нужен именно `const`:

- аргументы атрибутов: `[Obsolete(Message)]` → `Message` обязан быть `const string`;
- метки `case` в `switch` (не-константа → `CS0150`);
- значения optional-параметров: `void Page(int size = MaxItemsPerPage)`;
- инициализация других `const`.

Это самостоятельная причина существования `const` помимо перформанса.

> 🔬 **Глубже / на синьора**
> 
> - `const`-поле помечается в метаданных флагом `literal` и хранит значение прямо в метаданных типа. Per-instance storage у него нет вовсе; на каждом обращении эмитится `ldc`/`ldstr`. Отсюда «нулевая стоимость доступа» (нет field load) — но ровно это порождает versioning hazard (раздел 3).
> - `static const` → `CS0504` именно потому, что const уже static по определению; язык запрещает дублировать модификатор.

---

### 2. readonly — runtime-fixed binding

**Как работает**

- `readonly`-поле присваивается **только** в инициализаторе поля, в теле конструктора или в `init`-аксессоре (C# 9) — но **не** в обычном методе, вызванном из конструктора (вынес инициализацию в `Init()` → `CS0191`). Для **instance `readonly`** — instance-конструктор; для **`static readonly`** — static-конструктор. После выхода из конструктора переприсвоить нельзя.
- Значение вычисляется в **рантайме**: `readonly Guid Id = Guid.NewGuid();` — у каждого экземпляра своё. `static readonly` вычисляется один раз при инициализации типа.
- `readonly` бывает instance **или** static (в отличие от const, который всегда static).
- Это **настоящее поле** — читается в рантайме через `ldfld`/`ldsfld`. Поэтому потребитель всегда видит актуальное значение из сборки-владельца (см. раздел 3).

**Shallow immutability — ключевая ловушка**

`readonly` фиксирует **binding** (саму ссылку/значение поля), а не объект за ссылкой.

csharp

```csharp
private readonly List<int> _items = new();
// _items = new();   // ошибка компиляции — переприсвоить нельзя
_items.Add(1);        // ОК — мутируем объект, ссылка та же

private readonly int[] _arr = { 1, 2, 3 };
_arr[0] = 99;         // легально — readonly ≠ immutable коллекция
```

Хочешь настоящую неизменяемость — `ImmutableArray<T>` / `ImmutableList<T>`. `IReadOnlyList<T>` лишь **прячет** API мутации, но не гарантирует, что объект не меняют через исходную ссылку.

> 🔬 **Глубже / на синьора**
> 
> - **Тайминг `static readonly` и `beforefieldinit`.** Если у типа есть **явный** static-конструктор, рантайм обязан инициализировать его перед первым обращением к статическому полю или созданием экземпляра («precise» семантика). Если static-ctor нет (только инициализаторы полей), компилятор ставит флаг `beforefieldinit`, и рантайм волен инициализировать тип **в любой момент до первого обращения к статическому полю** — потенциально раньше или лениво. Вывод: не закладывайся на точный момент инициализации `static readonly`, если не написал явный static-ctor.
> - **Defensive copy.** Если `readonly`-поле имеет тип **мутабельного** `struct`, то при вызове его метода/свойства (если член не помечен как не-мутирующий) компилятор делает **защитную копию** struct'а целиком и вызывает член на копии — чтобы readonly-поле гарантированно не изменилось. Для крупных struct'ов в горячем пути это ощутимая скрытая стоимость.
>     
>     csharp
>     
>     ```csharp
>     readonly Matrix4x4 _m;          // большой mutable struct
>     var d = _m.GetDeterminant();    // если член не readonly → копия _m целиком перед вызовом
>     ```
>     
>     Лечится `readonly struct` или `readonly`-членами (C# 8). Подробно — в главе о value types.

---

### 3. Versioning hazard — почему public const опасен

Поскольку `const` встраивается в IL **потребителя**, значение фиксируется в момент компиляции потребителя, а не владельца. Замена одной только библиотеки не меняет уже запечённый литерал.

![[Pasted image 20260619080911.png]]

**Вывод.** `public const` безопасен только для значений, которые не изменятся **никогда** (математические константы, зафиксированные протоколом величины). Всё, что «константа сегодня, но теоретически может поменяться» — `public static readonly`: значение читается из поля владельца в рантайме, и подмена библиотеки обновляет его без пересборки потребителей.

> 🔬 **Цена обходного пути.** Формально `static readonly` стоит один `ldsfld` против inline-литерала у `const`. На практике разрыв близок к нулю: RyuJIT в современных .NET способен промоутить `static readonly`-поля **примитивных** типов почти до констант после выполнения static-ctor (значение уже не изменится в пределах процесса). **VERIFY:** точные условия (версия .NET, tiered compilation / ReadyToRun) — перепроверить перед тем, как утверждать на собесе.

---

### 4. Decision matrix

|Критерий|`const`|`static readonly`|instance `readonly`|
|---|---|---|---|
|Значение известно при компиляции|**обязательно**|не требуется|не требуется|
|Когда вычисляется|компилятором (inline)|1 раз, при инициализации типа|на каждый экземпляр|
|Допустимые типы|примитив / `string` / `enum` / `null`|любой|любой|
|Своё значение на экземпляр|— (всегда static)|нет|**да**|
|Можно в compile-time контексте (атрибут / `case` / optional-параметр)|**да**|нет|нет|
|Безопасно в публичном API|только для вечных значений|да|да|
|Доступ в IL|литерал (`ldc`/`ldstr`), без load|`ldsfld`|`ldfld`|
|Можно как local|да|—|—|

**Rule of thumb:** нужен в compile-time контексте **или** примитив + по-настоящему вечное значение → `const`. Иначе общее для типа → `static readonly`. Зависит от экземпляра → instance `readonly`.

---

### 5. 🔬 Родственники keyword readonly (на синьора)

Это **другие применения** того же слова — полный разбор в главе о value types; здесь — чтобы не спутали на собесе:

- **`readonly struct`** (C# 7.2) — весь struct неизменяем, все поля readonly; компилятор не делает defensive copies при вызове членов.
- **`readonly` members** (C# 8) — пометить отдельный метод/свойство struct'а как не-мутирующий, чтобы убрать копию точечно.
- **`ref readonly` returns / `in` parameters** (C# 7.2) — передача/возврат по ссылке без права мутации; избегаем копирования больших struct'ов.
- **`init` accessors** (C# 9) — **не родственник**, а контраст: иммутабельность через сеттер, работающий только во время инициализации объекта. В отличие от `readonly`, позволяет задавать значение в object initializer, а не только в конструкторе.

---

### 6. Interview layer

#### Вопросы по частоте / уровню

1. **[очень часто]** Разница `const` и `readonly`?
2. **[часто]** `const` неявно `static`? Можно ли `static const`?
3. **[часто]** Какие типы допустимы для `const`?
4. **[часто, trap]** `readonly`-поле — массив/`List`: можно менять элементы?
5. **[mid+]** Чем опасен `public const` в библиотеке?
6. **[mid+]** `const` vs `static readonly` для публичной константы — что выбрать и почему?
7. **[mid+]** Когда `const` **обязателен**, а не просто предпочтителен?
8. **[🔬]** Когда инициализируется `static readonly`?
9. **[🔬]** Что такое defensive copy у `readonly`-поля struct'а?
10. **[adjacent]** Зачем `readonly struct`?

#### Follow-up деревья (Q → follow-up → trap)

- **Q1** Разница? → _когда вычисляется readonly?_ → **trap:** чем опасен const в публичной библиотеке? (versioning)
- **Q2** Неявно static? → _почему нельзя `static const`?_ → **trap:** где физически живёт значение const? (literal в метаданных, не поле)
- **Q3** Какие типы? → _почему нельзя `const DateTime`?_ (значение не compile-time) → **trap:** а `const decimal` как, если decimal не IL-литерал? (`DecimalConstantAttribute`)
- **Q4** Массив-поле? → _что именно фиксирует readonly?_ (binding, не объект) → **trap:** как сделать по-настоящему immutable? (`ImmutableArray`; `IReadOnlyList` только прячет API)
- **Q5** Опасность const? → _как починить?_ (`static readonly`) → **trap:** какой ценой? (`ldsfld` вместо литерала — почти нулевой, JIT-промоут примитивов)
- **Q7** Когда const обязателен? → _приведи место, где static readonly не скомпилируется_ → **trap:** почему атрибут требует именно const? (аргументы атрибутов вшиваются в метаданные на этапе компиляции)
- **Q8** Тайминг static readonly? → **trap:** `beforefieldinit` и почему явный static-ctor меняет момент инициализации
- **Q9** Defensive copy? → **trap:** как убрать (`readonly struct` / `readonly`-члены)

#### Частые слабые / неверные ответы

- _«const и readonly — одно и то же, просто readonly можно в конструкторе»_ — упускает compile-time vs runtime, versioning и static-природу const.
- _«readonly делает объект неизменяемым»_ — нет, фиксирует только binding; `readonly List` всё ещё наполняется.
- _«const быстрее, поэтому всегда лучше»_ — игнорирует versioning hazard; в публичном API это анти-паттерн.
- _«readonly можно присвоить один раз где угодно»_ — нет, только инициализатор / тело ctor / `init`; из helper-метода → `CS0191`.
- _«const — только ради скорости»_ — упускает, что в атрибутах, `case` и optional-параметрах он **обязателен**.

#### Tight model answer

**RU.** «`const` — compile-time константа: значение должно быть известно при компиляции и встраивается компилятором прямо в IL потребителя. Такое поле неявно `static` и допускает только примитивы, `string`, `enum` или `null`. `readonly` — runtime-константа: настоящее поле, присваиваемое один раз — в объявлении или в конструкторе — и читаемое в рантайме, поэтому может быть вычисляемым (`Guid.NewGuid()`), любого типа и per-instance. Практическое следствие: для публичного API я беру `static readonly`, потому что `const` запекается в код клиента, и смена значения требует пересборки всех потребителей. И `readonly` фиксирует только ссылку, а не сам объект — `readonly List` всё ещё можно наполнять».

_Senior add-on:_ «`const` всё же обязателен в compile-time местах — аргументы атрибутов, метки `case`, значения optional-параметров; `static readonly` там не компилируется. А на struct'ах `readonly`-поле мутабельного struct'а порождает defensive copy при вызове его членов — поэтому крупные struct-поля стоит делать `readonly struct` или помечать члены `readonly`».

**EN.** “`const` is a compile-time constant: the value must be known at compile time and the compiler inlines it into the consumer's IL. A const field is implicitly `static` and limited to primitives, `string`, `enum`, or `null`. `readonly` is a runtime constant — a real field assigned once, at the declaration or in a constructor, and read at runtime, so it can be computed (`Guid.NewGuid()`), any type, and per-instance. Practical consequence: for a public API I prefer `static readonly`, because a `const` is baked into the consumer's compiled code, so changing it requires recompiling every consumer. Also `readonly` only freezes the binding, not the object — a `readonly List` can still be mutated.”

_Senior add-on:_ “That said, `const` is mandatory in compile-time positions — attribute arguments, `case` labels, optional-parameter defaults — where `static readonly` won't compile. And on structs, a `readonly` field of a mutable struct triggers a defensive copy whenever you call its members, so large struct fields should be `readonly struct` or expose `readonly` members.”