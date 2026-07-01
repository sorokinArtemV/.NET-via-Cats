### Вступление

`user.Address.City` бросает `NullReferenceException` в проде, хотя «всё же есть». Оказывается, `Address` не заполнен, и обращение к его полю падает. Ты добавляешь `if (user.Address != null)` — и тут же ловишь второй класс проблем: возраст пользователя приходит из БД как «может отсутствовать», но `int` не умеет быть «пустым», а `int age = null;` даже не компилируется. Приходится городить `-1` как «нет значения» и не забывать проверять его повсюду.

Оба случая — про одну вещь: **как язык выражает «значения нет».** У ссылочных типов «нет объекта» встроено — это `null`. У значимых типов такого состояния нет: `int` — это всегда какие-то 32 бита, «пустого int» не существует. Проблема, которую решает эта глава: понять, что `null` для ссылочных типов и «отсутствие» для значимых — это **два разных механизма**, где первый нативный, а второй эмулируется структурой `Nullable<T>`; разобрать её устройство, коварный boxing, поведение операторов над nullable и идиомы (`?.`, `??`, `??=`), которые убирают падения. Это фундамент, поверх которого потом ляжет compile-time история (NRT).

### Mental model

> **`null` — это валидное состояние _ссылки_ (слот не указывает ни на что). Значимый тип такого состояния не имеет, поэтому «отсутствие» для него моделируется структурой `Nullable<T>` = `{ bool HasValue; T value }`.** `int?` — это не ссылка и не «специальный int», это обычная значимая структура.

Первое, что нужно уложить — почему `null` бесплатен для ссылок и требует обёртки для значений:

![[Pasted image 20260701162153.png]]

### Как это работает

#### `null` у ссылочных и значимых типов

`null` означает, что ссылка не указывает ни на один объект. Это валидное значение для ссылочных типов (`string`, `object`, `class`, `List<T>` — их default тоже `null`) и для `Nullable<T>`. Значимые типы (`int`, `bool`, `DateTime`, `double`, любой `struct`) `null` не принимают в принципе: `int age = null;` — ошибка компиляции, потому что у них нет битового представления «пусто». Чтобы значимый тип мог быть «пустым», его оборачивают в `Nullable<T>`, для чего есть краткая форма `T?`:

```csharp
int? age = null;        // Nullable<int>
DateTime? date = null;
bool? flag = null;
```

#### Устройство `Nullable<T>`

`Nullable<T>` — это структура (значимый тип!) с ограничением `where T : struct`, внутри два поля: `bool HasValue` и `T value`. Никакой кучи, никакой ссылки: `int?` лежит там же, где лежал бы `int`, просто занимает на один `bool` больше.

```csharp
int? age = 25;
if (age.HasValue)
    Console.WriteLine(age.Value);   // 25
```

Важные следствия из того, что это обычная структура:

- Обращение к `.Value`, когда `HasValue == false`, бросает **`InvalidOperationException`** — не `NullReferenceException`. Это частая ловушка: люди ждут NRE, потому что «оно же null», но null тут не ссылка.
- Безопасно достать значение помогает `GetValueOrDefault()` (вернёт `default(T)` без исключения) или `GetValueOrDefault(fallback)`.
- `n.HasValue` и `n != null` — эквивалентны (компилятор поднимает сравнение). `is not null`, `HasValue`, `!= null` — про одно и то же.
- Вложить нельзя: `Nullable<Nullable<int>>` (то есть `int??`) запрещён — `T` не может сам быть nullable, это исключается ограничением `where T : struct`.

#### Boxing `Nullable<T>` — главный трюк CLR

Вот где чаще всего валят. Когда `Nullable<T>` боксится (передаётся туда, где ждут `object`), CLR **не боксит саму структуру `Nullable<T>`** — он делает специальную обработку:

![[Pasted image 20260701162209.png]]

Отсюда сразу несколько неочевидных фактов. `(int?)5` после boxing даёт boxed `int`, поэтому `boxed.GetType()` вернёт `System.Int32`, а не `System.Nullable<int>` — по инстансу вообще нельзя определить, что он был nullable (для этого смотрят на `Type` через `typeof(int?)`, а не на объект). `(int?)null` при boxing не создаёт объект вовсе — это настоящий `null`, поэтому `object o = (int?)null; o == null` → `true`, без аллокации в куче. Обратно: boxed `int` можно распаковать в `int?`; `null` распаковывается в `int?` как `HasValue == false`; а вот `(int)nullableNull` (распаковка отсутствия в невлозируемый `int`) бросит `InvalidOperationException`.

#### Lifted-операторы над nullable

Операторы `T` автоматически «поднимаются» на `T?` (lifted operators). Правило: если любой операнд `null`, арифметика/операция возвращает `null`:

csharp

```csharp
int? a = null, b = 5;
int? c = a + b;   // null
```

