### Подводка: зачем эта троица вместе

Это анатомия состояния объекта, и спрашивают их в связке, потому что между ними проходит главная граница ООП — граница между _хранилищем_ и _контрактом_:

- **field** — _где_ данные физически лежат;
- **property** — _как_ к этим данным пускают снаружи (контролируемая дверь поверх хранилища);
- **constructor** — _как_ объект рождается уже в валидном состоянии, до того как им начнут пользоваться.

Поле — деталь реализации, его прячут. Свойство и конструктор — часть публичного контракта типа, который нельзя менять безболезненно. Почти все вопросы на собесе растут отсюда: «почему не public field», «что выполнится раньше», «как сделать immutable».

---

### 1. Field

**Mental model:** Поле — именованная ячейка памяти внутри объекта. Настоящее хранилище, а не обёртка: в методы не компилируется.

#### Как работает (mid+)

Field — переменная-член класса или struct. В отличие от property, это реальный слот в памяти, а не пара методов.

**Где живёт:**

- у reference type (class) поля лежат внутри объекта на **heap**;
- у value type (struct) поля лежат **инлайн** — там же, где сама структура: на стеке (если это локальная переменная) или внутри объекта-владельца на heap.

**Default values:** неинициализированное поле получает `0` / `false` / `null`. Это отличает поля от локальных переменных — те компилятор требует инициализировать явно (definite assignment), поля — нет.

**Модификаторы, которые надо держать в голове:**

- по умолчанию поле `private` — поля почти всегда прячут, наружу дают property;
- `static` — поле принадлежит типу, а не экземпляру, одно на всех;
- `readonly` — присвоить можно только в объявлении или в конструкторе, дальше заморожено. Важно: для reference type замораживается **ссылка**, а не содержимое объекта (`readonly List<T>` всё ещё можно `.Add()`).

#### 🔬 Глубже / на синьора

- **`const` vs `static readonly`** — `const` вшивается в вызывающий код на этапе компиляции (inline литерала), `static readonly` читается в рантайме. Отсюда versioning-ловушка при смене значения в чужой сборке. Полный разбор — в главе про `const`/`readonly`, здесь только указатель.
- **Порядок инициализации полей** — instance field initializers выполняются в текстовом порядке и (контринтуитивно) **до тела конструктора**, даже до тела конструктора базового класса. Диаграмма — в секции Constructor.
- **`readonly struct` / defensive copies** — pointer на главу про `readonly`.

---

### 2. Property

**Mental model:** Поле = хранилище. Property = контролируемая точка входа к нему. Снаружи выглядит как поле, внутри — пара методов `get`/`set`.

#### Как работает (mid+)

Property компилируется в методы-аксессоры: `get` → `get_X()`, `set` → `set_X(value)`. Обращение `obj.X` — это вызов метода (который JIT в релизе обычно инлайнит, так что в горячем коде цена нулевая).

**Auto-property** — `public int X { get; set; }`. Компилятор сам генерит скрытый backing field и тривиальные get/set. Удобно, пока не нужна логика.

**Full property** — backing field и логика руками:

csharp

```csharp
private int _age;
public int Age
{
    get => _age;
    set => _age = value < 0
        ? throw new ArgumentOutOfRangeException(nameof(value))
        : value;
}
```

Что важно для мид+:

