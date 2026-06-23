
### Зачем это вообще

До enum «закрытый набор вариантов» кодировали либо «магическими» числами (`if (status == 2)`), либо строковыми константами (`"active"`). Оба способа теряют главное: компилятор не знает, что `2` и `"active"` — из одного набора. В метод `SetStatus(int)` можно передать `42`, в сравнение — опечатку `"actve"`, и всё это всплывёт только в рантайме. `enum` собирает варианты под одно имя типа: появляется группировка, IntelliSense предлагает только валидные имена, а сигнатура `SetStatus(OrderStatus)` сама документирует, что сюда идёт _только_ статус. Цена — иллюзия полной безопасности: под капотом это всё равно integer, и `(OrderStatus)42` пройдёт.

### Mental model

> **`enum` — это поименованные константы поверх целочисленного типа. Type safety здесь только на этапе компиляции по _имени_; в рантайме это голый integer, и любое число underlying-типа в переменную влезает.**

Веди ответ с обеих половин сразу: первая показывает пользу, вторая закрывает главный follow-up («а это правда безопасно?») ещё до того, как его зададут.

### Как это работает

#### Базовый синтаксис и нумерация

csharp

```csharp
enum OrderStatus       // underlying type по умолчанию — int
{
    Pending,           // 0
    Paid,              // 1
    Shipped,           // 2
    Cancelled = 10,    // можно задать вручную
    Refunded           // 11 — продолжает от предыдущего
}
```

Нумерация идёт с `0` и автоинкрементится. Значения можно задавать вручную, оставлять дыры и даже назначать **два имени на одно значение** (алиасы) — это легально.

#### Underlying types — какие бывают

Часто спрашивают прямо. Базовый тип меняется через `: Type`:

csharp

```csharp
enum Priority : byte { Low, Normal, High }
```

|Тип|Размер|Диапазон|Когда брать|
|---|---|---|---|
|`byte`/`sbyte`|1 B|0..255 / −128..127|Память важна: enum в больших массивах/структурах, сетевые протоколы, interop|
|`short`/`ushort`|2 B|±32k / 0..65k|Тот же мотив, нужен диапазон шире byte|
|`int`|4 B|±2.1 млрд|**По умолчанию.** Бери, если нет причин не брать|
|`uint`|4 B|0..4.2 млрд|Interop с unsigned-кодом|
|`long`/`ulong`|8 B|очень большой|`[Flags]` с >32 флагами (нужно >32 бит)|

**Ловушка-вопрос: «может ли enum быть на основе `string`?»** — Нет. В C# underlying type обязан быть одним из **integral**-типов выше. `char`, `bool`, `float`, `double`, `decimal`, `string` — **запрещены** (в отличие от TypeScript/Java, где «string enum» существует — оттуда и путаница). Строковое поведение в C# даёт `ToString()`, а не базовый тип.

#### Конверсии и `default`

csharp

```csharp
int n = (int)OrderStatus.Shipped;      // 2 — нужен явный cast
var s = (OrderStatus)2;                // Shipped — тоже явный
OrderStatus z = 0;                     // ОК: литерал 0 неявно конвертируется
// OrderStatus x = 1;                  // ошибка компиляции: 1 нужен cast
```

`0` — особенный: только литеральный `0` неявно приводится к любому enum. Отсюда же `default(OrderStatus) == 0`. **Ловушка:** дефолт всегда `0`, _даже если члена со значением 0 нет_ — тогда `default` даёт значение без имени. Поэтому первый член (значение 0) стоит делать осмысленным: `None`, `Unknown`, `Pending`.

#### `[Flags]` — перечисление как битовая маска

**Зачем.** Когда варианты не взаимоисключающие, а комбинируемые (права, опции, наборы возможностей), нужно хранить «несколько сразу» в одном значении. Решение — отдать каждому флагу **отдельный бит**: значения = степени двойки (1, 2, 4, 8…), тогда биты не пересекаются, и комбинация — это просто `OR`.

csharp

```csharp
[Flags]
enum FileAccess
{
    None      = 0,                              // 0000 — обязательный «пустой» член
    Read      = 1,                              // 0001
    Write     = 2,                              // 0010
    Execute   = 4,                              // 0100
    Delete    = 8,                              // 1000
    ReadWrite = Read | Write,                   // 0011 — составной член, легально
    All       = Read | Write | Execute | Delete // 1111
}
```

`None = 0` обязателен по конвенции: это нейтральный элемент для `OR` и осмысленный `default`. Составные члены (`ReadWrite`, `All`) — частый приём, чтобы не собирать ходовые маски руками.

**Четыре операции над маской** — все это обычные битовые операции, `[Flags]` для них не нужен:

|Действие|Код|Механика|
|---|---|---|
|**set** (добавить)|`rights \|= Write;`|`OR` — выставить бит|
|**clear** (снять)|`rights &= ~Read;`|`AND` с инверсией — погасить бит|
|**toggle** (переключить)|`rights ^= Execute;`|`XOR` — инвертировать бит|
|**check** (проверить)|`rights.HasFlag(Read)` или `(rights & Read) == Read`|сравнить бит|

