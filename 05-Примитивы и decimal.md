Примитивные типы — это базовые «атомы» данных, встроенные прямо в язык: целые числа, дробные, логическое значение, символ. Они не собраны из других типов — из них собирается всё остальное. Язык даёт им ключевые слова (`int`, `bool`, `char`) и литералы (`42`, `true`, `'a'`), а рантайм работает с ними машинными инструкциями напрямую, без накладных расходов на вызов методов.

Звучит как самая простая тема в C# — и именно поэтому на ней спотыкаются. Слово «примитив» перегружено: C# и CLR вкладывают в него **разное**, и `decimal` сидит ровно на этом разломе — отсюда любимый вопрос «является ли `decimal` примитивом». Второй вечный сюжет — точность дробных типов: почему `0.1 + 0.2 != 0.3` и когда брать `decimal` вместо `double`.

Эта глава — не справочник по каждому типу (диапазоны, суффиксы, overflow, конверсии идут дальше по главам). Её задача — дать **карту понятий** («что вообще значит примитив») и закрыть **триаду точности** `float` / `double` / `decimal`: follow-up'ы интервьюера бьют именно туда.

### Mental model

«Примитив» — это три почти-вложенных множества: C# `built-in` ⊃ C# `simple`, а CLR `primitive` ≈ те же simple-типы **минус `decimal`, плюс `IntPtr`/`UIntPtr`**. Проще запомнить через исключения: `decimal` — simple, но не CLR-примитив; `string` — built-in, но не value type; `IntPtr`/`UIntPtr` — CLR-примитивы, но не simple.

### How it works

#### Три слоя терминологии

Три разных понятия, которые на собесе постоянно сваливают в одно:

