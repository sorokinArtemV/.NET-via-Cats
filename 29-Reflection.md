Компилятор стирает большую часть того, что ты написал: имена локальных переменных, структуру выражений, удобные абстракции — всё это превращается в IL и метаданные. Но кое-что он **сохраняет в самой сборке** в машиночитаемом виде: описание всех типов, их методов, полей, свойств, конструкторов и навешанных атрибутов. Reflection — это API, который читает эти сохранённые метаданные во время выполнения и позволяет работать с типами, которых ты не знал на этапе компиляции: создавать объекты, вызывать методы, читать и писать поля, перечислять атрибуты, подгружать чужие сборки.

Зачем это нужно, если обычно тип известен заранее? Потому что целый класс задач принципиально динамический: DI-контейнер собирает граф объектов, не зная конкретных типов; сериализатор обходит произвольный объект; ORM сопоставляет свойства с колонками; тест-раннер находит `[Fact]`-методы; система плагинов грузит DLL, которой не было при сборке. Везде, где код должен работать с типами «вообще», а не с конкретным `Person`, под капотом сидит Reflection (или то, что её исторически заменяет — source generators, об этом в 🔬).

### Mental model

**Reflection не «смотрит на твой код» — она читает метаданные, которые компилятор уже запёк в сборку.** `Type`, `MethodInfo`, `FieldInfo` — это не сам код, а read-view (дескрипторы) над таблицами метаданных. Отсюда два следствия, которые тянутся через всю главу: (1) если метаданные удалить или не сгенерировать — Reflection слепнет (trimming/AOT); (2) каждый «вопрос к метаданным» — это lookup в рантайме, поэтому он стоит денег.

![[Pasted image 20260630175240.png]]

### Как это работает (mid+ core)

#### Получение `Type` — три двери, не одна

`Type` — точка входа во всё остальное. Получить его можно тремя способами, и разница между ними — частый вопрос на собеседовании:

```csharp
Type t1 = typeof(Person);              // compile-time, по статическому типу
Type t2 = obj.GetType();               // runtime, по фактическому типу объекта
Type t3 = Type.GetType("MyApp.Person, MyApp"); // runtime, по строке
```

- `typeof(T)` резолвится **на этапе компиляции** в metadata token, объект не нужен, промахнуться нельзя. Самый дешёвый.
- `obj.GetType()` возвращает **фактический** тип во время выполнения. Если `Person p = new Student()`, то `typeof(Person)` даст `Person`, а `p.GetType()` — `Student`. Это ключевое различие. (Нюанс: вызов `GetType()` на значимом типе боксит его — `GetType` живёт на `object`.)
- `Type.GetType(string)` ищет по имени в рантайме и возвращает **`null`**, если не нашёл (либо бросает с `throwOnError: true`). Главная ловушка: для типа за пределами текущей сборки и `System.Private.CoreLib` нужно **assembly-qualified name** (`"Ns.Type, AssemblyName"`), иначе `null`.

#### Иерархия `MemberInfo`

Всё описание членов наследуется от `MemberInfo`. Стоит держать карту в голове, потому что методы возврата (`GetMembers`, `GetMethods`…) типизированы:

```
MemberInfo
├── Type            (через TypeInfo)
├── MethodBase
│   ├── MethodInfo
│   └── ConstructorInfo
├── PropertyInfo
├── FieldInfo
└── EventInfo
```

#### `BindingFlags` — почему `GetMethod("X")` ничего не находит

Это источник 90% «почему рефлексия не видит мой метод». Перегрузки без флагов возвращают **только public**, а `GetMethods()` по умолчанию отдаёт public instance + static, **включая унаследованные**. Чтобы достать что-то приватное или ограничить выборку, флаги обязательны, и `Instance`/`Static` нужно комбинировать с `Public`/`NonPublic`:

```csharp
// Найдёт ТОЛЬКО если SayHello — public:
type.GetMethod("SayHello");

// Приватный instance-метод:
type.GetMethod("_compute", BindingFlags.NonPublic | BindingFlags.Instance);
```

Полезные флаги: `DeclaredOnly` (исключить унаследованное), `FlattenHierarchy` (включить унаследованные static), `IgnoreCase`.

#### Создание объектов

```csharp
object o1 = Activator.CreateInstance(type);          // вызовет parameterless ctor
object o2 = Activator.CreateInstance(type, arg1, arg2); // подберёт ctor по аргументам
var o3 = Activator.CreateInstance<Person>();          // generic-вариант
```

