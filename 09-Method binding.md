### Зачем эта глава

Слово-ловушка здесь — «полиморфизм». На собесе под ним мешают две разные вещи: overloading с overriding, и `new` с `override`. Оба узла развязываются одним вопросом: **когда выбирается метод — на компиляции или в рантайме?** Overloading решается компилятором по статическим типам ещё до запуска; виртуальная диспетчеризация — рантаймом по фактическому типу объекта. Интервьюер бьёт по этому двумя типами вопросов: «что выведет код» (про dispatch) и «как компилятор выбирает перегрузку» (про resolution). Глава строит модель времени связывания, из которой оба ответа выводятся.

### Mental model

**Overloading — это выбор сигнатуры на компиляции по статическим типам. Virtual dispatch — выбор реализации в рантайме по фактическому типу через vtable.**

||Overloading|Virtual dispatch|
|---|---|---|
|Когда|компиляция (early binding)|рантайм (late binding)|
|По чему|статические типы аргументов|фактический тип объекта|
|Механизм|overload resolution|vtable (method table)|
|Чем управляешь|сигнатурами|`virtual` / `override` / `abstract`|

Всё дальнейшее — это раскрытие двух колонок и разбор того, что происходит, когда их путают (`new`) или совмещают (перегруженный + виртуальный метод).

![[Pasted image 20260623105241.png]]

### 1. Overloading + overload resolution

**Overloading** — несколько методов с одним именем и разным списком параметров (по типу, числу или порядку). Различать перегрузки по **return type нельзя**. По модификаторам передачи: by-value vs by-ref (`ref`/`out`/`in`) — **можно**, а только `ref` vs `out` (различие лишь в kind) — **нельзя** (CS0663).

**Overload resolution** — как компилятор выбирает, простыми словами: собирает кандидатов с этим именем → оставляет **applicable** (каждый аргумент неявно конвертируется в тип параметра) → выбирает **самый специфичный** (better function member — грубо, тот, чьи параметры требуют наименьших/наиболее точных конверсий). Если единственного лучшего нет — ошибка `ambiguous`; если applicable нет вообще — «нет подходящей перегрузки». Всё это происходит **на компиляции по статическим типам** аргументов.

> **🔬 Глубже: ловушки resolution.**
> 
> - `Console.WriteLine(null)` → **ошибка компиляции** (CS0121): `null` подходит и к `string`, и к `char[]` (оба специфичнее `object`), но между собой они несравнимы → ambiguous. Лечится кастом: `WriteLine((string)null)`.
> - Числовые литералы: при `F(long)` vs `F(double)` вызов `F(1)` выберет `long` (конверсия `int`→`long` «лучше», чем `int`→`double`); при наличии `F(int)` — точное совпадение выигрывает.
> - `params`: неразвёрнутая форма предпочтительнее развёрнутой `params`-формы.
> - `dynamic`: если хоть один аргумент `dynamic`, разрешение перегрузки **откладывается в рантайм**.
> - Instance-методы приоритетнее extension-методов: extension рассматривается, только если нет applicable instance-метода.

### 2. virtual / override + vtable

`virtual` делает метод переопределяемым, `override` подменяет реализацию в derived, а **какую реализацию вызвать — решает рантайм по фактическому типу** объекта. Механизм — vtable (method table): у каждого типа есть таблица, виртуальные методы занимают в ней слоты. Виртуальный вызов (`callvirt`) в рантайме смотрит слот в vtable **фактического** типа объекта. Когда derived `override`-ит метод, он **заменяет адрес в том же слоте** на свою реализацию — поэтому даже через ссылку базового типа выполняется derived-версия. Это и есть runtime-полиморфизм. `override` возможен, только если базовый член `virtual`, `abstract` или сам `override`.

> **🔬 Глубже.** Виртуальные вызовы компилируются в `callvirt` (поиск по vtable + null-check), невиртуальные — в `call`. Индекс слота фиксируется на компиляции, а адрес в нём разрешается в рантайме по типу объекта — отсюда и «позднее связывание».

### 3. new (hiding) — dispatch по статическому типу

`new` не переопределяет, а **скрывает**: он не трогает слот базового метода в vtable, а заводит **отдельный** член. Поэтому вызов через ссылку идёт по **статическому** (compile-time) типу этой ссылки — компилятор привязывается к члену, видимому на статическом типе, никакой подмены слота не происходит. Классический «что выведет»:

csharp

```csharp
class Base { public virtual string Who() => "Base"; }
class Over : Base { public override string Who() => "Over"; }
class Hide : Base { public new      string Who() => "Hide"; }

Base b1 = new Over();  b1.Who();  // "Over" — override: dispatch по фактическому типу
Base b2 = new Hide();  b2.Who();  // "Base" — new: dispatch по статическому типу (Base)
Hide h  = new Hide();  h.Who();   // "Hide" — статический тип Hide
```

![[Pasted image 20260623105301.png]]