![[Pasted image 20260623105605.png]]

**`None` и `HasFlag` — ловушка.** `x.HasFlag(None)` всегда `true`, потому что `(x & 0) == 0` истинно для любого `x`. Проверять «маска пуста» через `HasFlag(None)` бессмысленно — сравнивай напрямую: `rights == FileAccess.None`.

**`HasFlag` vs битовый тест.** Для одного флага они эквивалентны. `(rights & Read) == Read` исторически быстрее (без бокса), но с .NET Core 2.1 `HasFlag` стал JIT intrinsic, и разница исчезла (детали в 🔬). Битовый тест нужен там, где `HasFlag` не умеет — проверка «есть хотя бы один из набора»:

csharp

```csharp
bool anyWriteOrExec = (rights & (Write | Execute)) != 0;   // HasFlag так не умеет
```

**`[Flags]` влияет только на `ToString()`/`Parse()`.** Сам атрибут не «включает» битовые операции — они работают на любом enum. Что он реально меняет — строковый round-trip:

csharp

```csharp
var r = FileAccess.Read | FileAccess.Write;
r.ToString();                            // с [Flags]:  "Read, Write"
                                         // без [Flags]: "3"  (нет члена со значением 3)
Enum.Parse<FileAccess>("Read, Write");   // с [Flags] разбирает комбинацию обратно
```

Без `[Flags]` код не ломается — деградирует только читаемость комбинаций (печатаются числом).

**Больше 32 флагов** — переключай underlying type на `long`/`ulong`: у `int` всего 32 бита.

csharp

```csharp
[Flags] enum Caps : ulong { F0 = 1, F1 = 2, /* … */ F63 = 1UL << 63 }
```

#### Enum как value type — мелочи, на которых валятся

- **Nullable.** Enum — value type, `null` ему не присвоить. Нужно состояние «значения нет» — бери `OrderStatus?` (`Nullable<OrderStatus>`), а не выдумывай для этого член. (Для `[Flags]` член `None` всё равно нужен, но по другой причине — как нейтральный элемент маски, а не как «null».)
- **Расширение методами.** Добавить метод _внутрь_ enum нельзя, но extension methods работают и это идиоматично — навесить поведение, не превращая enum в класс:

csharp

```csharp
public static bool IsTerminal(this OrderStatus s)
    => s is OrderStatus.Cancelled or OrderStatus.Refunded;
```

#### Иллюзия type safety и валидация

Главный mid+/senior follow-up: **enum гарантирует только _тип_, но не _валидность значения_.** Под капотом это integer, поэтому в переменную влезает любое число underlying-типа — даже то, которому не соответствует ни один член:

csharp

```csharp
var s = (OrderStatus)999;     // компилируется, НЕ бросает
Console.WriteLine(s);          // "999"
```

Единственная встроенная проверка — `Enum.IsDefined`:

![[Pasted image 20260623105615.png]]

csharp

```csharp
Enum.IsDefined(typeof(OrderStatus), 999);   // false
Enum.IsDefined(typeof(OrderStatus), 2);     // true
```

**Подвох с `IsDefined` и `[Flags]`:** на комбинациях он возвращает `false`, потому что `Read | Write` (= 3) — не объявленный член:

csharp

```csharp
Enum.IsDefined(typeof(FileAccess), FileAccess.Read | FileAccess.Write);  // false!
```

Для флагов `IsDefined` не годится — валидируй маской: `(value & ~allValid) == 0`.

#### API `System.Enum` (коротко)

Это статические методы на `System.Enum`, не на конкретном типе enum:

csharp

```csharp
Enum.TryParse<OrderStatus>("Shipped", out var st);  // безопасный разбор (.NET FW 4.0+)
Enum.Parse<OrderStatus>("Shipped");                 // бросит при неудаче (generic — .NET Core 2.0+)
Enum.GetValues<OrderStatus>();                       // все значения (generic — .NET 5+)
Enum.GetNames<OrderStatus>();                        // все имена (generic — .NET 5+)
Enum.GetUnderlyingType(typeof(Priority));            // typeof(byte)
```

До generic-перегрузок (.NET 5) использовались негенерик-версии с `typeof` и боксящим `Array` — на это можно нарваться в старых таргетах и в `netstandard2.0`.

### 🔬 Глубже / на синьора

**Представление в рантайме — ноль overhead.** Enum существует в основном в метаданных и как compile-time константы. В рантайме переменная/поле enum — это **просто его underlying integer**, без объекта и без аллокации на значение. `switch` по enum компилируется в jump table (как `switch` по int), а не в цепочку сравнений. Стоимости относительно «голого int» нет.

**Boxing.** Enum — value type, поэтому приведение к `object`, `Enum` или интерфейсу (`IComparable`, `IConvertible`) **боксит**. Исторически это било по `HasFlag`: он принимает параметр типа `Enum` и боксил оба операнда → аллокации в горячем цикле. **С .NET Core 2.1** JIT распознаёт `Enum.HasFlag` как intrinsic и сводит вызов к `(value & flag) == flag` без бокса — но только когда оба операнда одного enum-типа. **На .NET Framework `HasFlag` по-прежнему боксит**, и в перф-критичном коде там предпочитают ручную битовую проверку (≈ вдвое быстрее).

