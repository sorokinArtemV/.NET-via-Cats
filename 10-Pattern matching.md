### Зачем эта глава

Слово-ловушка — «pattern matching это просто `is Type x`». На деле с C# 8 по 11 это вырос целый язык сопоставления: relational- и logical-комбинаторы, property- и positional-паттерны, list-паттерны со срезами, switch-выражения с проверкой полноты. Зачем он появился: чтобы заменить лестницы `if/else` с ручными кастами и булевыми флагами на **декларативную таблицу «форма значения → результат»**, где полноту проверяет компилятор. На собесе бьют по этому, потому что это современный идиом, а ловушки (`is not null`, exhaustiveness, порядок arm'ов) сразу показывают глубину.

### Mental model

**Паттерн — это композиция проверки + извлечения в одном выражении. `switch`-выражение — исчерпывающая диспетчеризация значения в значение.**

Что в каком C# появилось (пригодится для version-follow-up):

|Паттерн|C#|
|---|---|
|constant, type, declaration (`is T x`)|7.0|
|property, positional, tuple, `var`, switch expression, discard `_`|8.0|
|relational (`< > <= >=`), logical (`and`/`or`/`not`)|9.0|
|extended property (вложенный `{ A.B: … }`)|10|
|list, slice (`[…]`, `..`)|11|

### 1. Семейства паттернов

