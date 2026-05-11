# Функциональные типажи и поля структур

В [предыдущих главах](you_dont_want_drop.md) мы рассмотрели вариант, как точечно добавить `Drop` поведение к какому-либо полю структуры и не утратить возможность разбирать структуру на запчасти без каких-либо `Option`-состояний.

Мы переизобрели `DropGuard` вместо того чтоб взять готовый. Но у готового `DropGuard` есть проблема -- `FnOnce` типаж.

Возьмем тот же пример:

```Rust

struct NotUsedGuard(Option<ErrorHandle>);

...

impl Drop for NotUsedGuard {
    fn drop(&mut self) {
        let Self(Some(err)) = self else { return; };
        err.fail("Destroyed before usage");
    }
}

struct Gadget {
    data: Box<dyn Data>, 
    not_used: NotUsedGuard,
}
```

И попробуем использовать `std::mem::DropGuard`  (представим на минутку, что он stable) вместо самодельного `NotUsedGuard`

```Rust
struct Gadget {
    data: Box<dyn Data>,
    not_used: DropGuard<ErrorHandle, F>, // F=???
}
```

Что написать в качестве типа `F`, описывающего Drop-поведение?


Наше Drop-поведение, описывается функцией

```Rust
fn report_not_used(err: ErrorHandle) {
    err.fail("Destroyed before usage");
}
```

Как нам ее туда запихнуть?

Мы знаем, что в Rust для каждой функции компилятор заводит отдельный уникальный тип. Можем ли мы это использовать?
```rust

// может быть так?

struct Gadget {
    data: Box<dyn Data>,
    not_used: DropGuard<ErrorHandle, report_not_used>, // Compilation error
}
```
Увы, нет. В stable Rust пока нет возможности добраться до типа функции и назвать его 

Можно сделать так

```Rust
pub struct Gadget<F: FnOnce(ErrorHandle)> {
    data: Box<dyn Data>,
    not_used: DropGuard<ErrorHandle, F>,
}

pub fn new_gadget(data: Box<dyn Data>, err: ErrorHandle) -> impl Gadget<impl FnOnce(ErrorHandle)> {
    Gadget {
        data,
        not_used: DropGuard::new(err, report_not_used)
    }
}
```

Но пользователи публичной структуры `Gadget` проклянут вас за то, что вы инфицировали весь их код этим дженерик-параметром, и теперь у них время компиляции неадекватно растет


Можно использовать указатель на функцию:

```rust

pub struct Gadget {
    data: Box<dyn Data>,
    not_used: DropGuard<ErrorHandle, fn(ErrorHandle)>,
}

impl Gadget {
    pub fn new(data: Box<dyn Data>, err: ErrorHandle) -> Self {
        Self {
            data,
            not_used: DropGuard::new(err, report_not_used)
        }
    }
}
```

Но пользователи публичной структуры `Gadget`, заботящиеся о потреблении ресурсов и производительности, проклянут вас, ведь теперь `size_of::<DropGuard<ErrorHandle>, fn(ErrorHandle)>()   >    size_of::<ErrorHandle>()`. А мы в прошлой серии сделали все, чтоб никаких дополнительных байтиков не хранилось!

Да, обратите внивание на это:

```rust
fn foo() {
    println!("foo");
}

fn main() {
    let f = foo;
    println!("{}", std::mem::size_of_val(&f)); // тут 0 -- тип f -- некоторый zero-size тип impl Fn*

    let g: fn() = foo;
    println!("{}", std::mem::size_of_val(&g)); // тут 8 (на x64), fn() -- это самый обыкновенный указатель, как в C или С++
}
```

А может быть мы можем подложить собственную структуру, тип который мы можем назвать?