`new` почти всегда — не то, что нужно. Единственный легитимный кейс — **версионирование библиотек**: базовый класс из чужой сборки добавил метод с твоей сигнатурой, и `new` явно говорит «я сознательно скрываю», убирая предупреждение компилятора.

### 4. abstract-методы

`abstract`-метод — это **virtual без тела**, объявленный в abstract-классе: он занимает слот vtable, который наследник **обязан заполнить** через `override` (иначе наследник сам остаётся abstract, а для конкретного класса — ошибка компиляции). Не может быть `private` или `static`, не имеет реализации. По сути это «virtual, у которого базовой реализации нет вообще, поэтому переопределение не опционально, а принудительно».

### 5. base

`base.M()` вызывает базовую реализацию **напрямую** — это **невиртуальный** вызов (`call`, не `callvirt`), он обходит vtable. Поэтому внутри `override` можно дёрнуть `base.M()`, чтобы **расширить**, а не заменить базовое поведение (частый паттерн: derived делает своё + `base.M()`). Тем же словом вызываются конструкторы базового класса (`base(...)`). Важно для follow-up: нет, это не виртуальный вызов — он жёстко привязан к базовому типу на компиляции.

### 6. sealed + devirtualization

`sealed`-класс нельзя наследовать; `sealed override` (`public sealed override void M()`) переопределяет метод, но **запрещает дальнейшее** переопределение ниже по иерархии. Применение — зафиксировать критичную реализацию, закрыть точку расширения.

> **🔬 Глубже: devirtualization.** Когда JIT может **доказать точный тип** объекта (переменная sealed-типа, sealed-метод, либо анализ пинит тип сразу после `new`), он заменяет виртуальный `callvirt` на прямой `call`, убирая индирекцию vtable, и часто **инлайнит** метод. `sealed` даёт JIT эту уверенность дёшево. Современный JIT умеет и **guarded devirtualization**: спекулирует наиболее вероятный тип и делает прямой вызов с проверкой-страховкой на случай промаха. Это оптимизация, а не гарантия, — но именно она стоит за «sealed может быть быстрее».

### 7. Когда метод и перегружен, и виртуальный

Самый аккуратный синтез главы. Если метод одновременно имеет несколько перегрузок и виртуален, происходят **два разных решения в два разных момента**:

csharp

```csharp
class Base
{
    public virtual void M(object o) => Console.WriteLine("object");
    public virtual void M(string s) => Console.WriteLine("string");
}

Base b = new Base();
object x = "hi";
b.M(x);   // "object" — перегрузка выбрана на компиляции по СТАТИЧЕСКОМУ типу (object)
```

Какая **перегрузка** (сигнатура) — выбирается на компиляции по статическим типам аргументов. Какая **реализация** выбранной сигнатуры — выбирается в рантайме по фактическому типу объекта через vtable. Поэтому `string`, лежащий в переменной типа `object`, уедет в `M(object)`, хотя значение — строка. Формула: **overloading by static type, overriding by runtime type**.

### Interview layer

Вопросы по убыванию частоты. Формат: вопрос → likely follow-up → trap; затем shallow-ответ (почему плох) и tight model answer (RU + EN).

**Q1 (очень часто). Разница overloading vs overriding?**  
→ FU: когда выбирается каждый?  
→ trap: а если метод и перегружен, и виртуальный?

- _Shallow:_ «overloading — разные параметры, overriding — переопределение». Не сказано про время связывания — а это весь смысл.
- _RU:_ «Overloading — несколько методов с одним именем и разными параметрами; какой вызвать, решает компилятор на компиляции по статическим типам аргументов (compile-time полиморфизм). Overriding — derived заменяет реализацию virtual-метода; какую вызвать, решает рантайм по фактическому типу через vtable. Ключ — разное время связывания.»
- _EN:_ «Overloading is several same-named methods with different parameters; the compiler picks one at compile time by the static argument types. Overriding is a derived class replacing a base virtual method's implementation; the runtime picks which by the object's actual type via the vtable. The key is different binding time.»

**Q2 (очень часто). Что выведет код с `new` vs `override`?**  
→ trap: через базовую ссылку.