Если parameterless-конструктора нет, а ты зовёшь версию без аргументов — `MissingMethodException`. Для точного контроля берут `ConstructorInfo` через `type.GetConstructor(...)` и зовут `Invoke`.

#### Вызов методов и доступ к полям/свойствам

```csharp
MethodInfo m = type.GetMethod("SayHello");
object result = m.Invoke(target, new object[] { "arg" }); // static → target = null

FieldInfo f = type.GetField("_secret", BindingFlags.NonPublic | BindingFlags.Instance);
f.SetValue(target, "new");
object v = f.GetValue(target);

PropertyInfo p = type.GetProperty("Name");
p.SetValue(target, "Anna");
```

**Ловушка с исключениями:** если метод, вызванный через `Invoke`, бросает исключение, Reflection оборачивает его в `TargetInvocationException`, а настоящее исключение лежит в `InnerException`. Это ломает наивные `catch (SpecificException)` вокруг `Invoke`.

#### Доступ к приватным членам

Да, через `BindingFlags.NonPublic` можно прочитать и записать приватное поле или вызвать приватный метод. В тестах это иногда оправдано, в продакшене — почти всегда симптом проблемы с дизайном (нарушение инкапсуляции, хрупкая связь с чужими внутренностями). Отдельная ловушка: запись в `readonly`/`init`-поле через `SetValue` исторически «работала» для классов, но официально не поддерживается, ненадёжна для структур и может падать под AOT — на это нельзя закладываться.

#### Reflection над generics — open vs closed

```csharp
Type open   = typeof(List<>);       // open generic (unbound) — IsGenericTypeDefinition == true
Type closed = typeof(List<int>);    // closed (constructed)
Type made   = open.MakeGenericType(typeof(int)); // == typeof(List<int>) в рантайме
```

`MakeGenericType` / `MakeGenericMethod` — то, как DI-контейнеры и мапперы строят `List<TUserType>`, не зная `T` заранее. Запомни эту пару: именно она потом окажется враждебна Native AOT (см. 🔬).

#### Загрузка сборок и плагины

```csharp
Assembly asm = Assembly.LoadFrom("Plugin.dll");
foreach (Type t in asm.GetTypes())
    if (typeof(IPlugin).IsAssignableFrom(t) && !t.IsAbstract)
        plugins.Add((IPlugin)Activator.CreateInstance(t));
```

В .NET Core/5+ механизм загрузки — `AssemblyLoadContext` (ALC), пришедший на замену `AppDomain` из .NET Framework. Главная ловушка плагинов здесь: **идентичность типа — это пара (тип + ALC)**. Один и тот же `IPlugin`, загруженный в двух разных контекстах, — это два разных типа, и каст между ними бросит `InvalidCastException`. Решается тем, что общий контракт (`IPlugin`) грузится в один shared-контекст.

#### Стоимость — и как её убрать

Главное, что нужно проговорить на собеседовании 2026 года: **«Reflection медленная» — миф наполовину.** Дорогой именно _per-call lookup_ — поиск члена по имени, сверка параметров, проверка доступа, упаковка значимых аргументов в `object[]`, indirect call без инлайнинга. Сам разовый «вопрос к метаданным» — это нормально. Проблема возникает, когда ты делаешь `GetMethod(...)` + `Invoke(...)` в горячем цикле.

Порядок величин (зависит от среды, это иллюстрация, не бенчмарк): «классический» `ConstructorInfo.Invoke` — примерно на порядок медленнее `new`, `Activator.CreateInstance` — около ×5, а скомпилированный делегат — практически как прямой вызов (разница в единицы наносекунд).

Лечение: вынеси lookup из горячего пути и закэшируй результат, а сам вызов сделай делегатом.

![[Pasted image 20260630175318.png]]

Практический паттерн «тёплого пути»:

```csharp
// ❌ В горячем цикле: lookup на каждой итерации
foreach (var x in items)
    type.GetMethod("Process").Invoke(x, null);

// ✅ Lookup один раз, вызов — делегатом
MethodInfo mi = type.GetMethod("Process")!;
var call = (Func<Target, int>)mi.CreateDelegate(typeof(Func<Target, int>));
foreach (var x in items)
    call(x); // ≈ прямой вызов, без boxing и lookup
```

Если делегат строго типизировать нельзя, второй по скорости вариант — `Expression.Compile` (см. главу **Expression Trees**, где этот путь разбирается целиком). А с .NET 8 у `MethodInfo.Invoke` появился ещё и встроенный быстрый путь плюс кэшируемые `MethodInvoker`/`ConstructorInvoker` — об этом ниже.