- **Асимметрия доступа:** `public int X { get; private set; }` — читать снаружи, писать только изнутри класса.
- **`init`** (C# 9): `{ get; init; }` — присвоить можно только в конструкторе или object initializer; после создания объект заморожен. Рабочий способ делать immutable без громоздких конструкторов.
- **Expression-bodied / computed:** `public string FullName => $"{First} {Last}";` — только для чтения, считается на лету, backing field нет.
- **Property можно объявить в interface; instance field — нельзя** (static-поля в интерфейсе допустимы с C# 8). Контракт типа выражается свойствами.

##### Почему property, а не public field (вопрос №1 на собесе)

Это не «инкапсуляция ради инкапсуляции». Конкретные механизменные причины:

1. **Binary compatibility.** Заменить `public field` на `property` позже — это **binary breaking change**: меняется способ доступа (memory access → method call), и все зависимые сборки надо перекомпилировать, иначе рантайм-падение. Если сразу property — будущую логику добавляешь без слома ABI.
2. **Можно добавить логику потом** — валидацию, lazy, логирование, raise `PropertyChanged` — не трогая сигнатуру.
3. **Ожидания экосистемы.** Data-binding (WPF/MAUI), сериализаторы, ORM, validation-атрибуты часто работают по свойствам; поле они не увидят.
4. **`init`/`private set`** дают контроль над мутацией, которого у поля нет.

#### 🔬 Глубже / на синьора

- **Backing field в IL** называется `<X>k__BackingField` — unspeakable name, из кода недоступен. Рефлексия видит `PropertyInfo` отдельно от `FieldInfo`.
- **Стоимость доступа:** в Debug аксессор — реальный call; в Release JIT инлайнит тривиальные get/set, и они эквивалентны прямому доступу к полю. «Property медленнее field» — миф для горячего пути с тривиальными аксессорами.
- **`required`** (C# 11): `public required int X { get; init; }` — компилятор заставляет инициализировать при создании, иначе ошибка компиляции. Закрывает дыру «забыл задать обязательное свойство».
- **Нельзя взять `ref` на обычное property** — это методы, а не ячейка памяти (в отличие от поля). Исключение — `ref`-возвращающее свойство; pointer на главу `ref`/`out`/`in`. Это же делает замену field→property не всегда даже source-compatible: если кто-то передавал поле по `ref`, после превращения в property код перестанет компилироваться.
- **`field` keyword** (C# 14 / .NET 10, ноябрь 2025 — вышел из preview): доступ к авто-сгенерированному backing field прямо в аксессоре, без ручного объявления поля:

csharp

```csharp
  public int Age
  {
      get;
      set => field = value >= 0 ? value : throw new ArgumentOutOfRangeException(nameof(value));
  }
```

Контекстный — keyword только внутри тела аксессора. Закрывает 15-летнюю боль «нужна валидация → теряешь auto-property». В C# 13 был preview-only.

---

### 3. Constructor

**Mental model:** Конструктор — ритуал рождения объекта: код, который обязан оставить объект в валидном состоянии до первого использования.

#### Как работает (mid+)

Специальный метод: вызывается при `new`, имя = имя класса, **нет возвращаемого типа** (даже `void`). Перегружается по сигнатуре параметров — конструкторы подчиняются обычному overload resolution.

**Default (parameterless) constructor — правила разные для class и struct:**

- **class:** если не написал ни одного конструктора, компилятор генерит публичный безпараметрический. Как только написал любой свой — авто-генерация пропадает.
- **struct:** исторически всегда есть неявный занулящий ctor, и убрать его нельзя. С **C# 10** можно объявить _свой_ parameterless ctor с логикой. Но при `default(T)` и `new T[]` он **не вызывается** — там всегда чистое зануление.

**Constructor chaining:**

- `: this(...)` — делегировать другому конструктору этого же класса (убрать дублирование инициализации);
- `: base(...)` — вызвать конструктор базы. Если не написал — неявно подставляется `: base()`; если у базы нет доступного безпараметрического ctor — ошибка компиляции.

**Field initializers** (`private int _x = 5;`) — тоже часть инициализации, и выполняются они не там, где кажется интуитивно.

#### Порядок выполнения (mid+ trap)

Для `new Derived()` порядок такой:

![[Pasted image 20260622121820.png]]

Итого: _Derived-поля → Base-поля → тело Base → тело Derived_. Поля инициализируются до тела base-конструктора, и поля derived — раньше полей base.

**Практическое следствие (senior-trap):** если конструктор базового класса вызывает `virtual`-метод, переопределённый в Derived, диспетчеризация уйдёт в derived-override — но он выполнится, когда тело Derived ctor (шаг 4) ещё не отработало. Значит поле Derived, заданное **инициализатором**, уже на месте, а заданное **в теле конструктора** — ещё `null`/`default`. Отсюда правило «не вызывай virtual из конструктора».

#### 🔬 Глубже / на синьора

- **Field initializers и `: this(...)`:** если конструктор чейнится через `: this(...)`, его field initializers **не выполняются** — они отрабатывают ровно один раз, в том конструкторе цепочки, который вызывает `base`. Иначе поля занулялись бы дважды.
- **Primary constructors** (C# 12): `class Person(string name) { ... }` — параметры видны во всём теле класса (в field initializers, свойствах, методах). Для non-record class это **не** генерит автоматически свойства (в отличие от record): параметр живёт как значение в скоупе класса, компилятор сам решает, хранить ли его в скрытом поле. Удобно для DI: `class Service(IRepo repo)` — `repo` доступен в методах.
- **Static constructor:** вызывается рантаймом один раз перед первым обращением к типу; влияет на `beforefieldinit` и тайминг инициализации static-полей — pointer на главу `const`/`readonly`.
- **Исключение в конструкторе:** ссылку на объект ты не получаешь, он считается несозданным. Но если ctor успел опубликовать `this` наружу или захватить unmanaged-ресурс до `throw` — будет утечка/полусоздание; finalizer может сработать на недоинициализированном объекте.
- **records:** генерят конструктор, деконструктор и init-свойства из позиционного списка — pointer на главу про records & equality.

> **Method overloading** (в т.ч. перегрузка конструкторов) — отдельная механика overload resolution. Здесь достаточно: конструкторы и методы перегружаются по сигнатуре параметров (тип / количество / порядок), но **не по возвращаемому типу**. Полный разбор правил выбора перегрузки — отдельная глава.

---

### Interview layer

#### Q1. В чём разница между field и property? Почему не сделать public field? `(частота: очень высокая)`

**Follow-up tree:** разница field/property → что будет, если в публичной библиотеке заменить public field на property → _(trap)_ это binary breaking или source breaking?

**Shallow / wrong:** «property — это инкапсуляция, так правильно по ООП». Не объясняет механизм, разваливается на первом follow-up.

**Trap-ответ:** замена field→property обычно **source-compatible** (синтаксис обращения тот же), но **binary breaking** — старые скомпилированные потребители падают в рантайме, их надо перекомпилировать. Не всегда даже source-compatible: если поле передавали по `ref`, код перестанет компилироваться.

**Модельный ответ (RU):** Field — это слот в памяти, property — пара методов get/set поверх него. Public field я не делаю по двум причинам: (1) превратить его в property позже — binary breaking change, надо перекомпилировать всех потребителей, потому что доступ меняется с memory read на вызов метода; (2) на property можно навесить валидацию/lazy/нотификацию и его понимают binding, сериализаторы, ORM — а поле нет. Для иммутабельности — `{ get; init; }` или `private set`.  
_Senior add-on:_ в Release JIT инлайнит тривиальные аксессоры, перф-разницы нет.

**Model answer (EN):** A field is a memory slot; a property is a get/set method pair over it. I avoid public fields for two reasons: swapping a field for a property later is a binary breaking change — callers must recompile, since access goes from a memory read to a method call; and a property lets me add validation, lazy init, or change notification later, and is what data-binding, serializers and ORMs expect. For immutability I use `{ get; init; }` or `private set`.  
_Senior:_ in Release the JIT inlines trivial accessors, so there's no perf cost.

#### Q2. Что такое auto-property, во что компилируется? `(высокая)`

**Follow-up tree:** где backing field → можно ли к нему обратиться → _(trap)_ как сделать read-only-свойство, задаваемое только в конструкторе?

**Shallow / wrong:** «это сокращённая запись» — не знает про сгенерированный backing field.

**Модельный ответ (RU):** Компилятор генерит скрытый backing field (`<X>k__BackingField`, недоступен из кода) и тривиальные get/set. Read-only с заданием в ctor: `{ get; }` (присваивается только в конструкторе) или `{ get; init; }` (ещё и в object initializer).  
**(EN):** The compiler generates a hidden backing field and trivial get/set accessors. For read-only-but-settable-in-ctor use `{ get; }`, or `{ get; init; }` to also allow object initializers.

#### Q3. Как сделать immutable-объект? `(высокая, особенно EU)`

**Follow-up tree:** разница `private set` vs `init` → `init` даёт настоящую иммутабельность или это compile-time? → _(trap)_ можно ли обойти `init` рефлексией?

**Shallow / wrong:** «сделать поля readonly» — работает, но игнорирует `init`/`required` и сценарий с object initializer.

**Модельный ответ (RU):** `{ get; init; }` — задать можно только при создании (ctor или object initializer), дальше нельзя; `private set` разрешает мутацию изнутри класса в любой момент — это не настоящая иммутабельность. `required` (C# 11) заставляет инициализировать обязательные свойства. `init` — это ограничение уровня **компилятора** (set, помеченный `IsExternalInit`), не рантайма, так что рефлексией обойти можно.  
**(EN):** `{ get; init; }` can only be set during construction (ctor or object initializer); `private set` still allows internal mutation later, so it's not true immutability. `required` (C# 11) forces obligatory properties to be set. `init` is a compiler-level restriction, not a runtime one — reflection can bypass it.

#### Q4. Порядок: field initializers vs constructor body vs base constructor? `(средняя, любимый trap)`

**Follow-up tree:** что раньше — поле derived-класса или тело конструктора базы → _(trap)_ base ctor зовёт `virtual`-метод, переопределённый в derived и читающий derived-поле; что прочитает?

**Shallow / wrong:** «сначала база полностью, потом derived» — неверно для field initializers (derived-инициализаторы бегут раньше тела базы).

**Модельный ответ (RU):** Порядок: field initializers Derived → field initializers Base → тело Base ctor → тело Derived ctor. Поэтому поле Derived из инициализатора уже установлено к моменту тела базы, а заданное в теле Derived ctor — ещё нет. Virtual-вызов из base-конструктора прочитает derived-поля до того, как тело derived-ctor отработало — отсюда правило «не вызывай virtual из ctor».  
**(EN):** Order is: Derived field initializers → Base field initializers → Base ctor body → Derived ctor body. So a Derived field set by an initializer is already in place when the base body runs, but one set in the Derived ctor body is not. A virtual call from the base constructor sees the derived fields half-initialized — hence "don't call virtual from a constructor."

#### Q5. Default constructor: когда генерируется, class vs struct? `(средняя)`

**Follow-up tree:** можно ли убрать parameterless ctor у struct → _(trap)_ твой struct parameterless ctor вызовется при `new T[10]`?

**Shallow / wrong:** «компилятор всегда генерит пустой конструктор» — неверно для class после первого своего ctor, и неверно про массив для struct.

**Модельный ответ (RU):** class — авто-генерит public parameterless ctor только если не написан ни один свой. struct — всегда есть неявный занулящий ctor; с C# 10 можно написать свой parameterless с логикой, но при `default(T)` и `new T[]` он не вызывается, там чистое зануление.  
**(EN):** A class gets an implicit public parameterless ctor only if you declare none. A struct always has an implicit zeroing ctor; since C# 10 you can write your own parameterless one with logic, but `default(T)` and `new T[]` bypass it — they zero-initialize.

#### Q6. `this()` / `base()` chaining — зачем? `(средняя)`

**Follow-up tree:** разница `this` vs `base` → _(trap)_ при `: this(...)` field initializers выполнятся дважды?

**Модельный ответ (RU):** `: this(...)` — переиспользовать другой ctor этого класса, убрать дублирование; `: base(...)` — выбрать конкретный конструктор базы. При чейне через `this` field initializers выполняются один раз — в том ctor, что вызывает `base`, а не в каждом звене.  
**(EN):** `: this(...)` reuses another ctor of the same class to avoid duplication; `: base(...)` selects a base ctor. With `this`-chaining, field initializers run exactly once — in the ctor that calls `base`, not in every link.

#### Q7. Primary constructors (C# 12) — что это и где подвох? `(растущая)`

**Follow-up tree:** параметр primary ctor — это поле или свойство → _(trap)_ разница для record vs обычного class?

**Модельный ответ (RU):** В `record` позиционные параметры генерят public init-свойства. В обычном class/struct — нет: параметр просто в скоупе тела класса, компилятор сам решает, хранить ли в скрытом поле (если параметр используется в методах — захватит). Частая ошибка — ждать авто-свойства от primary ctor обычного класса.  
**(EN):** In a `record`, positional parameters generate public init properties. In a plain class/struct they don't — the parameter is just in scope across the class body, and the compiler captures it into a hidden field only if needed. The trap is expecting auto-properties from a non-record primary constructor.

#### Q8. Static constructor — когда вызывается? `(низкая, кросс-реф)`

Кратко: один раз, лениво, перед первым обращением к типу; связан с `beforefieldinit`. Полный разбор — глава `const`/`readonly`.