**1. Built-in type (C#)** — ключевые слова языка, которые Roslyn разворачивает в типы BCL: `int` → `System.Int32`, `bool` → `System.Boolean`, `string` → `System.String`, `object` → `System.Object`. После компиляции `int` и `System.Int32` неразличимы. Важно: сюда входят и **reference**-типы (`string`, `object`), не только value.

**2. Simple type (C#)** — подмножество built-in: предопределённые value-типы, у которых есть **литералы**: `sbyte, byte, short, ushort, int, uint, long, ulong, char, float, double, decimal, bool`. Ключевой момент — **`decimal` здесь есть.** Именно поэтому он «ощущается» примитивом.

**3. CLR primitive type** — типы, которые рантайм обрабатывает специальными IL-инструкциями (`ldc.i4`, `add`, `conv.*`) и помечает в метаданных. Их 14: `Boolean, Byte, SByte, Int16, UInt16, Int32, UInt32, Int64, UInt64, Char, Single, Double, IntPtr, UIntPtr`. **`decimal` и `string` сюда не входят.** Проверяется рефлексией: `typeof(decimal).IsPrimitive == false`, `typeof(int).IsPrimitive == true`.

Эти два множества — `simple` и `primitive` — **пересекаются, а не вкладываются**: общие у них 12 типов; только в `simple` — `decimal`; только в `primitive` — `IntPtr`/`UIntPtr` (у них нет литерала, поэтому они не simple). Развязка вопроса «`decimal` — примитив?»: **да** как C# simple/built-in type, **нет** как CLR primitive. На уровне IL `int + int` компилируется в инструкцию `add`, а `decimal + decimal` — в `call System.Decimal::op_Addition`, то есть обычный вызов метода.

![[Pasted image 20260622113051.png]]
#### Числовая семья — ориентир

Полную спеку (диапазоны, суффиксы, overflow, конверсии) разбираем в следующих главах; здесь — только опора по размеру и назначению.

|Тип|System-тип|Размер|Назначение|
|---|---|---|---|
|`bool`|`Boolean`|1 байт|логический (**не** 1 бит)|
|`char`|`Char`|2 байта|один UTF-16 code unit|
|`byte`|`Byte`|1 байт|0..255|
|`short`|`Int16`|2 байта||
|`int`|`Int32`|4 байта|целочисленный «по умолчанию»|
|`long`|`Int64`|8 байт||
|`float`|`Single`|4 байта|binary floating-point|
|`double`|`Double`|8 байт|binary floating-point «по умолчанию»|
|`decimal`|`Decimal`|16 байт|base-10, точные десятичные|

#### Floating-point: почему `double` «врёт»

`float`/`double` — это **binary** floating-point (IEEE 754): число хранится как `mantissa × 2^exponent`. Дробь, конечная в десятичной записи, в двоичной может быть бесконечной — ровно как `1/3` бесконечна в десятичной. `0.1` в двоичной = `0.0001100110011…` (периодическая), поэтому в `double` лежит **ближайшее представимое** значение, чуть-чуть не `0.1`.

Отсюда классика:

csharp

```csharp
double a = 0.1 + 0.2;        // 0.30000000000000004
Console.WriteLine(a == 0.3); // False
```

Это не баг C# и не баг процессора — фундаментальное свойство представления. Поэтому `double` **нельзя сравнивать на `==`** (сравнивают с epsilon) и **нельзя считать деньги в `double`**.

#### `decimal`: точные десятичные

`decimal` хранит число в **base-10**: 96-битная целая мантисса + десятичный масштаб (scale) + знак. Значение = `mantissa × 10^(−scale)`. Поэтому `0.1` представляется **точно** (это `1 × 10^−1`), и для `decimal` справедливо `0.1m + 0.2m == 0.3m` → `true`.

Цена: ~28–29 значащих десятичных цифр, **меньший диапазон**, чем у `double`, 16 байт, и арифметика **софтверная** (нет аппаратной поддержки FPU) → в разы медленнее.

Правило выбора:

- **деньги, финансы, любая обязательная десятичная точность** → `decimal`;
- **наука, графика, физика, большие диапазоны, скорость** → `double`;
- `float` — когда память/пропускная способность важнее точности (GPU, большие массивы).

#### `char` — это не «символ» в полном смысле

`char` — 16-битный **UTF-16 code unit**, а не «Unicode-символ». Символы из BMP (`U+0000`..`U+FFFF`) — один `char`. Всё за пределами (эмодзи, редкие иероглифы, `U+10000`+) — **surrogate pair**, два `char`. Поэтому `"😀".Length == 2`, а не `1`.

#### Миф: «value type всегда на стеке»

Неверно. Value type хранится **там, где живёт его слот**: локальная переменная/параметр — обычно стек; поле класса — в куче внутри объекта; захваченная замыканием переменная — в куче; элемент массива — в куче; при boxing — в куче. «Value type = стек» — самая частая ошибка на собесе. _(Полный разбор stack/heap — в главе про value vs reference types.)_

### 🔬 Глубже / на синьора

**IL: примитив vs не-примитив.** Операторы примитивов — это IL-инструкции (`add`, `mul`, `clt`), без вызова метода. Операторы `decimal` — это `op_Addition`/`op_Multiply` и т.д., обычные статические методы на `System.Decimal`, компилируемые в `call`. Это и есть машинная разница «примитив / не примитив»: не «сахар», а другой путь исполнения.

**`Type.IsPrimitive`.** Возвращает `true` ровно для 14 типов выше — включая `IntPtr`/`UIntPtr`. А вот `decimal`, `string`, `DateTime` → `false`. Именно `IntPtr`/`UIntPtr` ломают «чистую вложенность»: они CLR primitives, но **не** C# simple types (нет литерала), поэтому в диаграмме держим их за скобками.

**Представление `decimal` детальнее.** 128 бит: 96 бит мантиссы (три `uint`) + знак и scale (0–28) в четвёртом слове. Два разных bit-pattern могут давать одно значение: `1.0m` и `1.00m` различаются по scale, но равны по `==`. Нормализация при операциях не гарантируется — `decimal` хранит «значащие нули».

**`const decimal`.** `const` допустим для всех simple-типов (`float`, `double`, `decimal`, `bool`…) плюс `string`. `decimal` особенный не тем, что его можно сделать `const`, а тем, что у него **нет** формы настоящей metadata-константы: компилятор кодирует значение через `DecimalConstantAttribute`, тогда как `const int`/`const double` — это реальные IL-литералы.

**`nint` / `nuint` (C# 9).** Built-in-алиасы для `IntPtr`/`UIntPtr` с целочисленной семантикой (native-sized int: 4 байта на x86, 8 на x64). Сами `IntPtr`/`UIntPtr` — CLR primitives; `nint`/`nuint` — языковая надстройка над ними. `typeof(nint).IsPrimitive == true` (по алиасу).

**`bool` в памяти.** `sizeof(bool) == 1` (байт), не бит. В IL `true` — это `ldc.i4.1`.

**Соседи, о которых спросят на синьора.** `Rune` (`System.Text.Rune`, .NET Core 3.0) — «правильный» Unicode code point поверх пары `char`, для корректной работы с эмодзи и суррогатами. `Half` (`System.Half`, binary16, .NET 5) — 16-битный IEEE-float для ML/интеропа, та же binary-неточность, что у `float`/`double`.

### Interview layer

---

**1. `decimal` — примитивный тип?** _(очень частый)_

Q → likely follow-up → trap:

- «В каком смысле примитив?» → три слоя.
- «Тогда почему он ведёт себя как примитив?» → потому что C# simple type (литерал `m`, операторы), **но** операторы — это method calls, а не IL-инструкции.

Слабый ответ: «да, примитив» или «нет, это struct». Оба фейлят, потому что не различают уровень C# и уровень CLR — а интервьюер копает именно туда.

Model answer (RU): «Зависит от уровня. В C# `decimal` — simple/built-in type: есть литерал `3.14m`, операторы, работает как обычный value-тип. Но на уровне CLR он **не** примитив: `typeof(decimal).IsPrimitive == false`, а арифметика компилируется в вызовы методов `System.Decimal` (`op_Addition`), не в IL-инструкции вроде `add`. Так что „примитив" — да в C#-смысле, нет в CLR-смысле.»

Model answer (EN): «Depends on the level. In C# `decimal` is a _simple/built-in_ type — it has the `m` literal and full operator support. But it's **not** a CLR primitive: `typeof(decimal).IsPrimitive` is `false`, and its arithmetic compiles to method calls on `System.Decimal` (`op_Addition`), not to IL instructions like `add`. So it's a primitive in the C# sense, not in the CLR sense.»

---

**2. Разница `float` / `double` / `decimal`, когда что?** _(частый)_

Q → follow-up → trap:

- «Почему нельзя считать деньги в `double`?»
- «А почему тогда не использовать `decimal` всегда?»

Слабый ответ: «`decimal` точнее» без механизма. Фейлит, потому что не объясняет _почему_ и не знает цену.

Model answer (RU): «`float`/`double` — binary floating-point (IEEE 754), быстрые, аппаратные, но не представляют десятичные дроби точно. `decimal` — base-10, точен для десятичных, но в разы медленнее (софтверный, без FPU), меньший диапазон, 16 байт. Деньги → `decimal`, наука/графика → `double`, `float` — когда память важнее точности.»

Model answer (EN): «`float`/`double` are binary IEEE-754 — fast, hardware-backed, but can't represent decimal fractions exactly. `decimal` is base-10 — exact for decimals, but several times slower (software, no FPU), smaller range, 16 bytes. Money → `decimal`, science/graphics → `double`, `float` when memory matters more than precision.»

---

**3. Почему `0.1 + 0.2 != 0.3`?** _(частый, проверяет механизм)_

Q → follow-up → trap:

- «Как тогда сравнивать `double`?» → через epsilon.
- «А в `decimal` это `true`? Почему?» → да, base-10, `0.1` точен.

Слабый ответ: «ошибка округления» — верно по слову, но без механизма не проходит follow-up.

Model answer (RU): «`double` хранит `mantissa × 2^exp`; `0.1` в двоичной — периодическая дробь, поэтому в `double` лежит ближайшее представимое, не ровно `0.1`. Сумма даёт `0.30000000000000004`. Сравнивать `double` надо через epsilon. В `decimal` — `true`, потому что он base-10 и `0.1` там точен.»

Model answer (EN): «`double` stores `mantissa × 2^exp`; `0.1` is a repeating fraction in binary, so `double` holds the nearest representable value, not exactly `0.1`. The sum is `0.30000000000000004`. Compare doubles with an epsilon. In `decimal` it's `true` — it's base-10, so `0.1` is exact.»

---

**4. `char` — сколько байт, что хранит?** _(средний)_

Q → follow-up → trap:

- «`"😀".Length`?» → `2`.
- «Почему 2, это же один символ?» → surrogate pair; `char` = UTF-16 code unit, не code point. _(Senior add-on: правильно итерировать через `Rune`.)_

Model answer (RU): «`char` — 2 байта, один UTF-16 code unit, не полноценный Unicode-символ. Код-поинты за пределами BMP (`U+10000`+) кодируются surrogate pair — двумя `char`. Поэтому `"😀".Length == 2`. Для корректной работы с code points есть `System.Text.Rune`.»

Model answer (EN): «`char` is 2 bytes — a single UTF-16 code unit, not a full Unicode character. Code points beyond the BMP take a surrogate pair — two chars. So `"😀".Length == 2`. Use `System.Text.Rune` to iterate real code points.»

---

**5. Value type всегда на стеке?** _(trap)_

Слабый/неверный ответ: «да, value types на стеке, reference в куче». Фейлит, потому что путает **тип** с **местом хранения**.

Model answer (RU): «Нет. Место зависит от слота: локал/параметр — обычно стек; как поле класса, элемент массива, захват замыканием или после boxing — куча. „Value type" определяет семантику копирования, не местоположение.»

Model answer (EN): «No. Location depends on the slot: locals/params usually on the stack; as a class field, array element, captured variable, or boxed — on the heap. „Value type" defines copy semantics, not storage location.»

---

**6. Что значит «built-in type»?** _(средний)_

Слабый/неверный ответ: «built-in = примитив». Фейлит, потому что `string`/`object` тоже built-in, но не примитивы и не value types.

Model answer (RU): «Built-in — это кейворд-алиасы C# на типы BCL (`int`→`Int32`, `string`→`String`). Шире, чем примитивы: `string`, `object` — built-in, но reference-типы и не CLR-примитивы.»

Model answer (EN): «Built-in types are C# keyword aliases for BCL types (`int`→`Int32`, `string`→`String`). Broader than primitives: `string`, `object` are built-in but reference types and not CLR primitives.»

---

**7. 🔬 `Type.IsPrimitive` для `decimal`? А для `nint`?** _(senior-добивка)_

Model answer (RU): «`decimal` → `false` (не CLR primitive). `nint` — алиас `IntPtr`, а `IntPtr` входит в 14 примитивов CLR, значит `typeof(nint).IsPrimitive == true`. Любопытный момент: `IntPtr`/`UIntPtr` — примитивы CLR, но не C# simple types.»

Model answer (EN): «`decimal` → `false` (not a CLR primitive). `nint` aliases `IntPtr`, which _is_ one of the 14 CLR primitives, so `typeof(nint).IsPrimitive == true`. Note the quirk: `IntPtr`/`UIntPtr` are CLR primitives but not C# simple types.»