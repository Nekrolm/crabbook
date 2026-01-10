# Ссылочные типы и обобщенные функций

Вы разрабатываете библиотеку (например, стандартную) и она должна быть максимально гибкой, удобной, чтоб все ею пользовались, радовалиcь, ставили вам звездочки на github, а вы потом на конференциях... впрочем, я увлекся.

Библиотека. Обобщенная. Допустим, как пример, эта библиотека предоставляет фунции для сортировки массивов.
Какой интерфейс должен быть у функций сортировки?

В простейшем случае:

```rust
fn sort<T>(arr: &mut [T])
where T: Ord
```

сортируем все что можно упорядочить. 

В C++, Java, Go, Python и еще в куче языков есть такой или похожий вариант функции сортировки.

Но ведь не всегда есть единственный способ упорядочить объекты, да?
Довольно часто мы храним массивы из пользователей, товаров, операций, еще черт знает чего.

```rust
struct Order {
    price: i64,
    id: u64,
}

let orders: &mut [Order] = ....;

// Хочу отсортировать по цене, по убыванию
```

И на такой случай стандартные библиотеки языков программирования предоставляют перегрузку или альтернативную версию функции сортировки: принимающую компаратор -- функцию сравнения.

```rust
fn sort_by<T>(arr: &mut [T], comp: impl FnMut(&T, &T) -> Ordering)
```
В зависимости от дизайна, `Ordering` -- либо `bool`, говорящий меньше ли левый аргумент чем правый. Либо это результат 3-way-comparison: enum (или int), говорящий что левый аргумент больше (>0), меньше (<0) или равен (0) правому.

В стандартной библиотеке Rust используется 3-way-comparison. Так что, например, сортировка заказов по убыванию цены может выглядеть так

```rust
sort_by(orders, |lhs, rhs| lhs.price.cmp(&rhs.price).reverse())
```

Но это как-то некрасиво. Сортировка по какому-то полю (ключу) это очень частое явление и писать такую цепочку для нее -- слишком. Да еще и опечататься можно и сам с собой сравнить или поле не то взять.

В C++20, вместе с ranges пришла идея еще сильнее обобщить функции сортировок и использовать так называемые проекции ключей.

```C++
#include <ranges>
#include <utility>
#include <span>
#include <algorithm>
#include <print>

struct Order {
    uint64_t id;
    int64_t price;
};

void process(std::span<Order> orders) {
    std::ranges::sort(      // сортируем
        orders,             // заказы
        std::greater<>{},   // по убыванию (да, слегка нелепо)
        &Order::price       // цены
    );
}

int main() {

    Order orders[] = {
        {.id = 5, .price = 22},
        {.id = 7, .price = 44},
    };

    process(std::span(orders));

    for (auto order: orders) {
        std::println("id: {} price: {}", order.id, order.price);
    }

};
```

Красиво и довольно хорошо читаемо.

Хорошая практика перенимать хорошие практики из одного языка в другой. Так они обычно и развиваются.

Чем ответит Rust?

Попробуем хотя бы с проекциями

```rust
fn sort_by_key<T>(arr: &mut [T], key: impl FnMut(&T) -> impl Ord)
```

ой
```
error[E0562]: `impl Trait` is not allowed in the return type of `Fn` trait bounds
```

Ладно, возможно, когда-нибудь, так будет можно. Не отчаиваемся!

```rust
fn sort_by_key<T, K>(arr: &mut [T], key: impl FnMut(&T) -> K)
where K: Ord
```

Так можно. Отлично. Компилятор доволен. В стандартной библиотеке Rust именно такая функция, так что
можем написать

```Rust

sort_by_key(orders, 
            |order| Reverse(order.price));

```

и радоваться.

На этом все? Идеально удобное API готово?


Не спешите. Иногда бывает такое, что нужно, cкажем, города или страны в алфавитном порядке упорядочить. Чтоб людям при заполнении формочек их из списка выбирать удобнее было. А названия обычно строковые.

```rust

struct Order {
    country: String,
    code: u32,
}

fn process(orders: &mut [Order]) {
    sort_by_key(orders, |order| &order.country);
}
```

Красиво! Только borrow checker не доволен.

