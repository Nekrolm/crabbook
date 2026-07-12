# Опасная вариантность

Лайфтаймы в Rust обычно берутся от ссылок. Напрямую или через другие структуры, содержащие ссылки (или их подобия).

```rust
let s = "hello".to_string();
let sref = &s; // sref имеет тип &'a String. 
```

Причем что такое `'a` в этом примере -- на самом деле сильно зависит от реализации borrow checkerа. А их несколько! Но для простых смертных в принципе не должно быть разницы в большинстве случаев. Не важно, как его определяют внутри -- как множество строк кода/область видимости, где ссылка валидна. Или как хитрое множество конкретных точек заимствований (Polonius)[^1].

Мы же сейчас сосредоточимся на том, что лайфтайм-параметр -- это часть типа. А типы в Rust иногда неявно приводятся друг к другу (coercion [^4]) -- в основном для сохранения психики тех, кто не хочет бороться с лайфтаймами в самых обыденных случаях.

Например,

```rust

let service = "Service".to_string();
let field_name: &'static str = "timing";
// names имеет тип Vec<&'a str>, где 'a -- не превосходит области валидности
// строки-объекта service
let mut names : Vec<&str> = vec![&service];
// сигнатура метода push
// fn push(self: &mut Vec<T>, v: T)
// т.е, в этом случая
// fn push(self: &mut Vec<&'a str>, v: &'a str); С некоторым фиксированным 'a 
names.push(field_name);
// Но у field_name тип &'static str!. &'static str != &'a str
println!("{names:?}")
```

Этот пример успешно компилируется, как и ожидается любым пользователем, кому просто нужно чтоб работало и решаело его проблему.
Он успеешно компилируется, поскольку 

`&'static str` это *подтип* для `&'a str` -- потому что `'static` обязан *пережить* любой другой `'a` лайфтайм. [^2]
И в Rust выполняются неявные преобразования в таком случае: `&'static str` *сужается* до `&'a str`.

Причем подобное сужение происходит не только непосредственно для ссылок:

```rust
fn extend(v: &mut Vec<Box<[&str]>>, extra: Box<[&'static str]>) {
    v.push(extra);
}
```

Такой пример тоже скомпилируется: `Box<[&'static str]>` оказывается подтипом для любого другого `Box<[&'a str]>`.
`Box<T> : Box<U>` если `T: U`. Или, как говорят, `Box` *ковариантен* (сохраняет направление наследования) относительно своего параметра. [^3]

Этой вводной части, думаю, нам должно быть достаточно. Но обратите внимание, что во всех примерах выше, неявному преобразованию подвергался только добавляемый аргумент. `&mut T` -- *инвариантен* по `T`. Из соображений корректности. Это нам еще пригодится.


## Unbounded лайфтайм

Иногда ссылки, откуда брать лайфтайм, нет. А взять его откуда-то надо. Обычно такое встречается в unsafe коде, но и для safe кода тоже попадаются случаи.

Вот, скажем, хотим мы завести функцию, которая всегда одну и ту же строку выдает, сгенерированную на этапе компиляции:

```rust
fn service_name() -> &str {
    "Service"
}
```
А оно не компилируется, ведь возвращаемой ссылке в сигнатуре функции просто необходимо откуда-то взять лайфтайм-параметр
```   Compiling playground v0.0.1 (/playground)
error[E0106]: missing lifetime specifier
  --> src/lib.rs:33:22
   |
33 | fn service_name() -> &str {
   |                      ^ expected named lifetime parameter
   |
   = help: this function's return type contains a borrowed value, but there is no value for it to be borrowed from
help: consider using the `'static` lifetime, but this is uncommon unless you're returning a borrowed value from a `const` or a `static`
   |
33 | fn service_name() -> &'static str {
   |                       +++++++
help: instead, you are more likely to want to return an owned value
   |
33 - fn service_name() -> &str {
33 + fn service_name() -> String {
```

Предложение от компилятора дельное:
```rust
fn service_name() -> &'static str {
    "Service"
}
```
И проблема решена... Или мы создали другую проблему?

Рассмотрим пример:

```rust
fn service_name() -> &'static str {
    "Service"
}

