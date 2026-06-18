
## 0. Что такое SOLID

**Mental model.** SOLID — пять принципов объектно-ориентированного дизайна (сформулировал Robert C. Martin; акроним предложил Michael Feathers), которые управляют связностью и направлением зависимостей так, чтобы код можно было менять и расширять, не ломая существующее. Это эвристики под изменяемость, а не законы.

- **S** — модуль отвечает перед одним актором (одна причина меняться).

- **O** — расширяй поведение новым кодом, не трогая существующий.

- **L** — подтип подставляется вместо базового без потери корректности.

- **I** — не заставляй клиента зависеть от того, чем он не пользуется.

- **D** — завись от абстракций, не от деталей.

Общая нить всех пяти: изолировать изменения и развернуть зависимости к стабильным абстракциям.

---

## 1. SRP — Single Responsibility

  
**Mental model.** У класса одна причина для изменения, потому что он отвечает перед одним актором (стейкхолдером).

**How it works.**

«Причина для изменения» — это группа пользователей/стейкхолдеров, чьи требования заставляют код меняться. Если один класс обслуживает двух разных акторов, их требования столкнутся в одном файле: правка ради одного рискует сломать другого. Разносим код по причинам изменения.

```csharp

// Нарушение: три актора в одном классе

class Employee

{

public decimal CalculatePay() { /* правила бухгалтерии — актор: Finance */ }

public void Save() { /* схема БД — актор: DBA */ }

public string ReportHours() { /* формат отчёта — актор: HR */ }

}

  

// Фикс: один класс — один актор

class PayCalculator { public decimal CalculatePay(Employee e) { ... } }

class EmployeeRepository { public void Save(Employee e) { ... } }

class HourReporter { public string ReportHours(Employee e) { ... } }

```

Важно: SRP **не** значит «класс делает одну вещь» или «один метод». Класс может иметь много методов — критерий в том, что все они меняются **по одной причине**.

### 🔬 Глубже / на синьора

- Определение эволюционировало: исходно «one reason to change» → уточнено Мартином до «модуль ответственен перед одним и только одним актором». Формулировка через актора операциональна, «одна вещь» — нет.

- Связь с **cohesion**: SRP — это когезия по *причине изменения*, а не по функциональной близости. Метрика-родственник — LCOM.

### Interview layer

**Вопросы:** «Что такое SRP?» → FU: «что считать responsibility?» → **trap:** «значит, у класса должен быть один метод?»

**Слабый ответ:** «Класс должен делать одно дело» — нет операционального критерия, не отвечает на «что такое одно дело» и рассыпается на trap.

**Model answer (RU).** SRP — у модуля одна причина для изменения, то есть он отвечает перед одним актором. Это не про число методов: класс с десятком методов соблюдает SRP, если все они меняются по требованиям одного стейкхолдера. Нарушение — когда в одном классе сходятся, скажем, правила расчёта (Finance) и формат отчёта (HR): правка для одних ломает других.

**Model answer (EN).** SRP means a module has a single reason to change — it answers to one actor. It's not about method count: a class with many methods still satisfies SRP if they all change for one stakeholder's requirements. The violation is when, say, pay rules (Finance) and report formatting (HR) live in one class, so a change for one risks breaking the other.

---

## 2. OCP — Open/Closed

**Mental model.** Модуль открыт для расширения (новое поведение добавляется), но закрыт для модификации (существующий проверенный код не трогаем).

**How it works.**

Достигается через **абстракцию**: выделяем ось вариативности в интерфейс/абстрактный класс, и новые варианты добавляются новыми реализациями, а не правкой старого кода.

```csharp

// Нарушение: каждый новый shape правит существующий switch

double Area(object shape) => shape switch

{

Circle c => Math.PI * c.R * c.R,

Rectangle r => r.W * r.H,

_ => throw new ArgumentException() // добавил Triangle — лезу сюда снова

};

  

// OCP: новая фигура = новый класс, калькулятор не трогаем

interface IShape { double Area(); }

class Circle : IShape { public double Area() => Math.PI * R * R; }

class Triangle : IShape { public double Area() => 0.5 * B * H; } // расширение без модификации

```
### 🔬 Глубже / на синьора

- Две исторические трактовки: **Meyer (1988)** — закрытость через наследование (класс закрыт после компиляции, открыт через подклассы); **Martin** — полиморфная OCP через абстракции (фактически опирается на DIP). На интервью ждут вторую.

- Нельзя быть открытым ко *всем* осям изменения сразу. Ты выбираешь ось, которую ожидаешь варьировать (тип фигуры, способ оплаты). Угадал — расширение дёшево; не угадал — всё равно правишь существующее. Поэтому OCP — ставка, а не бесплатная гарантия.
### Interview layer

**Вопросы:** «Что такое OCP, как закрыть для модификации?» → **trap:** «это же просто наследование?»

**Слабый ответ:** «Используй наследование и override» — наследование лишь один механизм; суть в выделении абстрактной точки вариативности, а виртуальные методы — частность.