Но у **сравнений** правило другое и контринтуитивное — трёхзначная логика в духе SQL: если хоть один операнд `null`, `<`, `>`, `<=`, `>=` возвращают **`false`** (а не `null`):

csharp

```csharp
int? x = null;
bool r = x > 5;   // false, НЕ null
```

Равенство — исключение: `==`/`!=` подняты так, что `null == null` → `true`, `null == 5` → `false`. А у `bool?` — своя трёхзначная логика для `&` и `|`:

|`a`|`b`|`a & b`|`a \| b`|
|---|---|---|---|
|`true`|`null`|`null`|`true`|
|`false`|`null`|`false`|`null`|
|`null`|`null`|`null`|`null`|

То есть `true | null == true` и `false & null == false` — результат известен, даже если второй операнд неизвестен.

#### Операторы `?.`, `??`, `??=`

`?.` (null-conditional) — обращение к члену, только если левая часть не `null`, иначе вся цепочка становится `null`:

```csharp
int? len = name?.Length;          // null, если name == null
var city = user?.Address?.City;   // короткое замыкание всей цепочки
```

Две тонкости `?.`: он превращает результат в nullable (у `string.Length` тип `int`, а у `name?.Length` — уже `int?`); и он **вычисляет левый операнд ровно один раз** — поэтому `handler?.Invoke(...)` безопасен там, где `if (handler != null) handler(...)` имеет гонку (ссылка могла обнулиться между проверкой и вызовом). Этот приём — основа безопасного вызова событий (см. главу про Events).

`??` (null-coalescing) — вернуть правое, если левое `null`; композируется в цепочку:

```csharp
string name = input ?? cached ?? "Anonymous";
int realAge = age ?? 18;   // age : int?, realAge : int
```