fn test_it() {
    let service = "Service".to_string();
    let mut names : Vec<&str> = vec![&service];
    names.resize_with(10, service_name);
}
```

Корректен ли он в принципе? Должен ли он скомпилироваться?

У `resize_with` сигнатура
`fn resize_with(self: &mut Vec<T>, f: impl FnMut() -> T)`
С конкретными типами
`fn resize_with(self: &mut Vec<&'a str>, f: impl FnMut() -> &'a str)`

`service_name` это `impl FnMut() -> &'static str`. [^5]

Функции в разумном мире, обычно, *ковариантны* по возвращаемому праметру. Да и в конкретном случае: это должно быть корректным -- добавлять долгоживущие ссылки в контейнер с короткоживущими. Даже через посредника.

Но оно не скомпилируется, увы.

```
error[E0597]: `service` does not live long enough
    --> src/lib.rs:44:38
     |
  43 |     let service = "Service".to_string();
     |         ------- binding `service` declared here
  44 |     let mut names : Vec<&str> = vec![&service];
     |                                      ^^^^^^^^ borrowed value does not live long enough
  45 |     names.resize_with(10, service_name);
     |     ----------------------------------- argument requires that `service` is borrowed for `'static`
  46 |     // resize_with_default(&mut names, service_name);
  47 | }
     | - `service` dropped here while still borrowed
     |
note: requirement that the value outlives `'static` introduced here
    --> /playground/.rustup/toolchains/stable-x86_64-unknown-linux-gnu/lib/rustlib/src/rust/library/alloc/src/vec/mod.rs:3174:23
     |
3174 |         F: FnMut() -> T,
```

Как это ни странно, но из `F: Fn() -> &'static str` не следует что `F: Fn() -> &'a str`.
В Rust, в настоящее время (1.97), типажи **инвариантны** по своим параметрам и ассоциированным типам.

```rust
fn f1<'a>(v: &mut Vec<&'a str>, f: impl FnOnce() -> &'a str) {}

fn f2<'a>(v: &mut Vec<&'a str>, f: impl FnOnce() -> &'static str) {
    f1(v, f)
}

fn f3<'a>(v: &mut Vec<&'a str>, f: impl FnOnce() -> &'static str) {
    // А вот так сработает -- мы создали новый тип, который уже удовлетворяет требованию,
    // так как полученный &'static str можно привести к &'a str
    f1(v, || f())
}
```

Так что более общим решением, которое не попадет в подобную неприятную ситуацию, будет использование *unbounded* параметра для функции
`service_name`:

```rust
fn service_name<'a>() -> &'a str {
    "Service"
}
```

## Очередь сообщений

В стандартной библиотеке Rust есть много всякого добра. Например, там есть удобные каналы для передачи сообщений между компонентами -- `std::sync::mpsc::channel`.

Пусть у нас буду сообщения

```rust
struct Message <'a> {
   id: u32, 
   description: &'a str,
}
```

И мы желаем отправлять их из двух разных потоков:
- Поток A будет слать сообщения, у которых `description` ссылаются на динамические строки, вычисленные в runtime.
- Поток B будет слать сообщения, у которых `desctiption` ссылаются на статические строки  

И нам очень бы хотелось использовать один и тот же канал для этого, а не два разных.

Да не вопрос!

```rust
use std::sync::mpsc::*;

struct Message <'a> {
   id: u32, 
   description: &'a str,
}