**Model answer (RU).** OCP — добавлять новое поведение, не меняя существующий код: выносим ось изменения в абстракцию, новые варианты делаем новыми реализациями. Классика — заменить разрастающийся `switch` по типу на полиморфизм/Strategy. Закрытость не абсолютна: ты выбираешь, по какой оси код будет открыт.

  

**Model answer (EN).** OCP means adding new behavior without editing existing code: extract the axis of change into an abstraction and add new variants as new implementations. The canonical move is replacing a growing type `switch` with polymorphism/Strategy. Closure isn't absolute — you pick the axis along which the code stays open.

---

  

## 3. LSP — Liskov Substitution

**Mental model.** Объект подтипа можно подставить везде, где ожидается базовый тип, не нарушив корректность программы. Наследуется не только сигнатура, но и **контракт** (behavioral subtyping).

**How it works.**

Правила подстановки (design by contract):

- предусловия подтипа **не сильнее** базовых (можно слабее — требовать меньше);

- постусловия **не слабее** базовых (можно сильнее — гарантировать больше);

- инварианты базового типа сохраняются;

- history constraint: подтип не допускает изменений состояния, запрещённых базовым.

Каноничное нарушение — `Square : Rectangle`:

```csharp

class Rectangle { public virtual int Width {get;set;} public virtual int Height {get;set;} }

class Square : Rectangle { // математически "квадрат — это прямоугольник"

public override int Width { set { base.Width = base.Height = value; } }

public override int Height { set { base.Width = base.Height = value; } }

}

  

void Test(Rectangle r) { // код, корректный для Rectangle:

r.Width = 5; r.Height = 4;

Debug.Assert(r.Width * r.Height == 20); // ломается, если r — Square (даст 16)

}

```

Геометрическое *is-a* ≠ поведенческое *is-a*: `Square` нарушает контракт «Width и Height независимы». Другие запахи: `override`, бросающий `NotSupportedException` (`Penguin.Fly()`); подтип, усиливающий предусловие (требует больше аргументов/состояния, чем базовый). Фикс — не наследовать там, где контракт расходится (например, общий `IShape` без сеттеров-ловушек, либо immutable-типы).

### 🔬 Глубже / на синьора

- Идея — Liskov (1987), формализована в Liskov & Wing, *A Behavioral Notion of Subtyping* (1994).

