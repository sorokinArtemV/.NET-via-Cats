Boxing — это мост между двумя семействами типов из первых двух глав: способ положить value type туда, где система типов ждёт `object` или интерфейс. Тема важна на собесе не сама по себе, а потому что boxing **невидим**: он не пишется явно, случается в безобидно выглядящем коде и тихо порождает аллокации в куче — а значит, нагрузку на GC. Половина вопросов про производительность value types сводится к «где здесь спрятался boxing». Поэтому ценность не в заучивании определения, а в умении показать пальцем на невидимые случаи и назвать настоящую цену.

Эта глава добивает forward-ref'ы из двух предыдущих: в Главе 1 мы сказали «переход value type к `object`/интерфейсу — это boxing», в Главе 2 — «boxed-значение живёт в куче, ≈24 байта». Здесь — механика и цена целиком.

### Mental model

**Boxing — это упаковка копии value type в новый объект на куче. Unboxing — извлечение копии значения обратно, с проверкой точного типа.** Ключевое слово — **копия**: boxing не «помечает» значение, он аллоцирует объект и копирует туда биты; unboxing копирует их назад.

### Как это работает

При boxing рантайм аллоцирует в куче объект с полноценным заголовком, копирует туда биты значения и возвращает ссылку. При unboxing — проверяет, что тип в `object` точно совпадает с целевым, и копирует биты обратно на стек. Это две независимые копии плюс одна аллокация:

```csharp
int x = 100;
object o = x;      // boxing: alloc в куче + копия 100
int y = (int)o;    // unboxing: проверка типа + копия назад
```

После этих трёх строк существуют **три** независимых места с числом 100: `x` на стеке, копия внутри boxed-объекта в куче, и `y` на стеке. Посмотрим на этот round-trip:

![[Pasted image 20260629115649.png]]
#### Когда происходит — и где он невидим

Явный случай один: присваивание value type в `object`. Опасны **невидимые**:

- присваивание в `object` или `dynamic` (в том числе неявно);
- использование struct через **интерфейс** (`IComparable x = myStruct;` — struct боксится, потому что интерфейс — reference type);
- добавление value type в **не-generic** коллекцию (`ArrayList`, `Hashtable`) или туда, где параметр — `object`;
- вызов у struct **неперекрытых** `ToString()`/`Equals()`/`GetHashCode()`, унаследованных от `object`/`ValueType`;
- `string.Format("{0}", x)` и интерполяция до C# 10 (аргументы боксятся в `object[]`).

И сразу разрушим частый миф из «шпаргалок»: **`Console.WriteLine(42)` НЕ боксит.** У `Console.WriteLine` есть перегрузка `WriteLine(int)`, и компилятор выбирает именно её. Boxing был бы, только если бы остался единственный overload с `object`. А вот `Console.WriteLine("{0}", 42)` — боксит, потому что там `params object[]`.

#### Цена

Не «оверхед на пометку», а конкретно: **аллокация объекта в куче** (≈24 байта на boxed `int` против 4 у голого — см. диаграмму), **копирование** битов туда и обратно, и **работа для GC** — лишний объект, который надо отследить и собрать. В горячем цикле это убивает производительность:

```csharp
for (int i = 0; i < 100_000; i++)
    object o = i;   // 100 000 аллокаций и столько же будущих сборок
```

#### Как избежать

- **Generics вместо `object`.** `void Print<T>(T value)` при `T = int` не боксит — generic над value type инстанцируется под конкретный тип, без перехода к `object`.
- **Generic-коллекции:** `List<int>` не боксит, `ArrayList` — боксит на каждом `Add`.
- **Перекрывай `Equals`/`GetHashCode`/`ToString`** у своих struct'ов (и реализуй `IEquatable<T>`) — убирает boxing при сравнении и в коллекциях.

#### Unboxing требует точного типа

Unboxing проверяет, что в `object` лежит **ровно** целевой тип — никаких неявных числовых конверсий:

csharp

```csharp
object o = 5;            // boxed int
double d = (double)o;    // InvalidCastException! не int → не распакуется в double
int i = (int)o;          // ОК, затем при желании double dd = i;
```

Это любимая ловушка: `(double)(object)5` падает в рантайме, хотя `int → double` обычно неявно разрешено. При boxing тип фиксируется, и unbox идёт только в него.

---

#### 🔬 Глубже / на синьора

**`Nullable<T>` боксится «без обёртки».** При boxing `int?` рантайм не создаёт boxed-`Nullable`: если `HasValue` — боксится **underlying** `T`, если нет — получается просто `null`.

```csharp
int? n = 5;  object o = n;  o.GetType();  // System.Int32, не Nullable<Int32>
int? m = null; object p = m;  // p == null  (не boxed-объект!)
```

**Вызов интерфейсного метода через generic-constraint НЕ боксит.** Это важная синьорская граница. Через переменную-интерфейс — боксит; через `where T : IFoo` — нет:

```csharp
void ViaInterface(IComparable x) { x.CompareTo(0); }      // аргумент-struct боксится
void ViaConstraint<T>(T x) where T : IComparable { x.CompareTo(0); } // НЕ боксит
```

Компилятор для constrained-вызова эмитит IL-префикс `constrained.`, и для struct'а, который реализует метод напрямую, диспетчеризация идёт без boxing. Именно поэтому `List<T>`, `Comparer<T>`, LINQ над value types не боксят — они построены на constraints, а не на `object`.

**Дефолтный `ValueType.Equals` боксит и медленный.** Унаследованная реализация для struct с reference-полями сравнивает поля через reflection, а сам вызов `Equals(object other)` боксит аргумент-struct. Двойная причина реализовать `IEquatable<T>`: и корректность, и ноль boxing/reflection в горячем пути. (Полностью — в главе про равенство.)