**`switch` не проверяет полноту.** C# не требует покрыть все члены enum:

csharp

```csharp
switch (status) {            // statement: пропущенный член — молча проваливается
    case OrderStatus.Paid: ...; break;
}
```

`switch`-**statement** на непокрытое значение просто ничего не делает. `switch`-**выражение** (C# 8+) выдаёт warning о неисчерпывающести и бросает `SwitchExpressionException` в рантайме на непокрытом значении. Ни то, ни другое не ошибка компиляции — «закрытость» набора enum иллюзорна и тут.

**Cross-assembly versioning — как у `const`.** Члены enum — compile-time константы. Когда сборка B пишет `OrderStatus.Paid`, в её IL «впекается» литерал `1`, а не ссылка. Перенумеруешь enum в сборке A (вставишь член в середину) — B продолжит видеть старые числа, пока её не перекомпилируют, и сломаются сериализованные/персистентные данные, где статус хранился как int. Правило: задавай значения явно, **только дописывай в конец**, не переставляй и не вставляй в середину. (Тот же механизм разбирали в главе про `const`/`readonly`.)

### Interview layer

#### Вопросы по частоте

**1. Что такое enum и зачем он нужен?** _(база)_  
→ follow-up: какой underlying type по умолчанию и можно ли поменять?  
→ **trap:** а может ли enum быть на основе `string`? _(нет — только integral-типы; «string enum» — это из TS/Java)_

**2. Type-safe ли enum?** _(mid+)_  
→ follow-up: что вернёт `(OrderStatus)999`? _(валидное значение, без исключения, ToString → "999")_  
→ **trap:** как тогда провалидировать вход? _(`Enum.IsDefined`; и его дыра на `[Flags]`-комбинациях → маска)_

**3. Зачем `[Flags]` и как он работает?** _(mid+)_  
→ follow-up: что именно делает сам атрибут? _(только `ToString()`/`Parse()`)_  
→ **trap:** а `|`/`&` работают без `[Flags]`? _(да, всегда — это операции над integer)_  
→ **trap-2:** как снять флаг и как переключить? _(`&= ~flag` снять, `^= flag` переключить)_

**4. Что такое `default` для enum, если члена со значением 0 нет?** _(mid+ trap)_  
→ `0`, приведённый к enum, — значение без имени. Поэтому 0 резервируют под `None`/`Unknown`.

**5. 🔬 Боксится ли enum? Сколько стоит `HasFlag`?**  
→ боксится при приведении к `object`/интерфейсу; `HasFlag` боксил оба операнда, с .NET Core 2.1 — JIT intrinsic без бокса, на .NET Framework по-прежнему боксит.

**6. 🔬 Что будет, если переставить члены enum между версиями сборки?**  
→ значения «впечены» как константы → рассинхрон с непересобранными потребителями и порча сериализованных данных.

#### Слабые ответы и почему проваливаются

- **«Enum — это type-safe набор констант, ошибиться нельзя».** Полу-правда: safety только по имени на компиляции; `(E)999` проходит. Интервьюер сразу спросит про невалидное значение — и ответ рассыплется.
- **«`[Flags]` позволяет комбинировать значения битовыми операциями».** Неверная причинность: операции и так работают; `[Flags]` лишь чинит строковое представление. Классический маркер поверхностного знания.
- **«Underlying type всегда int».** По умолчанию — да, но вопрос проверяет, знаешь ли ты, что его меняют и зачем (память, interop, >32 флага).

#### Модельный ответ

**RU:** «`enum` — это набор поименованных констант поверх integral-типа (по умолчанию `int`, можно `byte`…`ulong`, но не `string`). Он даёт читаемость и проверку _по имени_ на компиляции, но это не полноценная type safety: под капотом integer, и `(MyEnum)999` — валидное значение без исключения, поэтому внешний вход валидируют через `Enum.IsDefined`. Для битовых масок ставят `[Flags]` и степени двойки — но сам атрибут влияет только на `ToString()`/`Parse()`, операции `|`/`&` работают и без него.»  
_(senior add-on:_ «В рантайме enum — это голый integer без overhead; стоит помнить про boxing при приведении к `object`/интерфейсу и про то, что значения членов — compile-time константы, поэтому их нельзя переставлять между версиями сборки.»_)_

**EN:** “An `enum` is a set of named constants over an integral type — `int` by default, can be `byte`…`ulong`, but never `string`. It gives readability and compile-time checking _by name_, but it isn’t full type safety: under the hood it’s an integer, so `(MyEnum)999` is a valid value with no exception — that’s why external input is validated with `Enum.IsDefined`. For bitmasks you add `[Flags]` and powers of two — but the attribute itself only affects `ToString()`/`Parse()`; the `|`/`&` operators work without it.”  
_(senior add-on:_ “At runtime an enum is just its underlying integer with no overhead; watch for boxing when casting to `object`/an interface, and remember member values are compile-time constants, so you must never reorder them across assembly versions.”_)_