- _Shallow:_ «оба переопределяют». Неверно — `new` не переопределяет.
- _RU:_ «Через базовую ссылку `override` вызовет derived-реализацию (dispatch по фактическому типу — слот vtable заменён), а `new` — базовую (dispatch по статическому типу ссылки, потому что `new` не трогает слот, а добавляет отдельный член). `Base b = new Hide(); b.Who()` → "Base"; с override → "Over".»
- _EN:_ «Through a base reference, `override` calls the derived implementation (dispatch by runtime type — the vtable slot is replaced), while `new` calls the base one (dispatch by the reference's static type, since `new` adds a separate member instead of touching the slot). `Base b = new Hide(); b.Who()` → "Base"; with override → "Over".»

**Q3 (часто). Как работает virtual dispatch / что такое vtable?**  
→ FU: что `override` делает со слотом?

- _RU:_ «У каждого типа есть method table (vtable); виртуальные методы занимают слоты. Вызов `callvirt` в рантайме смотрит слот в vtable фактического типа объекта. `override` заменяет адрес в слоте на derived-реализацию, поэтому даже через базовую ссылку бежит derived.»
- _EN:_ «Each type has a method table (vtable); virtual methods occupy slots. A `callvirt` looks the slot up in the actual object's vtable at runtime. `override` replaces the slot's address with the derived implementation, so even through a base reference the derived one runs.»

**Q4 (часто). `new` vs `override` — разница и когда `new` оправдан?**

- _RU:_ «`override` — настоящее переопределение через vtable (рантайм-полиморфизм). `new` — сокрытие: отдельный член, dispatch по статическому типу. `new` почти всегда не нужен; легитимный кейс — версионирование библиотек: базовый класс из чужой сборки добавил метод с твоей сигнатурой, и `new` явно фиксирует "сознательно скрываю".»
- _EN:_ «`override` is real overriding via the vtable (runtime polymorphism). `new` is hiding: a separate member, dispatch by static type. `new` is almost never needed; the legit case is library versioning — a base class from another assembly adds a method matching your signature, and `new` explicitly states "I intend to hide."»

**Q5 (mid). Что делает `base.M()`?**  
→ trap: это виртуальный вызов?

- _RU:_ «`base.M()` вызывает базовую реализацию напрямую — невиртуальный вызов (`call`, не `callvirt`), обходит vtable. Применяется внутри override, чтобы расширить, а не заменить базовое поведение. Нет, это не виртуальный вызов — он привязан к базовому типу на компиляции.»
- _EN:_ «`base.M()` calls the base implementation directly — a non-virtual call (`call`, not `callvirt`) that bypasses the vtable. Used inside an override to extend rather than replace base behavior. No, it's not a virtual call — it's bound to the base type at compile time.»

**Q6 (mid). `sealed` — зачем и как влияет на перформанс?**

- _RU:_ «`sealed`-класс нельзя наследовать; `sealed override` стопает дальнейшее переопределение. Перф: при запечатанном типе/методе JIT знает точный тип и может девиртуализировать — заменить виртуальный вызов прямым и заинлайнить, убрав индирекцию vtable. Это оптимизация JIT, не гарантия.»
- _EN:_ «A sealed class can't be inherited; `sealed override` stops further overriding. Perf: with a sealed type/method the JIT knows the exact type and can devirtualize — replace the virtual call with a direct call and inline it, removing the vtable indirection. It's a JIT optimization, not a guarantee.»

**Q7 (mid+). Как компилятор выбирает перегрузку?**  
→ trap: `Console.WriteLine(null)`; `dynamic`.

- _RU:_ «Собирает кандидатов с этим именем → оставляет applicable (аргументы неявно конвертируются в параметры) → выбирает самый специфичный (better function member); ничья → ошибка ambiguous. `Console.WriteLine(null)` ambiguous между `string` и `char[]` (нужен каст). Аргумент `dynamic` откладывает разрешение в рантайм.»
- _EN:_ «It gathers candidates with that name → keeps the applicable ones (arguments implicitly convert to parameters) → picks the most specific (better function member); a tie is an ambiguous error. `Console.WriteLine(null)` is ambiguous between `string` and `char[]` (needs a cast). A `dynamic` argument defers resolution to runtime.»

**Q8 (mid+). Метод перегружен И виртуальный — как резолвится вызов?**

- _RU:_ «Два решения в два момента: какая перегрузка (сигнатура) — на компиляции по статическим типам; какая реализация выбранной сигнатуры — в рантайме по фактическому типу через vtable. Поэтому `string` в переменной типа `object` уедет в `M(object)`. Формула: overloading by static type, overriding by runtime type.»
- _EN:_ «Two decisions at two times: which overload (signature) — at compile time by static types; which implementation of that signature — at runtime by the actual type via the vtable. So a `string` held in an `object` variable goes to `M(object)`. Formula: overloading by static type, overriding by runtime type.»

**Q9 (🔬). Можно перегрузить только по `ref`/`out`?**

- _RU:_ «По by-value vs by-ref (`ref`/`out`/`in`) — можно. Различать только `ref` vs `out` (различие лишь в kind) — нельзя, CS0663.»
- _EN:_ «By-value vs by-ref (`ref`/`out`/`in`) — yes. Distinguishing only `ref` vs `out` (kind only) — no, CS0663.»

**Q10 (🔬). Когда JIT девиртуализирует?**

- _RU:_ «Когда доказан точный тип: переменная sealed-типа, sealed-метод, либо анализ пинит тип (например, сразу после `new`). Тогда `callvirt` → `call` + инлайн. Современный JIT умеет guarded devirtualization — спекуляция вероятного типа с проверкой-страховкой.»
- _EN:_ «When the exact type is proven: a variable of a sealed type, a sealed method, or analysis pins the type (e.g., right after `new`). Then `callvirt` becomes `call` + inline. The modern JIT also does guarded devirtualization — speculating the likely type with a fallback type check.»