**Интерполяция: C# 10 убрал boxing.** Начиная с C# 10 / .NET 6 интерполированная строка опускается до `DefaultInterpolatedStringHandler` с дженерик-методом `AppendFormatted<T>(T)`, который **не боксит** value-аргументы (и специализируется под тип). На .NET Framework / netstandard2.0 по-прежнему `string.Format` с boxing в `object[]`. Так что «интерполяция медленнее конкатенации» — устаревший совет для .NET 6+.

---

### Interview layer

#### Вопросы по частоте

1. **«Что такое boxing/unboxing?»** — почти гарантированно. Ждут «копия value type в объект на куче и обратно», а не «преобразование типов».
2. **«Где boxing происходит незаметно?»** — главная проверка глубины: интерфейсы, `object`-параметры, не-generic коллекции, неперекрытый `ToString`/`Equals`.
3. **«В чём цена и как избежать?»** — аллокация + GC; generics и generic-коллекции.
4. **«Что будет: `double d = (double)(object)5;`?»** — `InvalidCastException` (нужен точный тип).
5. (синьор) **«Как боксится `int?`?»** / **«Боксит ли вызов через `where T : IComparable`?»**

#### Follow-up tree

> **Q:** Что такое boxing?  
> **→ likely:** Где он живёт и сколько стоит?  
> **→ trap:** «`Console.WriteLine(42)` боксит?» (Нет — есть overload `WriteLine(int)`.)

> **Q:** Как избежать boxing?  
> **→ likely:** Почему `List<int>` не боксит, а `ArrayList` боксит?  
> **→ trap:** «А вызов интерфейсного метода struct'а через `where T : IFoo` боксит?» (Нет — `constrained.`-вызов.)

> **Q:** `object o = 5; double d = (double)o;` — что будет?  
> **→ likely:** Почему, ведь `int → double` неявно можно?  
> **→ trap:** «А `int? n = 5; object o = n; o.GetType()`?» (`Int32`, не `Nullable<Int32>`.)

#### Частые поверхностные ответы и почему проваливаются

- **«Boxing — это просто преобразование типов».** Упускает суть: это **аллокация в куче + копия**, отсюда вся цена. Без слова «аллокация» ответ пустой.
- **«`Console.WriteLine(42)` боксит».** Распространённая ошибка из шпаргалок — есть перегрузка `WriteLine(int)`. Boxing там нет.
- **«Boxing медленный из-за приведения типов».** Нет — из-за аллокации и давления на GC. Приведение копеечное.
- **«`(double)(object)5` сработает, ведь int → double неявно».** Нет — unbox требует **точного** типа, будет `InvalidCastException`.
- **«Generics всё равно боксят value types».** Наоборот — generic над value type инстанцируется под конкретный тип, поэтому boxing и не возникает. В этом весь смысл `List<T>`.

#### Model answer (mid+)

**RU.** Boxing — упаковка копии value type в объект на куче, чтобы его можно было передать туда, где ждут `object` или интерфейс; unboxing — извлечение копии значения обратно с проверкой точного типа. Это не «пометка», а аллокация плюс копирование, поэтому и дорого: лишний объект в куче (boxed `int` ≈ 24 байта против 4) и нагрузка на GC. Случается часто незаметно — при присваивании в `object`, использовании struct через интерфейс, добавлении в не-generic коллекцию, вызове неперекрытого `ToString`/`Equals`. Избегают через generics и generic-коллекции: `List<int>` не боксит, `ArrayList` боксит на каждом `Add`. Unboxing требует ровно того же типа — `(double)(object)5` бросит `InvalidCastException`.

_Senior add-on._ Вызов интерфейсного метода через `where T : IFoo` не боксит (IL `constrained.`), в отличие от вызова через переменную-интерфейс. `int?` боксится в underlying (`null`, если пусто), а не в boxed-`Nullable`. C# 10/.NET 6 убрали boxing в интерполяции через `DefaultInterpolatedStringHandler`.

**EN.** Boxing wraps a copy of a value type in a heap object so it can go where an `object` or interface is expected; unboxing extracts a copy back, checking the exact type. It's not a "tag" — it's an allocation plus a copy, which is why it costs: an extra heap object (a boxed `int` is ~24 bytes vs 4) and GC pressure. It often happens invisibly — assigning to `object`, using a struct through an interface, adding to a non-generic collection, calling a non-overridden `ToString`/`Equals`. You avoid it with generics and generic collections: `List<int>` doesn't box, `ArrayList` boxes on every `Add`. Unboxing needs the exact type — `(double)(object)5` throws `InvalidCastException`.

_Senior add-on._ Calling an interface method via `where T : IFoo` doesn't box (IL `constrained.`), unlike calling through an interface-typed variable. An `int?` boxes to its underlying value (or `null` if empty), never to a boxed `Nullable`. C# 10/.NET 6 removed boxing in interpolation via `DefaultInterpolatedStringHandler`.

---

#### Что я проверил и осталось под `VERIFY:`

- **Проверено (web):** C# 10/.NET 6 интерполяция не боксит (`DefaultInterpolatedStringHandler.AppendFormatted<T>`); на .NET Framework/netstandard2.0 — боксит через `string.Format`. RECON-катч про `Console.WriteLine(42)` — подтверждён (есть `WriteLine(int)`).
- **Уверен без отдельного пина:** `constrained.`-вызов без boxing, поведение boxing `Nullable<T>`, `InvalidCastException` при unbox не в точный тип, ≈24 байта у boxed `int` на x64.
- **VERIFY (опционально):** ничего критичного не осталось открытым; если захочешь добавить бенчмарк-цифры (аллокации/время) для секции «цена» — отдельно прогоню BenchmarkDotNet-данные перед вставкой.