`??=` (null-coalescing assignment, C# 8) — присвоить, только если переменная `null`:

```csharp
name ??= "Polina";   // эквивалент: if (name is null) name = "Polina";
```

Разница `??` vs `??=`: `??` возвращает значение, не трогая переменную; `??=` присваивает. Работает `??=` только по присваиваемой цели — `GetName() ??= "x"` не скомпилируется.

#### `default(T)` и null

`default(T)` даёт «нулевое» значение типа: для `string` (и любого ссылочного) — `null`; для `int` — `0`; для `bool` — `false`; для `Nullable<int>` — `null` (`HasValue == false`); для пользовательской структуры — экземпляр со всеми полями в default. С C# 7.1 можно писать просто `default` там, где тип выводится.

### 🔬 Глубже / на синьора

#### `is null` надёжнее `== null`

Оператор `==` можно перегрузить, и кривая перегрузка способна неправильно обработать `null` (или заходить в рекурсию). Паттерн `is null` / `is not null` **не вызывает перегруженный `==`** — он всегда делает чистую проверку на null (для ссылок — reference-null, для `Nullable<T>` — `!HasValue`), которую нельзя переопределить. Поэтому для типов с перегруженным равенством (см. главы про `Equals`/`==`) в качестве null-проверки предпочтителен `is null`:

```csharp
if (user is null) ...        // не зависит от operator ==
if (user is not null) ...
```

Компилятор о проблеме с `== null` не предупредит, а баг из-за неверной перегрузки доезжает до прода — это прямо отмечается как классический риск.

#### `Nullable<T>` и производительность

Плюс модели «структура, а не ссылка»: `int?` не аллоцируется в куче, не грузит GC, лежит inline. Минус проявляется при boxing: если гонять nullable через `object`/не-generic API в горячем цикле, каждый непустой `int?` боксится (пустой — нет, он становится null без аллокации). Для generic-кода это одна из причин, почему `EqualityComparer<T>.Default` имеет спец-ветку `NullableEqualityComparer` — чтобы сравнивать nullable без boxing (см. главу про компараторы).

#### Три вида «пустоты» — не путать

На синьорском уровне важно держать раздельно три вещи, которые новички смешивают: (1) `null` ссылочного типа — отсутствие объекта в рантайме; (2) `Nullable<T>` — значимая обёртка «значение или его отсутствие», тоже рантайм; (3) nullable reference types (`string?`) — это **аннотация компилятора без рантайм-следа**: `string` и `string?` в рантайме один и тот же `System.String`, `?` в CLR не существует. Первые две — эта глава; третья — следующая. Смешение (1)/(2)/(3) — самый частый провал на follow-up.

### Interview layer

#### Вопросы, по частоте/сеньорности

**1. Может ли `int` быть null? Как сделать nullable? (частый опенер)**

- → _likely follow-up:_ чем `int?` отличается от `int`?
- → _trap:_ `int?` — это класс или структура? (Структура `Nullable<int>`, значимый тип, в кучу не идёт.)

**2. Что происходит при boxing `int?` со значением и без? (senior trap)**

- → _likely follow-up:_ что вернёт `((int?)5).GetType()`? (`Int32`, не `Nullable`.)
- → _trap:_ `object o = (int?)null; o == null` — что и почему? (`true`, boxing отсутствия даёт настоящий null без аллокации.)

**3. `.Value` при `HasValue == false` — что бросит? (часто)**

- → _trap:_ это `NullReferenceException`? (Нет — `InvalidOperationException`; безопаснее `GetValueOrDefault` или паттерн `is int v`.)

**4. `int? x = null; x > 5` — чему равно? (mid+ trap)**

- → _likely follow-up:_ а `null == null`? (`true`.) А `x + 5`? (`null`.)
- Ответ: `false` — сравнения над nullable по трёхзначной логике возвращают `false`, если операнд null.

**5. Чем `?.` отличается от `if (obj != null)`? (часто)**

- → _trap:_ почему `handler?.Invoke()` безопаснее? (`?.` вычисляет операнд один раз — нет гонки между проверкой и вызовом.)

**6. `??` vs `??=`? Где `??=` не работает? (mid)** (`??` возвращает, не меняя; `??=` присваивает; не работает по не-lvalue: `GetX() ??= …` — ошибка.)

**7. Почему `is null` лучше `== null`? (senior)** (Обходит перегруженный `operator ==`.)

#### Типичные поверхностные ответы и почему они проваливаются

- **«`int?` — это ссылочная обёртка / класс».** Нет: `Nullable<T>` — структура, значимый тип, лежит inline, не аллоцируется в куче. `null` у неё — это `HasValue == false`, а не ссылка.
- **«`.Value` кидает `NullReferenceException`».** Нет: `InvalidOperationException`. NRE тут неоткуда взяться — это не разыменование ссылки.
- **«`int? x = null; x > 5` вернёт null».** Нет: `false`. Возвращает null только _арифметика_; сравнения дают `false`.
- **«boxing `int?` даёт boxed `Nullable<int>`».** Нет: CLR боксит содержимое — либо boxed `T`, либо настоящий null. По объекту nullable не отличить от `T`.
- **«`?.` — просто короткая запись `if (x != null)`».** Не совсем: `?.` возвращает nullable, короткозамыкает цепочку и вычисляет операнд один раз.

#### Model answers

**Q: Может ли `int` быть null и что такое `int?`?**

**RU:** Сам `int` — значимый тип, у него нет представления для «нет значения», поэтому `int x = null` не компилируется. Чтобы значимый тип мог быть «пустым», его оборачивают в `Nullable<T>`, краткая форма — `int?`. Это по-прежнему **структура** (значимый тип) с полями `HasValue` и `value`, а не ссылка: она не идёт в кучу. `null` для неё означает `HasValue == false`.

**EN:** `int` itself is a value type with no representation for "no value", so `int x = null` doesn't compile. To let a value type be "empty" you wrap it in `Nullable<T>`, shorthand `int?`. It's still a **struct** (value type) with `HasValue` and `value` fields, not a reference — it doesn't go on the heap. `null` for it means `HasValue == false`.  
_(senior add-on:_ `Nullable<T>` can't nest, and `?` here is a real runtime type, unlike `string?` which is a compile-time annotation.)*

**Q: Что происходит при boxing `int?`?**

**RU:** CLR не боксит саму структуру `Nullable<int>`. Если `HasValue == true`, боксится подлежащее значение — boxed `int` (`System.Int32`), поэтому `GetType()` вернёт `Int32`, а `is int` истинно. Если `HasValue == false`, boxing даёт настоящий `null` без аллокации в куче, и `o == null` → `true`. То есть по boxed-объекту нельзя понять, что он был nullable.

**EN:** The CLR doesn't box the `Nullable<int>` struct itself. If `HasValue == true`, it boxes the underlying value — a boxed `int` (`System.Int32`), so `GetType()` returns `Int32` and `is int` is true. If `HasValue == false`, boxing yields a real `null` with no heap allocation, and `o == null` is `true`. So you can't tell from the boxed object that it was nullable.

**Q: `.Value` при отсутствии значения — что бросает и как правильно?**

**RU:** `InvalidOperationException`, а не `NullReferenceException` — потому что это не разыменование ссылки, а проверка `HasValue` внутри структуры. Безопасно доставать значение через `GetValueOrDefault()`, оператор `??` (`age ?? 18`) или паттерн `if (n is int v)`, который сразу и проверяет наличие, и извлекает значение без boxing.

**EN:** `InvalidOperationException`, not `NullReferenceException` — because it's not a reference dereference but a `HasValue` check inside the struct. Get the value safely via `GetValueOrDefault()`, the `??` operator (`age ?? 18`), or the pattern `if (n is int v)`, which both checks presence and extracts the value without boxing.