fn t1<'a>(_storage: &'a Vec<String>, _s: Sender<Message<'a>>) {}
fn t2(_s: Sender<Message<'_>>) {}


fn run1() {

    let storage = vec!["descr".to_string()];
    let (tx, _rx) = channel();
    std::thread::scope(|s| {
        s.spawn(|| t1(&storage, tx.clone()));
        s.spawn(|| t2(tx.clone()));
    });
}
```

Действительно. Если мы можнем отправить один и тот же lifetime параметр в оба потока, все просто.
Усложним задачу! `t2` теперь обязательно требует `'static`, от которого никак избавиться нельзя.

Рассмотрим функцию отправки сообщения:

```rust
fn add_message<'a, 'b: 'a>(s: &Sender<Message<'a>>, m: Message<'b>) {
    s.send(m);
}
```
Очевидно, она должна компилироваться (и она компилируется). Ведь ровно как и в самом начале -- не должно быть никаких проблем для добавления долгоживущих ссылок в контейнер с короткоживущими.

Значит, и вот так должно работать

```rust
fn add_message<'a>(s: &Sender<Message<'a>>, m: Message<'static>) {
    s.send(m);
}
```

Да и в целом, как будто бы выглядит, что должно быть безопасно и корректно просто преобразовать
```rust
fn t2(s: Sender<Message<'_>>) { 
   let other_thread_sender: Sender<Message<'static>> = s;
   std::thread::spawn(move || { other_thread_sender; });
}
```
Ведь мы ж не сможем через него больше короткоживущие ссылки отправить! Безопасно же!

`Sender` ж ведет себя как функция `F(Message<'a>)`, а функции, как известно, *контравариантны* по своим аргументам. Все же сходится!

```rust
fn t1<'a>(_storage: &'a Vec<String>, _s: Sender<Message<'a>>) {}
fn t2(_s: Sender<Message<'static>>) {}


fn run() {

    let storage = vec!["descr".to_string()];
    let (tx, _rx) = channel();
    t1(&storage, tx.clone());
    t2(tx.clone());
}
```

Увы, но нет.

```
error[E0597]: `storage` does not live long enough
  --> src/lib.rs:19:8
   |
17 |     let storage = vec!["descr".to_string()];
   |         ------- binding `storage` declared here
18 |     let (tx, _rx) = channel();
19 |     t1(&storage, tx.clone());
   |        ^^^^^^^^ borrowed value does not live long enough
20 |     t2(tx.clone());
   |     -------------- argument requires that `storage` is borrowed for `'static`
21 | }
   | - `storage` dropped here while still borrowed
```

Ух эта стандартная библиотека! Надо просто взять все в свои руки, немного `unsafe {}`, и сделать свой собственный **контравариантный** канал.

Я не буду мешать вам реализовывать свой собственный канал. Я просто покажу, почему не все так просто, и о каком ужасном краевом случае нужно не забыть.

Допустим, вам удалось реализовать свой канал:

```rust

fn channel<T>() -> (Sender<T>, Receiver<T>);
```

где `Sender` контравариантен по `T`.

Рассмотрим такой простенький тип для сообщения:

```rust
struct Message<'a>(&'a str);
impl Drop for Message<'_> {
    fn drop(&mut self) {
        println!("{}", self.0);
    }
}
```

```rust
fn t1(s: Sender<Message<'static>>) {
    std::thread::spawn(move || { s.send(Message("hello")); std::thread::sleep(Duration::from_millis(10)); });
}

fn run() {
    let message = "world".to_string();
    let (tx, rx) = channel();
    t1(tx.clone());
    tx.send(Message(&message));
    drop(rx);
    drop(tx);
    drop(message); 
    // Если к этому моменту &message все еще находится во внутреннем буфере канала, живущего в t1
    // мы получим use-after-free в деструкторе Message
}
```
Такой код [скомпилируется](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&gist=df407aaa28ba3f9d20a92794aa3ff5f1)

Неожиданные деструкторы все время портят жизнь!

В качестве домашнего задания можете подумать, есть ли безопасный способ все-таки реализовать контравариантный канал:
- Для однопоточного кода (Sender: !Send)
- Для многопоточного кода


[^1]: https://smallcultfollowing.com/babysteps/blog/2018/04/27/an-alias-based-formulation-of-the-borrow-checker/
[^2]: Мне кажется, это одна из самых путающих и сбивающих с толку штук, когда пытаешься понять или объяснить отношение вложенности типов с лайфтаймами. Вот есть у нас `class Animal` и `class Dog : Animal`. `Dog` это подкласс/подтип (т.е. что-то более конкретное) для `Animal`. `Dog` *вложен* в `Animal`. С лайфтаймами как бы та же самая запись: `static : 'a`. И `&static str` это подтип для `&'a str`, но мы говорим **наоборот**: `'static` *шире* чем `'a`. `'static` *переживет* `'a` -- просто потому как **множество** строк кода, где валиден `'static`, включает в себя **множество** строк, где валиден `'a`. Вот так Rust подложил нам (если бросаться умными терминами) неявную *контра*-варинатность, от которой у многих болит голова.   
[^3]: https://doc.rust-lang.org/nomicon/subtyping.html
[^4]: https://doc.rust-lang.org/reference/type-coercions.html#coercion-types
[^5]: Каждая функция в Rust имеет свой уникальный анонимный тип `impl Fn(Arg) -> R`. Этот тип можно неявно привести к указателю на функцию `fn(Arg) -> R`.  Но указателем на функцию он не является.