### 🔬 Глубже / на синьора

**Что на самом деле такое `Type`.** `obj.GetType()` и `typeof(T)` возвращают `RuntimeType` — закэшированный рантаймом синглтон на каждый тип в рамках одного ALC. Сам `RuntimeType` — это «ручка» к `MethodTable` объекта и через неё к таблицам метаданных. У каждого члена есть `MetadataToken` — по сути адрес записи в metadata-таблице. Важный нюанс производительности: рантайм кэширует `RuntimeType`, но **`GetMethod("X")` всё равно делает поиск при каждом вызове** — поэтому кэшировать `MemberInfo` нужно тебе, рантайм за тебя этого не сделает.

**Fast-invoke внутренности (.NET 8+).** Раньше `MethodInfo.Invoke` шёл через медленный обобщённый путь. В .NET 7/8 рантайм при первом вызове строит специализированный invoker-стаб (через IL-эмиссию там, где она доступна), поэтому первый `Invoke` медленнее последующих. В .NET 8 добавили публичные `MethodInvoker.Create(mi)` и `ConstructorInvoker.Create(ci)` — их кэшируешь и зовёшь перегрузками без `object[]` (до 4 аргументов), что убирает аллокацию массива и часть боксинга и даёт примерно ×2 по CPU относительно классического `Invoke`. Это «средний» путь: быстрее старого `Invoke`, но всё ещё медленнее строго типизированного делегата.

**Trimming и Native AOT — самый горячий senior-топик 2026 (.NET 10).** Trimmer и AOT-компилятор статически вычисляют, какой код достижим, и выкидывают остальное. Reflection по строке (`Type.GetType(userString)`, перебор `asm.GetTypes()`) для них **непрозрачна** — они не видят, какие типы тебе понадобятся:

- Код, несовместимый с trimming, помечается `[RequiresUnreferencedCode]` → предупреждение **IL2026** у вызывающих. Member, до которого «не дотянулась» статическая видимость, может быть вырезан — и Reflection упадёт в рантайме.
- `MakeGenericType`, `Reflection.Emit` и другая рантайм-кодогенерация несовместимы с AOT → `[RequiresDynamicCode]` (**IL3050**). Тонкость: `MakeGenericType` только с reference-типами обычно работает (shared generics переиспользуют один машинный код), а с value-типами требует генерации нового кода, которого под AOT нет.
- Инструмент, чтобы сохранить нужные члены: `[DynamicallyAccessedMembers(...)]` (доступен с .NET 5) — пробрасываешь по цепочке вызовов до места, где тип известен статически.
- Под Native AOT вдобавок: нет `Reflection.Emit`, нет `Assembly.LoadFile`, а `System.Linq.Expressions` работают **в интерпретируемом режиме** (медленнее скомпилированных) — то есть привычный трюк «`Expression.Compile` вместо рефлексии» под AOT теряет смысл, и кэшированный `CreateDelegate` становится предпочтительным.
- Стратегическое направление платформы: **source generators вытесняют рантайм-рефлексию**, перенося её на этап компиляции — `System.Text.Json` (`JsonSerializerContext`), биндинг конфигурации, `[GeneratedRegex]`, `LoggerMessage`. Правильный ответ на «как сделать рефлексию AOT-safe» сегодня часто звучит как «не делать её в рантайме — заменить source generator’ом».

**`AssemblyLoadContext` и выгрузка.** ALC заменил `AppDomain`; для плагинов с возможностью выгрузки создают **collectible** ALC. Тип живёт ровно столько, сколько жив его ALC; пара (тип, ALC) определяет идентичность, отсюда падающие касты между контекстами.

**Pseudo-custom attributes.** Часть атрибутов (`[Serializable]`, `[StructLayout]`, `[DllImport]`) компилятор кладёт не как обычный custom-attribute blob, а как **флаги прямо в метаданные**, и `GetCustomAttributes` их не вернёт — их читают через специальные свойства (`type.IsSerializable`, `type.StructLayoutAttribute`). Разбор — в главе **Attributes**.

### Interview layer

#### Вопросы по частоте

1. **Что такое Reflection и где её реально применяют?** (очень часто, mid)
2. **Почему Reflection медленная и как ускорить?** (очень часто, mid+)
3. **`typeof` vs `GetType()` vs `Type.GetType()` — в чём разница?** (часто, mid)
4. **Как достать/вызвать приватный член и стоит ли?** (часто, mid)
5. **Open vs closed generic, зачем `MakeGenericType`?** (mid+)
6. **Как Reflection ломается под trimming/Native AOT и как чинить?** (senior, 2026-актуально)
7. **`AssemblyLoadContext`: плагины, выгрузка, почему каст между контекстами падает?** (senior)