**Constant + type (C# 7).** База: `x is null`, `x is 42`, `x is Point p`.

**Relational (C# 9)** — сравнения с константой, чистые диапазоны без `&&`:

csharp

```csharp
var grade = score switch { >= 90 => "A", >= 80 => "B", >= 70 => "C", _ => "F" };
```

**Logical: and / or / not (C# 9)** — комбинируют **паттерны**, а не булевы выражения:

csharp

```csharp
bool working = age is >= 18 and < 65;
if (cmd is "yes" or "y" or "ok") ...
if (node is not null) ...          // см. ловушку в §5
```

**Property-паттерн (C# 8), вложенный (C# 10).** `{ }` сам по себе значит «не null»:

csharp

```csharp
if (person is { Age: >= 18, Name.Length: > 0 }) ...   // Name.Length — extended pattern, C# 10
var shipping = order switch
{
    { Total: > 100 }             => 0m,
    { Customer.IsPremium: true } => 0m,
    _                            => 9.99m,
};
```

**Positional-паттерн / деконструкция (C# 8).** Нужен `Deconstruct` (есть у record и tuple):

csharp

```csharp
// record Point(int X, int Y);
var where = point switch
{
    (0, 0)     => "origin",
    (var x, 0) => "on X axis",
    (0, _)     => "on Y axis",
    (> 0, > 0) => "Q1",
    _          => "elsewhere",
};
```

**`var`-паттерн** — матчит всегда и связывает; полезен с `when`:

csharp

```csharp
var label = value switch
{
    int n when n % 2 == 0                    => "even",
    int n                                    => "odd",
    string { Length: var len } when len > 10 => "long string",
    _                                        => "other",
};
```

> **🔬 Глубже.** Вложенные property-паттерны **null-безопасны**: если `Name` равно `null`, паттерн `Name.Length: > 0` просто не матчится, а не кидает NRE. И `{ }` — это «любой не-null»; `obj is { }` фактически эквивалентно `obj is not null` (с поправкой на nullable value types).

### 2. List-паттерны + срез `..` (C# 11)

csharp

```csharp
var desc = arr switch
{
    []                        => "empty",
    [var only]                => $"one: {only}",
    [var first, .., var last] => $"{first}…{last}",
    [_, _, ..]                => "two or more",
};
if (path is ['/', ..]) ...    // начинается со слэша
```

`..` (slice pattern) поглощает любое число элементов в середине. Он может быть и **связывающим** (`[var head, .. var rest]`), но связывающий срез требует sliceable-тип (массив, `Span<T>`); у `List<T>` его нет — хотя обычный list-паттерн и несвязывающий `..` на `List<T>` работают. Требования к типу — см. §5.

### 3. switch-выражения

`switch`-выражение превращает значение в значение через arm'ы `паттерн => результат`. Три вещи, которые надо знать на мид+:

- **`when`-guard** — произвольное условие поверх паттерна: `int n when n % 2 == 0 => …`.
- **Полнота (exhaustiveness).** Если покрыты не все случаи, компилятор предупреждает (CS8509), а непокрытое значение в рантайме кидает `SwitchExpressionException`. `_` (discard) — явный arm «всё остальное».
- **Порядок arm'ов.** Проверяются сверху вниз, побеждает **первый** подходящий. Широкий паттерн выше узкого делает узкий недостижимым (предупреждение CS8510). Специфичное — выше общего.

### 4. Worked example: таблица переходов автомата

Любимый на собесах кейс — конечный автомат одной таблицей на tuple-паттернах, без вложенных `switch`:

csharp

```csharp
State Next(State s, Event e) => (s, e) switch
{
    (State.Idle,    Event.Start) => State.Running,
    (State.Running, Event.Pause) => State.Paused,
    (State.Paused,  Event.Start) => State.Running,
    (_,             Event.Stop)  => State.Idle,
    _ => throw new InvalidOperationException($"{s} не принимает {e}"),
};
```

Каждая строка — переход; `_` ловит недопустимые комбинации. Читается как сама таблица состояний.

### 5. Три ловушки

**1. `is not null` ≠ `!= null`.** Если у типа перегружен `operator !=`, то `obj != null` вызовет **пользовательский** оператор — он может соврать, кинуть исключение, иметь сайд-эффект. `obj is not null` — чистая проверка на null на уровне языка, она **не зовёт** пользовательские операторы. Поэтому в библиотечном и обобщённом коде `is not null` безопаснее.

**2. Порядок arm'ов.** Первый матч побеждает; пересекающиеся паттерны в неправильном порядке делают часть кода недостижимой (CS8510). Узкие случаи — выше.

**3. Полнота.** `switch`-выражение требует исчерпывающести: не покрыл всё — предупреждение CS8509 и `SwitchExpressionException` в рантайме. `_` закрывает полноту явно, а не «на всякий случай».

> **🔬 Глубже: во что это компилируется.** `switch` **по типам** обычно лоуэрится в цепочку проверок типа (`isinst`) с условными переходами — это последовательность, а не jump table. Jump table / хеш-диспетчеризация получается у `switch` по **целочисленным/enum-константам** и строкам. Поэтому длинный type-switch — это серия проверок, и порядок arm'ов влияет и на корректность (первый матч), и на стоимость. **Требования list-паттерна:** обычный list-паттерн (включая несвязывающий `..`) требует счётной длины (`Length`/`Count`) + индексатора — массивы, `List<T>`, `Span<T>`, `string`. **Связывающий** срез (`.. var x`) дополнительно требует sliceable (`Slice` или индексатор по `Range`) — массивы и `Span<T>` подходят, а `List<T>` — нет.

### Interview layer

Вопросы по убыванию частоты. Формат: вопрос → likely follow-up → trap; затем shallow-ответ (почему плох) и tight model answer (RU + EN).

**Q1 (часто). Чем `is not null` лучше `!= null`?**  
→ trap: а если у типа перегружен `==`/`!=`?

- _Shallow:_ «это одно и то же, просто синтаксис». Неверно при перегруженных операторах.
- _RU:_ «При перегруженном `operator !=` выражение `!= null` вызовет пользовательский оператор — он может соврать, кинуть, иметь сайд-эффект. `is not null` — чистая языковая проверка на null, не зовёт пользовательские операторы. Поэтому в библиотечном/обобщённом коде безопаснее `is not null`.»
- _EN:_ «With an overloaded `operator !=`, `!= null` invokes the user operator — which can lie, throw, or have side effects. `is not null` is a language-level null check that never calls user operators, so it's safer in library/generic code.»

**Q2 (часто). Что выведет `switch` с пересекающимися паттернами?**

- _RU:_ «Arm'ы проверяются сверху вниз, побеждает первый подходящий. Широкий паттерн выше узкого делает узкий недостижимым (предупреждение CS8510). Специфичные случаи ставят выше общих.»
- _EN:_ «Arms are checked top-to-bottom; the first match wins. A broad pattern above a narrow one makes the narrow one unreachable (warning CS8510). Put specific cases above general ones.»

**Q3 (часто). Зачем `_` в switch-выражении?**

- _RU:_ «switch-выражение требует исчерпывающести: не покрыл все случаи — предупреждение CS8509, а непокрытое значение в рантайме кидает `SwitchExpressionException`. `_` — явный arm "всё остальное", закрывающий полноту.»
- _EN:_ «A switch expression must be exhaustive: miss a case and you get warning CS8509, and an uncovered value throws `SwitchExpressionException` at runtime. `_` is the explicit "everything else" arm that closes exhaustiveness.»

**Q4 (mid). Как развернуть массив через list-паттерн?**

- _RU:_ «`arr is [var first, .., var last]` связывает первый и последний, `..` поглощает середину. Пустой — `[]`, один — `[var only]`, "два и больше" — `[_, _, ..]`.»
- _EN:_ «`arr is [var first, .., var last]` binds first and last, `..` swallows the middle. Empty is `[]`, single is `[var only]`, "two or more" is `[_, _, ..]`.»

**Q5 (mid+). Что нужно типу для positional- и list-паттернов?**

- _RU:_ «Positional-паттерн — `Deconstruct` (или быть record/tuple, у них он есть). List-паттерн (с несвязывающим `..`) — счётная длина (`Length`/`Count`) + индексатор: массивы, `List<T>`, `Span<T>`. **Связывающий** срез (`.. var x`) дополнительно требует sliceable (`Slice`/индексатор по `Range`) — массивы и `Span<T>` подходят, `List<T>` нет.»
- _EN:_ «A positional pattern needs `Deconstruct` (or to be a record/tuple, which have one). A list pattern (with a non-binding `..`) needs a countable length (`Length`/`Count`) plus an indexer: arrays, `List<T>`, `Span<T>`. A **binding** slice (`.. var x`) additionally needs slicing (`Slice` or a `Range` indexer) — arrays and `Span<T>` qualify, `List<T>` does not.»

**Q6 (mid+). relational + logical: задай диапазон без `&&`.**

- _RU:_ «`temp is >= 0 and <= 100` — диапазон без `&&` и без повторного упоминания переменной; в switch — `>= 0 and < 60 => "cold"`. `and`/`or`/`not` комбинируют паттерны, не булевы выражения.»
- _EN:_ «`temp is >= 0 and <= 100` — a range without `&&` and without repeating the variable; in a switch, `>= 0 and < 60 => "cold"`. The `and`/`or`/`not` combine patterns, not boolean expressions.»

**Q7 (🔬). Во что компилируется `switch` по типу?**

- _RU:_ «Обычно в цепочку проверок типа (`isinst`) с условными переходами — не jump table. Jump table / хеш получается у switch по целочисленным/enum-константам и строкам. Поэтому длинный type-switch — серия проверок, и порядок arm'ов влияет и на корректность, и на стоимость.»
- _EN:_ «Usually into a chain of type tests (`isinst`) with conditional branches — not a jump table. Jump tables/hashing appear for switches over integral/enum constants and strings. So a long type switch is a sequence of checks, and arm order affects both correctness and cost.»