```
error: lifetime may not live long enough
  --> src/lib.rs:14:33
   |
14 |     sort_by_key(orders, |order| &order.country);
   |                          ------ ^^^^^^^^^^^^^^ returning this value requires that `'1` must outlive `'2`
   |                          |    |
   |                          |    return type of closure is &'2 String
   |                          has type `&'1 Order`
```

И что же пошло не так?!

```rust
// в сигнатуре функции key фигурирует ссылка -- а значит где-то должен быть и lifetime
fn sort_by_key<T, K>(arr: &mut [T], key: impl FnMut(&T) -> K)

// так что на самом деле сигнатура выглядит так
fn sort_by_key<T, K>(arr: &mut [T], key: impl for<'a> FnMut(&'a T) -> K)
```

По-умному и теоретико-типовому этот ужас `key: impl for <'a> FnMut(&'a T) -> K` в Rust зовется
_Higher-Rank Trait Bounds (HRTBs)_. 
А по простому рабоче-крестьянскому это следует понимать как:
**для любого 'a ** [^1] `key` реализует `FnMut(&'a T) -> K`.

Любители C++ похожий случай могли бы найти полезным для `template` параметров шаблонов

```C++
template <class A, class T>
concept AccessorOf = requires (A& accessor) {
    { accessor.get() } -> std::same_as<T&>;
};

template <class T>
struct AccessTracker {
    T& object;
    AccessTracker(T& obj) : object {obj} {}
    T& get() {
        std::println("accessed");
        return this->object;
    }
};

template <
    template <class T> class Accessor = std::reference_wrapper
> 
// хотелось бы выразить: для любого T, requires AccessorOf<
//        Accessor<T>,  T
// > 
void process() {
    std::vector<int> data;
    std::vector<std::string> names;
    auto data_accessor = Accessor(data);
    auto names_accessor = Accessor(names);
    data_accessor.get();
    names_accessor.get();
    // ...
}

void test() {
    process();
    process<AccessTracker>();
}
```

Возвращаясь к нашей ржавой проблеме:

```rust
`key: impl for <'a> FnMut(&'a T) -> K
```

Лайфтайм, принимаемый на входе произвольный, а возвращаемый тип -- зафиксирован и от лайфтайма не зависит.

```rust
// лайфтайм-аннотации не поддерживаются для замыканий
// это просто для наглядности:
sort_by_key(orders, for <'any> |order: &'any Order| -> &'fixed String { &order.country });
// 'fixed получен из ссылки на поле order, значит обязательно 'any : 'fixed 
// но 'any -- произволен. Удовлетворить это условие невозможно -> ошибка компиляции
```

Что же делать?!

Со стандартной библиотекой Rust -- просто не использовать `sort_by_key` для таких случаев. Использовать `sort_by`

Если же мы пишем свою версию, у нас есть некоторые варианты.

## 1. А что если зафиксировать лайфтайм?

```Rust
fn sort_by_key<'a, T, K>(arr: &mut [T], key: FnMut(&'a T) -> K) where K: Ord;
```

```
   Compiling playground v0.0.1 (/playground)
error[E0309]: the parameter type `T` may not live long enough
 --> src/lib.rs:3:59
  |