На stable Rust -- нет. Потому что детали типажа `trait FnOnce` [не стабилизированы](https://doc.rust-lang.org/stable/core/ops/trait.FnMut.html#tymethod.call_mut).
Это, может быть, когда-нибудь изменится, но вряд ли скоро.

Скорее раньше будет стабилизирована другая фича[^1] которая позволила бы добиться того же эффекта

```Rust

#![feature(impl_trait_in_assoc_type)]
#![feature(drop_guard)]

use std::mem::DropGuard;

...

trait DropBehavior {
    type OnDrop: FnOnce(ErrorHandle);
    fn make() -> Self::OnDrop;
}

struct NotUsed;

impl DropBehavior for NotUsed {
    // Вот такая штука нам поможет:
    type OnDrop = impl FnOnce(ErrorHandle);
    fn make() -> Self::OnDrop {
        report_not_used
    }   
}

pub struct Gadget {
    data: Box<dyn Data>,
    not_used: DropGuard<ErrorHandle, <NotUsed as DropBehavior>::OnDrop>,
}

impl Gadget {
    pub fn new(data: Box<dyn Data>, err: ErrorHandle) -> Self {
        Self {
            data,
            not_used: DropGuard::new(err, NotUsed::make())
        }
    }
}
```

Но это будет когда-нибудь потом. А пока для любой stable кодовой базы `Fn*` дженерик параметр является проблемой *в публичных структурах, точный тип которых пользователю будет нужен*. 

Например, типы итератор-комбинаторов:

```rust
// https://doc.rust-lang.org/std/iter/trait.Iterator.html#method.map
fn map<B, F>(self, f: F) -> Map<Self, F> ⓘ
where
    Self: Sized,
    F: FnMut(Self::Item) -> B,
```
В очень редких случаях кому-то потребуется завести поле типа `Map<<Vec<T> as IntoIterator>::Iter, MyFunctionType>`, но если потребуется -- они будут страдать.

Я заострил внимание на примере с `DropGuard`, потому что он гораздо более вероятно окажется в полях структуры -- вы не хотите писать `Drop` для всей структуры из-за его избыточно негативных эффектов.

Дополнительной мотивирующим примером может послужить `std::unique_ptr` из C++ -- один из подобных основных примитивов для точечной реализации деструкторов на месте, без определения деструкторов самому -- ведь в C++ вы тем более не хотите писать деструкторы вручную [^2]

```C++

#include <memory>

struct ErrHandle;

struct Deleter {
    // С C++23 стало возмозно оператор вызова пометить как static (а значит, скорее всего, stateless)
    static void operator()(ErrHandle*) {}
};

// Вариант с указателем на функцию также плох как в Rust
static_assert(sizeof(std::unique_ptr<ErrHandle, void(*)(ErrHandle*)>) > sizeof(ErrHandle*));
// Но в C++ всегда можно было указать пользовательскую структуру. И если она пустая -- не платить за нее
static_assert(sizeof(std::unique_ptr<ErrHandle, Deleter>) == sizeof(ErrHandle*));
// А с C++20 стало можно напрямую запихивать анонимные функции
static_assert(sizeof(std::unique_ptr<ErrHandle, decltype([](ErrHandle*){})>) == sizeof(ErrHandle*));

```

### Что же делать?

На самом деле, решение очень простое: использовать свой собственный типаж вызываемых объектов в публичном API структур

```Rust

trait DropBehavior<T> {
    fn on_drop(self, val: T);
}

impl<T, F: FnOnce(T)> DropBehavior<T> for F {
    fn on_drop(self, val: T) {
        self(val);
    }
}

pub struct DropGuard<T, F: DropBehavior<T>>(...);

impl<T, F: DropBehavior<T>> DropGuard<T, F> {
    pub fn new(val: T, on_drop: F) -> Self {
        ...
    }
}
```

И в таком виде пользователи `DropGuard` смогли бы 
- Продолжать использовать функции и замыкания на месте
- Указать свои собственные типы без плясок с бубнами вокрут unstable фич.

Не все могут позволить себе использовать `unstable` фичи -- стоит это учитывать при проектировании библиотек.

Немного странно просить такого от `DropGuard` из стандартной библиотеки (он пока и так unstable) -- зачем создавать еще один стандартный функциональный типаж? -- так можно в безумие многообразия C++ впасть...  
Но вот внешние реализации, например, [scopeguard](https://docs.rs/scopeguard/latest/scopeguard/), могли бы учесть проблему.



-----------

[^1]: На стабилизацию `impl_trait_in_assoc_type` в скором времени (ну, год или два...) есть большие надежды: ведь эта фича очень необходима для эффективного и удобного использования `async` Rust в типажах. Корпоративные гиганты активно используют `async` для своих веб сервисов, так что это одно из приоритетных направлений. А стабилизацию `fn_traits` может быть никогда и не увидим.

[^2]: Про правила нуля, трех и пяти стоит взглянуть на прекрасную таблицу Говарда Хиннанта -- https://accu.org/conf-docs/PDFs_2014/Howard_Hinnant_Accu_2014.pdf#page=30 
Страшная штука.