- Связь с вариантностью: параметры по теории **контравариантны**, возвращаемые значения **ковариантны** — поэтому covariant return types (C# 9) безопасны по LSP (сужение результата не ломает вызывающего), а вот сужение типа параметра — сломало бы. (C# на практике требует точного совпадения типа параметра в `override`, поэтому контравариантность тут теоретическая.)

- `NotSupportedException`/`NotImplementedException` в переопределении — почти всегда явный маркер нарушения LSP.

### Interview layer

**Вопросы:** «LSP, пример нарушения?» → FU: «что с пред/постусловиями?» → **trap:** «но ведь квадрат — это прямоугольник, почему нельзя наследовать?»

**Слабый ответ:** «Наследник не должен ломать базовый класс» — по сути верно, но без механики (контракт, пред/постусловия, инварианты) не проходит follow-up.


**Model answer (RU).** LSP — объект подтипа должен подставляться вместо базового, не нарушая корректности: подтип не усиливает предусловия, не ослабляет постусловия и сохраняет инварианты. Square/Rectangle — нарушение, потому что Square ломает контракт независимости сторон: код, рассчитанный на Rectangle, начинает врать. Математическое «is-a» не равно поведенческому.

**Model answer (EN).** LSP means a subtype must be substitutable for its base without breaking correctness: it must not strengthen preconditions, weaken postconditions, or violate invariants. Square/Rectangle breaks it because Square violates the "independent sides" contract, so code written against Rectangle starts giving wrong results. A mathematical "is-a" isn't a behavioral "is-a."

---

## 4. ISP — Interface Segregation

**Mental model.** Клиент не должен зависеть от членов интерфейса, которыми он не пользуется; лучше несколько узких ролевых интерфейсов, чем один «толстый».

**How it works.**

Толстый интерфейс заставляет реализаторов писать заглушки, а клиентов — зависеть (и перекомпилироваться) из-за изменений в членах, которые им не нужны. Дробим по ролям клиента.

```csharp

// Нарушение: старый принтер вынужден реализовывать Scan/Fax заглушками

interface IMultifunctionDevice { void Print(); void Scan(); void Fax(); }

class OldPrinter : IMultifunctionDevice {

public void Print() { ... }

public void Scan() => throw new NotSupportedException(); // ещё и LSP-запах

public void Fax() => throw new NotSupportedException();

}

  

// ISP: ролевые интерфейсы

interface IPrinter { void Print(); }

interface IScanner { void Scan(); }

class OldPrinter : IPrinter { public void Print() { ... } } // зависит только от нужного

```
### 🔬 Глубже / на синьора

- **ISP vs SRP:** SRP смотрит со стороны *класса* — сколько у него причин меняться; ISP — со стороны *клиента* — от скольких лишних членов он вынужден зависеть. Часто ведут к похожему дроблению, но мотивация разная (производитель vs потребитель).

- Связь с code smell «fat interface» и понятием role interface (Fowler).

### Interview layer

**Вопросы:** «Что такое ISP?» → FU: «ISP vs SRP?»

**Слабый ответ:** «Интерфейсы должны быть маленькими» — мельчить ради мельчения не цель; цель — не навязывать клиенту зависимость от лишнего.

**Model answer (RU).** ISP — не заставлять клиента зависеть от членов интерфейса, которые он не использует. Толстый интерфейс плодит заглушки у реализаторов и лишние зависимости у клиентов; решение — дробить на ролевые интерфейсы под конкретных клиентов. От SRP отличается точкой зрения: ISP про связанность клиента с интерфейсом, SRP про причины изменения класса.

**Model answer (EN).** ISP means clients shouldn't be forced to depend on interface members they don't use. A fat interface breeds stub implementations and needless client coupling; the fix is splitting it into role interfaces shaped by each client. It differs from SRP in viewpoint: ISP is about a client's coupling to an interface, SRP about a class's reasons to change.

---
## 5. DIP — Dependency Inversion

**Mental model.** И высокоуровневые, и низкоуровневые модули зависят от абстракции; саму абстракцию определяет потребитель (политика), а не деталь. «Инверсия» — стрелка `High → Low` разворачивается к абстракции.

**How it works.**

Обычно политика (high-level) напрямую создаёт и вызывает конкретную деталь (low-level) — high жёстко зависит от low. DIP вводит абстракцию, от которой зависит high; low её реализует. Теперь high не знает о конкретной детали, и деталь можно подменить.

```csharp

// Нарушение: high-level зависит от конкретной детали

class PasswordReminder { private readonly MySqlConnection _db = new(); /* ... */ }

  

// DIP: оба зависят от абстракции, деталь внедряется извне

interface IDbConnection { ... } // абстракция, нужная потребителю

class MySqlConnection : IDbConnection { ... } // деталь реализует абстракцию

class PasswordReminder {

private readonly IDbConnection _db;

public PasswordReminder(IDbConnection db) => _db = db; // зависимость подаётся снаружи

}

```

Ключ: абстракцию формирует high-level слой под свои нужды (как «порт»), а не low-level. Иначе интерфейс просто повторит деталь — и инверсии не будет.
![[Pasted image 20260618151233.png]]

### 🔬 Глубже / на синьора

**DIP ≠ DI ≠ IoC** — главная ловушка:

- **DIP** — *принцип* дизайна: завись от абстракций.

- **DI (Dependency Injection)** — *техника* подачи зависимости извне (constructor/property/method injection); один из способов соблюсти DIP.

- **IoC (Inversion of Control)** — общий *концепт*: поток управления отдаётся фреймворку («Hollywood principle: don't call us, we'll call you»). DI — частный случай IoC применительно к зависимостям.

  

Следствия: DIP можно соблюдать без всякого контейнера — просто передать интерфейс в конструктор руками. И наоборот, можно использовать DI-контейнер и при этом нарушать DIP, если инжектишь конкретные типы. Кто владеет абстракцией: в чистом DIP интерфейс объявлен в слое потребителя, а реализация — снаружи.

  

### Interview layer

**Вопросы:** «Что такое DIP?» → **trap:** «это про DI-контейнер?» → FU: «разница DIP / DI / IoC?»

**Слабый ответ:** «DIP — это использовать DI-контейнер» — путает принцип с инструментом; не звучит про абстракцию и владение ею.

**Model answer (RU).** DIP — высокоуровневые модули не зависят от низкоуровневых напрямую; оба зависят от абстракции, которую определяет слой-потребитель. Это не про DI-контейнер: DI — лишь техника подачи зависимости, а IoC — более общий концепт «фреймворк вызывает тебя». Можно соблюдать DIP вручную и нарушать его с контейнером.

*(senior add-on)* Важно, кто владеет абстракцией: интерфейс принадлежит политике, деталь его реализует — иначе инверсии нет.

  

**Model answer (EN).** DIP means high-level modules don't depend on low-level ones directly; both depend on an abstraction owned by the consuming layer. It's not about a DI container: DI is just a technique for supplying a dependency, and IoC is the broader "the framework calls you" concept. You can follow DIP by hand and violate it with a container.

*(senior add-on)* What matters is who owns the abstraction: the interface belongs to the policy, the detail implements it — otherwise there's no inversion.

---
## 6. 🔬 Когда SOLID вредит (на синьора)

Полностью bonus-слой. SOLID — эвристики под *ожидаемую* изменяемость, и слепое применение даёт over-engineering: интерфейсы с единственным реализатором «на всякий случай», дробление, после которого по коду тяжелее навигировать, абстракции ради абстракций. Если ось изменения не наступает, гибкость — чистый оверхед (YAGNI). Зрелый ответ на «всегда ли применяешь SOLID»: применяю там, где реально ожидаю изменение или переиспользование, и не плачу за гибкость, которая не понадобится. Критика принципов (расплывчатость формулировок, риск карго-культа) — нормальная часть синьорского разговора, а не ересь.