3 | fn sort_by_key<'a, T, K>(arr: &mut [T], mut key: impl FnMut(&'a T) -> K) where K: Ord {
  |                --                                ^^^^^^^^^^^^^^^^^ ...so that the reference type `&'a T` does not outlive the data it points at
  |                |
  |                the parameter type `T` must be valid for the lifetime `'a` as defined here...
  |
help: consider adding an explicit lifetime bound
  |
3 | fn sort_by_key<'a, T: 'a, K>(arr: &mut [T], mut key: impl FnMut(&'a T) -> K) where K: Ord {
```

Хорошо, добавляем. 
Проверим, например, простейшей сортировкой:

```rust
fn sort_by_key<'a, T: 'a, K>(arr: &mut [T], mut key: impl FnMut(&'a T) -> K) where K: Ord {
    for i in 0..arr.len() {
        for j in (i+1)..arr.len() {
            if key(&arr[j]) < key(&arr[i]) {
                arr.swap(i, j);
            }
        }
    }
}
```

```
error[E0621]: explicit lifetime required in the type of `arr`
 --> src/lib.rs:6:16
  |
6 |             if key(&arr[i]) < key(&arr[j]) {
  |                ^^^^^^^^^^^^ lifetime `'a` required
  |
help: add explicit lifetime `'a` to the type of `arr`
  |
3 | fn sort_by_key<'a, T: 'a, K>(arr: &'a mut [T], mut key: impl FnMut(&'a T) -> K) where K: Ord {
  |                                     ++

```

Понятные ошибки. Исправим.

```rust
fn sort_by_key<'a, T, K>(arr: &'a mut [T], mut key: impl FnMut(&'a T) -> K) 
where K: Ord,
// T:'a больше не нужно. Оно автоматически добвалено от слайса 
{
    for i in 0..arr.len() {
        for j in (i+1)..arr.len() {
            if key(&arr[j]) < key(&arr[i]) {
                arr.swap(i, j);
            }
        }
    }
}
```

```
error[E0502]: cannot borrow `*arr` as mutable because it is also borrowed as immutable
   --> src/lib.rs:9:17
    |
  3 | fn sort_by_key<'a, T, K>(arr: &'a mut [T], mut key: impl FnMut(&'a T) -> K) 
    |                -- lifetime `'a` defined here
...
  8 |             if key(&arr[j]) < key(&arr[i]) {
    |                ------------
    |                |   |
    |                |   immutable borrow occurs here
    |                argument requires that `arr[_]` is borrowed for `'a`
  9 |                 arr.swap(i, j);
    |                 ^^^^^^^^^^^^^^ mutable borrow occurs here
    |
note: requirement that the value outlives `'a` introduced here
   --> /playground/.rustup/toolchains/stable-x86_64-unknown-linux-gnu/lib/rustlib/src/rust/library/core/src/ops/function.rs:163:37
    |
163 | pub const trait FnMut<Args: Tuple>: FnOnce<Args> {
    |                                     ^^^^^^^^^^^^
```

На этом моменте начинается отчаяние. И могут возникнуть позывы к `unsafe {}`.
Но не спешите размахивать сырыми указателями!
Проблема в том что `impl FnMut(&'a T)` принимает ссылку, захватывает ассоцированный lifetime и не обязательно лишь возвращает объект-ссылку.
Вдруг какой-нибудь безумный гений передаст функцию:

```rust 
let mut storage : Option<&Order> = None;
sort_by_key(orders, |order|  {
    storage = Some(order);
    &order.country
})
```

Очевидно, что ссылка storage будет инвалидирована после любого обмена значений. Но с такой сингнатурой функции `sort_by_key` мы не получим ошибку на стороне вызова.

На этом этапе раскрывается еще одно тайное знание:

```Rust
key: impl FnMut(&'a T) // 'a фиксирован. key может куда-то сохранить полученную ссылку 

key: impl for<'a> FnMut(&'a T) // 'a произвольно меняется. Невозможно статически зафиксировать хранилище, куда можно было бы присвоить эту ссылку
```

**`for<'a> FnMut(&'a T)` -- HRTB - гарант того что переданную ссылку никуда не сохранят.**

У нас нет выбора! Для `sort_by_key` функция `key` просто обязана быть чем-то с HRTB `impl for<'a> SOMETHING<'a>`.
Варианты с фиксированными лайфтаймами открывают возможность нарушить safe гарантии [^2].

## 2. А что если спросить функцию ее возвращаемый тип?

В C++ мы можем написать `using R = decltype(projection(argument))` и сделать дальше что угодно с полученным типом.

Что если б мы могли написать что-то такое:

```Rust

fn sort_by_key<T, F>(arr: &mut [T], key: F)
where 
    for <'a> F: FnMut<&'a T>,
    for <'a> <F as FnMut<&'a T>>::Output: Ord

```

В Rust... В Rust встроенные `Fn*` типажи так не позволяют сделать. Во-первых, этот синтаксис для аргументов `Fn*` не стабилизирован.
Во-вторых, есть функции, параметризированные выходным типом. И он не зависит от входных типов! [^3]

```rust
fn produce<C>() -> C where C: FromIterator<i32> {
    [1,2,3,4,5].into_iter().collect()
}

fn test() {
    let v: Vec<_> = produce();
    let s: HashSet<_> = produce();
}
```

`produce` реализует бесконечно много вариантов: `for <C : FromIterator<i32>> Fn() -> C`. Какой из них вы хотите, запросив `Fn<>::Output`?.
Для случаев сортировки такая ситуация маловероятна, так что допустим, что нам такие функции в качестве проекций подавать не будут. А если будут -- то пользователь сам дурной.

--------------------------

Если со встроенными не получается, можем завести свои!

### Вариант 1

```rust
trait FnOutput<In> {
    type Output;
    fn call(&mut self, i: In) -> Self::Output;
}

// Реализуем сразу для вызываемых типов чтоб было удобно
impl <F, In, Out> FnOutput<In> for F 
where F: FnMut(In) -> Out {
    type Output = Out;
    fn call(&mut self, i: In) -> Out {
        self(i)
    }
}


fn sort_by_key<T, F>(arr: &mut [T], mut key: F)
where 
    for<'a> F: FnOutput<&'a T>,
    for<'a> <F as FnOutput<&'a T>>::Output: Ord
{
    for i in 0..arr.len() {
         for j in (i+1)..arr.len() {
             if key.call(&arr[j]) < key.call(&arr[i]) {
                 arr.swap(i, j);
             }
         }
    }
}
```

Попробуем посортировать

```rust
fn process(orders: &mut [Order]) {
    sort_by_key(orders, |ord| &ord.country);
}
```

```
error[E0282]: type annotations needed
  --> src/lib.rs:65:27
   |
65 |     sort_by_key1(orders, |ord| &ord.country);
   |                           ^^^   --- type must be known at this point
```

Легко!

```rust
sort_by_key(orders, |ord: &Order| &ord.country);
```

```
error: lifetime may not live long enough
  --> src/lib.rs:65:40
   |
65 |     sort_by_key(orders, |ord: &Order| &ord.country);
   |                               -     - ^^^^^^^^^^^^ returning this value requires that `'1` must outlive `'2`
   |                               |     |
   |                               |     return type of closure is &'2 String
   |                               let's call the lifetime of this reference `'1`
```

Кажется, мы пришли туда откуда начали... Та же самая ошибка... НЕТ! Мы пришли в совершенно другое место!

Если мы объявим функцию вместо лямбда-замыкания
```rust
fn country(x: &Order) -> &str {
    &x.country
}

sort_by_key(orders, country);
```

О чудо, оно скомпилируется и будет счастливо!

И дело не в фунцкии, а в том, что лямбда-замыкания в Rust прокляты. По [legacy причинам давно минувших дней](https://github.com/rust-lang/rust/issues/22557#issuecomment-77467069),
для `|order: &Order| -> &str { &order.country }` компилятор по умолчанию выводит тип `closure(&'a Order) -> &'b str` с фиксированным `'a` и `'b`, а не `for<'a> closure(&'a Order) -> &'a str`.

Можно принудить корректный вывод с помощью дополнительного шаманства с явным указанием типов
```rust
fn force_hrtb<F>(f: F) -> F where F: for<'a> FnMut(&'a Order) -> &'a str {
        f
}

sort_by_key(orders, force_hrtb(|order| { &order.country }));
```

Так наконец-то работает. Обобщенный макрос для подобного шаманства можно найти в [higher_order_closure](https://docs.rs/higher-order-closure/latest/higher_order_closure/).

Но, мне кажется, это не то что стоит вываливать на пользователя вашей библиотеки.


### Вариант 2

Забыть про `Fn*` типажи -- это перспективный путь.

Введем собственный типаж, специализированно для ключей сортировки.
И чтоб сигнатуру сортировки сильно не захламлять, захламим сам типаж: будем использовать Generic Associative Types (GATs) чтоб выразить желаемое: функция-проектор берет ссылку и может вернуть ссылочный тип.

```rust
trait KeyProjection<In> {
    // Выходной тип теперь параметризирован лайфтаймом
    type Out<'a>: Ord 
        where Self: 'a, In: 'a;

    fn project<'a>(&mut self, i: &'a In) -> Self::Out<'a>;
}
```

Функция сортировки становится не такой уродливой

```rust
fn sort_by_key<T>(arr: &mut [T], mut key: impl KeyProjection<T>)
{
    for i in 0..arr.len() {
         for j in (i+1)..arr.len() {
             if key.project(&arr[j]) < key.project(&arr[i]) {
                 arr.swap(i, j);
             }
         }
    }
}
```

И как теперь сортировать?

Можно сказать пользователю: "Реализуй свою проекцию и радуйся"

```rust

struct Country;
impl KeyProjection<Order> for Country {
    type Out<'a> = &'a str;
    fn project<'a>(&mut self, i: &'a Order) -> &'a str {
        &i.country
    }
}

sort_by_key(orders, Country);
```

Так много писать кода для частых случаев!

Можем предоставить пользователям вспомогательные функции

```Rust
struct FieldRef<F, Out>(F, PhantomData<Out>);
struct FieldVal<F, Out>(F, PhantomData<Out>);


fn field_ref<F, In, Out>(f: F) -> FieldRef<F, Out> 
where
    for <'a> F: FnMut(&'a In) -> &'a Out
{
    FieldRef(f, PhantomData)
}

fn field_val<F, In, Out>(f: F) -> FieldVal<F, Out>
where F: FnMut(&In) -> Out 
{
    FieldVal(f, PhantomData)
}


impl <In, F, Out: Ord> KeyProjection<In> for FieldRef<F, Out> 
where for <'a> F: FnMut(&'a In) -> &'a Out {

    type Out<'a> = &'a Out where In: 'a, F: 'a, Out : 'a;
    fn project<'a>(&mut self, i: &'a In) -> &'a Out {
        self.0(i)
    }
    
} 


impl <In, F, Out: Ord> KeyProjection<In> for FieldVal<F, Out> 
where F: FnMut(&In) -> Out {

    type Out<'a> = Out where In: 'a, F: 'a, Out : 'a;
    fn project<'a>(&mut self, i: &'a In) -> Out {
        self.0(i)
    }
} 
```

Так что они cмогут писать

```Rust
sort_by_key(orders, field_ref(|o: &Order| &o.country));
sort_by_key(orders, field_val(|o: &Order| o.code));
```

Теперь они счастливы?...

НЕТ! Я хочу сортировать страны в обратном порядке! Мне придется реализовать KeyProjection вручную и указать тип `Reverse<&'a str>`. А что если моя цепочка преобразований по вычислению ключа невероятно длинна? Мне придется указать ужасно длинный тип!

Хорошо, мы и здесь можем облегчить страдания пользователя.
Благодаря Return-position impl trait in trait (RPITIT)... и ценой чьего-то рассудка

```Rust
trait KeyProjection<In> {
    fn project<'a>(&mut self, i: &'a In) -> impl Ord + use<'a, Self, In> where Self: 'a;
}
```

Теперь пользователю не придется писать длинный конкретный тип. Он может написать длинный непрозрачный тип [^4].

```Rust
    use std::cmp::Reverse;
    struct Country;
    impl KeyProjection<Order> for Country {
        fn project<'a>(&mut self, i: &'a Order) -> impl Ord + use<'a> where Self: 'a {
            Reverse(&i.country)
        }
    }
```

Либо же мы можем пойти по пути C++ и просить и проекцию, и компаратор

```Rust

trait KeyProjection<In> {
    type Out<'a> where Self: 'a, In: 'a;
    fn project<'a>(&mut self, i: &'a In) -> Self::Out<'a>;
}

// field_val и field_ref остаются

trait Comparator<In> {
    fn cmp(&mut self, lhs: In, rhs: In) -> Ordering;
}

impl <In, F> Comparator<In> for F
where F: FnMut(In, In) -> Ordering {
    fn cmp(&mut self, lhs: In, rhs: In) -> Ordering {
        self(lhs, rhs)
    }
}


fn sort_by<T, K, C>(arr: &mut [T], mut key: K, mut compare: C)
where
    K: KeyProjection<T>,
    for<'a> C:  Comparator<K::Out<'a>>,
{
    for i in 0..arr.len() {
         for j in (i+1)..arr.len() {
             if compare.cmp(key.project(&arr[j]), key.project(&arr[i])) == Ordering::Less {
                 arr.swap(i, j);
             }
         }
    }
}


struct Ascending;
impl<T:Ord> Comparator<T> for Ascending {
    fn cmp(&mut self, lhs: T, rhs: T) -> Ordering {
        lhs.cmp(&rhs)
    }
}

struct Descending;
impl<T:Ord> Comparator<T> for Descending {
    fn cmp(&mut self, lhs: T, rhs: T) -> Ordering {
        lhs.cmp(&rhs).reverse()
    }
}
```

И тогда пользователь наконец-то сможет отсортировать страны в обратном порядке!

```Rust
    let orders: &mut [Order] = ...;

    sort_by(orders, 
            field_ref(|o: &Order| &o.country),
            Descending);
```

Поздравляю! Из сражений с временами жизни мы построили целый маленький, но гордый DSL (domain specific language)!

-----------------
Какой вывод из этого нужно сделать?

Разумеется, прежде всего: стандартный метод сортировки `std::slice::sort_by_key` остановился на `FnMut(&T) -> K` и все. Используйте `std::slice::sort_by` с компаратором и не морочьте себе голову.

Но самое главное: страдания только начинаются.

-----------------

Программистов очень часто посещает светлая идея "а передам-ка я вот сюда коллбэк и тем самым организую...".
Идея действительно светлая, очень много проблем с ее помощью можно решить. Потом еще столько же создать при отладке логики (знаменитый callback hell).
И в Rust это все тоже работает. И очень часто надо...

Но каждый раз когда вас посещает идея сделать с помощью `Fn*` типажей generic callback, который принимает ссылку и возвращает что-то, потенциально эту ссылку содержащее -- вы ступаете на путь боли и отчаяния.

Особенно болезненный опыт оказаться на этом пути случайно, запрыгнув в async Rust через какой-нибудь можный web-фреймворк.

Вам надо раздать ссылку на объект-запрос во множество мест и вы решили сделать "удобный"

```Rust
async fn add_reactor<F: Future<Output = ()>>(
    self, 
    f: impl Fn(&Request) -> F
)
```

Ваш фреймворк скомпилировался... и...
Поздравляю, вы прокляты -- add_reactor не работает ни для кого. Поскольку в 99.9% таких async коллбек функций `F` захватывает переданную ссылку. 

любой 
```rust
async fn do_something(request: &Request) {}
```
на самом деле

```rust
fn do_something<'a>(request: &'a Request) -> impl Future<Output = ()> + use<'a>
```

До стабилизации RPITIT и специальных AsyncFn traits [^5] единственным адекватным спасением было заворачивание такого безобразия в аллоцированную `Box<dyn Future<Output = ()> + 'a>` [^6]. Динамическая аллокация, динамическая диспетчеризация. Счастье и сердечный приступ для любителей выжимать наносекунды производительности.

```Rust

async fn add_reactor(
    self, 
    f: impl for<'a> Fn(&'a Request) -> Box<dyn Future<Output = ()> + 'a>
)

```

-----------------

Ипользование `Fn` типажей в API обобщенных библиотек, конечно, в большинстве случаев удобно (и для вас и для пользователя) и оправдано. И даже со ссылками и лайфтаймами там не всегда проблемы возникают: пока есть конкретные типы, которые эти ссылки / лайфтаймы держат, все хорошо:

```Rust

<T, U>         f: Fn(T) -> U; // прекрасно, lifetime аннотаций нет. МЫ СЧАСТЛИВЫ
<T>            f: Fn(&T) -> ConcreteType; // ok, lifetime аннотация дальше не распространяется
<'a, T:'a, U>  f: Fn(&'a T) -> U; // вероятно ok, но возможно это не то что вы хотите
<T, U>         f: Fn(&T) -> U;    // вы прокляты, получили ли вы то что хотели?   

```




-----------------

[^1]: Да, `for <'a>` это всего лишь квантор всеобщности. В настоящее время в Rust (1.93) его можно навешивать только для lifetime-параметров. Но, может быть, в будущем и для типов, и для константных-параметров поддержку все-таки доведут до стабильности.

[^2]: Есть еще и другой аспект, почему в `key`-функции должен быть нефиксированный лайфтайм аргумента: не все сортировки inplace. Сортировке с дополнительным буфером (merge sort) может потребоваться передавать ссылку на элемент во временном буфере. Время жизни этого буфера неизвестно. Их вообще может быть много разных.

[^3]: В C++ с помощью реализации `template <typename T> operator T()` можно достичь похожего результата 

[^4]: И пользователи будут получать веселое предупреждение: 
_warning: impl trait in impl method signature does not match trait method signature_

[^5]: У AsyncFn приходит другая проблема: на тип получаемой `impl Future` пока невозможно навесить дополнительных требований типа `+ Send + 'static`. Что делает их трудноприменимыми с многопоточными средами исполнения (tokio): один несчастный `tokio::spawn` задачи с `C: AsyncFn` глубоко внутри -- и вы пропали в тонне ошибок: cannot send between threads...

[^6]: Там еще `Pin` должен быть, но я его тут опущу.