#### Деревья follow-up

**«Почему медленно?»**  
→ _likely:_ что именно дорого? → _(lookup по имени, проверка доступа, боксинг аргументов в `object[]`, indirect call без инлайнинга)_  
→ _trap:_ а что изменилось в .NET 8? чем `CreateDelegate` отличается от `Expression.Compile`? → _(.NET 8 — fast-invoke + `MethodInvoker`; делегат ≈ прямой вызов и не требует кодогенерации, поэтому работает и под AOT, в отличие от `Expression.Compile`, который под Native AOT интерпретируется)_

**«typeof vs GetType?»**  
→ _likely:_ что вернёт `GetType()` для `Person p = new Student()`? → _(`Student` — фактический тип)_  
→ _trap:_ а `typeof` для того же — и почему? → _(`Person`, потому что `typeof` берёт статический тип на этапе компиляции)_

**«Сделай плагины через LoadFrom»**  
→ _likely:_ как найти реализации `IPlugin` в чужой сборке? → _(`GetTypes()` + `IsAssignableFrom` + `!IsAbstract`)_  
→ _trap:_ почему каст загруженного типа к `IPlugin` бросает `InvalidCastException`? → _(контракт загрузился в другой ALC; идентичность типа = тип + контекст)_

#### Частые поверхностные ответы и почему они проваливаются

- **«Reflection медленная, потому что рантайм».** Не объясняет _что именно_ дорого и игнорирует, что разовый lookup нормален, а кэш + делегат сводят разницу почти к нулю; не знает про .NET 8 fast-invoke.
- **«`typeof` и `GetType()` — одно и то же».** Путает compile-time на статическом типе и runtime на фактическом; разваливается на примере с наследованием.
- **«Приватное через рефлексию не достать».** Достаётся через `BindingFlags.NonPublic | Instance`; правильный акцент — _можно, но это запах дизайна_.
- **«Под AOT просто добавь атрибут и заработает».** Не различает trimming (`RequiresUnreferencedCode`/`DynamicallyAccessedMembers`) и кодогенерацию (`RequiresDynamicCode`), не знает про source generators как стратегический ответ.

#### Модельные ответы

**«Почему Reflection медленная и как ускорить?»**

> _RU:_ Дорогой не сам факт рефлексии, а per-call работа: поиск члена по имени, проверка доступа, упаковка аргументов в `object[]` с боксингом значимых типов и непрямой вызов без инлайнинга. Лечится выносом lookup из горячего пути: кэшируешь `MemberInfo` один раз, а вызов делаешь строго типизированным делегатом через `CreateDelegate` — тогда он близок к прямому. В .NET 8 у `Invoke` появился быстрый путь и кэшируемые `MethodInvoker`/`ConstructorInvoker` без аллокации `object[]`. Для совсем горячих сценариев альтернатива — скомпилированное `Expression`, но под Native AOT оно интерпретируется, поэтому там предпочтителен делегат.

> _EN:_ Reflection itself isn’t the cost — the per-call work is: resolving the member by name, access checks, packing arguments into an `object[]` with boxing for value types, and an indirect, non-inlinable call. The fix is to move the lookup out of the hot path: cache the `MemberInfo` once and invoke through a strongly-typed delegate via `CreateDelegate`, which runs close to a direct call. .NET 8 added a fast-invoke path plus cacheable `MethodInvoker`/`ConstructorInvoker` that avoid the `object[]` allocation. For very hot paths a compiled `Expression` is an option, but under Native AOT it falls back to interpretation, so a cached delegate is preferable there.

**«typeof vs GetType vs Type.GetType?»**

> _RU:_ `typeof(T)` резолвится на этапе компиляции по статическому типу — объект не нужен. `obj.GetType()` возвращает фактический тип в рантайме, поэтому для `Person p = new Student()` даст `Student`, а `typeof(Person)` — `Person`. `Type.GetType("Ns.Type, Asm")` ищет по строке в рантайме и возвращает `null`, если не нашёл; для типа из другой сборки нужно assembly-qualified имя.

> _EN:_ `typeof(T)` is resolved at compile time from the static type — no instance needed. `obj.GetType()` returns the actual runtime type, so for `Person p = new Student()` it yields `Student`, while `typeof(Person)` yields `Person`. `Type.GetType("Ns.Type, Asm")` resolves a string at runtime and returns `null` on miss; for a type in another assembly you need the assembly-qualified name.