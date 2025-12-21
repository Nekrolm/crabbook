# Ссылки на реализации типажей

На код-ревью мне довольно часто попадался примерно следующий паттерн из структур

```rust
struct Request { /* неважно */ } 

trait Handler {
    fn handle(&mut self, r: Request);
}


struct RequestProcessor<'a, H: ?Sized> {
    /* какие-то данные и */
    handler: &'a mut H
}


impl<'a, H> RequestProcessor<'a, H>
where H : Handler + ?Sized {

    fn new(handler: &'a mut H) -> Self {
        Self { handler }
    }

    fn process(&mut self, r: Request) {
        /* ... */
        self.handle.handle(r);
    }
}
```

Возникает закономерный вопрос, а зачем там ссылка? Ну, нам очень сильно хочется явно сказать, что мы планируем использовать эту структуру как обертку и точно избежать копирований. Хорошо, допустим.

Дальше эта структура оказывается вложенной в другую структуру, например, такую

```rust

#[derive(Default)]
struct Service<'a> {
    handlers: Vec<RequestProcessor<'a, dyn Handler + Send>>
}

```

Вы уже чувствуете проблему? Нет, никакого неопределенного поведения или неправильной логики.

Lifetime-параметр инфецирует каждую структуру.

И в какой-то момент, когда слоев абстракций станет достаточно много и кто-то захочет сделать нечто подобное (запустить обработку в фоновом треде / асинхронной задаче)

```rust
let mut service = Service::default();
let mut h = HandlerImpl(42);
service.handlers.push(RequestProcessor::new(&mut h));
tokio::spawn(async move {
    service.run();
});
```

Borrow-checker Rust скажет, что он не согласен. Вы не правы 

```
error[E0597]: `h` does not live long enough
   --> src/main.rs:55:49
    |
 54 |       let mut h = HandlerImpl(42);
    |           ----- binding `h` declared here
 55 |       service.handlers.push(RequestProcessor::new(&mut h));
    |                                                   ^^^^^^ borrowed value does not live long enough
 56 | /     tokio::spawn(async move {
 57 | |         service.run();
 58 | |     });
    | |______- argument requires that `h` is borrowed for `'static`
 59 |       
 60 |   }
    |   - `h` dropped here while still borrowed
    |
note: requirement that the value outlives `'static` introduced here
   --> /playground/.cargo/registry/src/index.crates.io-1949cf8c6b5b557f/tokio-1.48.0/src/task/spawn.rs:171:28
    |
171 |         F: Future + Send + 'static,
    |                            ^^^^^^^
```

И это будет совершенно правильной ошибкой. Компилятор только что спас вас от выстрела в ногу через потенциально висячую ссылку. Потому как задача отправленная исполняться в фоне вполне может пережить скоуп текущей функции.
К сожалению, в силу сложности случая, компилятор не предложит никакого варианта решения и разработчику придется подумать своей собственной головой. Либо спросить своего любимого LLM-помощника.

Мне стало интересно узнать, а что же может предложить в качестве решения ChatGPT:

ChatGPT предложил изменить структуру `RequestProcessor`

```rust
struct RequestProcessor {
    handler: Box<dyn Handler + Send>,
}
```

Признаю, это решило бы проблему несчастного разработчика, столкнувшегося с всей этой возьней с временами жизни в высокоуровневого конца (у меня тут Service должен делать brr уже вчера, какие еще ссылки?!). И, быть может, он был бы совершенно доволен решением: компилируется -- работает -- запросы обрабатываются -- можно релизить и радовать пользователя.

Однако, такое решение ломает начальную идею автора структуры `RequestProcessor`. Теперь там всегда owned объект. Да еще и требующий дополнительной box-аллокации!

Разработчики, травмированные embedded или low latency разработкой, словят сердечный приступ или проведут остаток рабочей недели сокрушаясь и рассказывая, как невероятно долго времени занимает `malloc`. И в целом будут правы.

Мы можем сделать лучше. Rust имеет все возможности. Но для этого нужно немного расширить свое мировосприятие.

Допустим у нас есть generic-функция 

```rust
fn process_request<H: Handler>(hander: H, r: Request) {}
```

и переменная

```rust
let h: &mut dyn Handler = ...;
```

Можем ли мы вызвать? 

```rust
process_request(h, request)
```

НУ РАЗУМЕЕТСЯ НЕТ!

```
error[E0277]: the trait bound `&mut dyn Handler: Handler` is not satisfied
  --> src/lib.rs:50:21
   |
50 |     process_request(h, r);
   |     --------------- ^ the trait `Handler` is not implemented for `&mut dyn Handler`
   |     |
   |     required by a bound introduced by this call
```

В этот момент, я заметил, люди, пришедшие в Rust из C++ или Java, ломаются [^3]. 
Как так `&mut dyn Handler` не реализует `Handler`?! Тут же прямо в нем же и написано, что он `dyn Handler`!

В Java в принципе всё (кроме примитивных типов: `int`, `float`, и прочих) передается всегда по ссылкам и никакого специального синтаксиса и специальных типов не нужно. Так что недоумение понятно.

В C++ же ссылки не являются объектами. Нормальными объектами. Они представляют собой очень странный [^1] синтаксический сахар для ненулевых указателей.
А также они  **отбрасываются** в шаблонных параметрах:

```C++
// C++ concepts
template <Handler H>
void handle_request(H handler);

SomeHandlerImpl handler;
handle_request(handler); 
handle_request(std::as_const(handler));

// В обоих случах неявно ссылки будут отброшены и handler принят по значению
// т.е. скопирован
```


В Rust же ссылки являются полноправными самостоятельными объектами. Мы можем сделать массив ссылок. Мы можем менять ссылки (как в C мы меняем указатели, или как в Java ссылаемся на другой объект). Мы можем брать ссылки на ссылки!
У них свои типы. Которые просто так[^2] ни во что не трансформируются. Особенно при подстановке в generic функции.

Если у нас есть generic функция

```rust
fn process<T>(x: T) {}
```

И мы передаем в нее
```rust
let mut v : Vec<u8> = vec![1,2,3];
process(&mut v); // T = &mut Vec<u8>
process(&v);     // T = & Vec<u8>
process(v);      // T = Vec<u8>
```

Все три вызова будут работать с **разными** типами.

Точно также, если мы реализовали типаж для нашего *какого-то* типа,

```rust
struct MyHandler(i32);

impl Handler for MyHandler {
    fn handle(&mut self, r: Request) {
        println!("My handler {}", self.0);
    }
}
```
То только для этого `MyHandler` типа мы его и реализовали. Не для `&MyHandler` и не для `&mut MyHandler`.

И в ситуации выше `&mut dyn Handler` -- `dyn Handler` реализует `Handler`. А вот про ссылку нам ничего не обещали. Это другой тип. Реализация не предоставлена.

Так почему бы ее не предоставить?! Rust не делает этого автоматически, но мы всегда можем сделать вручную.

Это один из полезнейших и часто встречающихся шаблонов при описании абстрактных интерфейсов в Rust: предоставить реализацию для всех подходящих типов ссылок и стандартных умных указателей. Сделать это можно так

```rust
impl <H: Handler + ?Sized> Handler for &mut H {
    fn handle(&mut self, r: Request) {
        H::handle(self, r)
    }
}

impl <H: Handler + ?Sized> Handler for Box<H> {
    fn handle(&mut self, r: Request) {
        H::handle(self, r)
    }
}

// Для &H, Arc<H>, Rc<H> мы реализовать просто так не можем: требуется 
// &mut H, а мы уникальную ссылку из разделяемой безопасно не достанем
```
Очевидно, это довольно унылое занятие, если методов у нашего интерфейса очень много. Да еще и если все-таки реализовывать для разделяемых ссылок и умных указателей со счетчиками ссылок. Потому рекомендую завести макрос

```rust

macro_rules! impl_handler_for_ptr {
    (<$t:ident> for $ptr:ty) => {
        impl <$t: Handler + ?Sized> Handler for $ptr {
            fn handle(&mut self, r: Request) {
                $t::handle(self, r)
            }
        }
    }
}

impl_handler_for_ptr!(<H> for &mut H);
impl_handler_for_ptr!(<H> for Box<H>);
```

----

И теперь, если мы вернемся к `RequestProcessor` и просто уберем ссылку

```Rust
struct RequestProcessor<H> {
    /* какие-то данные и */
    handler: H
}

impl<H> RequestProcessor<H>
where H : Handler {

    fn new(handler: H) -> Self {
        Self { handler }
    }

    fn process(&mut self, r: Request) {
        /* ... */
        self.handle.handle(r);
    }
}
```

lifetime-парамет больше не будет расползаться по всей кодовой базе.

Бедолага с абстраткным сервисом сможет использовать `Box<dyn Handler + Send>`, как ему посоветовал ChatGPT

```rust
#[derive(Default)]
struct Service {
    handlers: Vec<RequestProcessor<Box<dyn Handler + Send>>
}
```

А те кто хотел передавать ссылки, избегать копий, минимизировать тем самым размер структур и проч. Все так же могут это делать

```Rust
let mut handler = MyHandler(42);
let processor = RequestProcessor(&mut hander);
```

И все счастливы. У нас вышел забавный пример парадокса изобреталеля:

Более обощенное решение оказывается проще чем узкоспециализированное. Но при этом покрывает его без каких-либо компромисов и проблем.

----

[^1]: В C++ они странные в плане поведения. Например, нельзя узнать размер самой ссылки напрямую. Проверка `static_assert(sizeof(const std::string&) == sizeof(const std::string*));` упадет -- потому как `sizeof(T&)` возвращает `sizeof(T)`. Но если обернуть ссылку в структуру `struct S { const std::string& s; };` то мы подтвердим, что размер ссылки совпадает с размером указателя (как и ожидется).
На ссылку нельзя взять ссылку или указатель. Нельзя узнать адресс, по которому хранится ссылка. При этом на указатель можно взять указатель. Но если ссылка завернута в структуру, то, чисто технически, взять ее адрес можно.
Эти спецэффекты создают огромное количество проблем, из-за которых, например,  поддержка `std::optional<T&>` появится только в C++26 (или позже).

[^2]: В Rust есть автоматическая [Deref coercion](https://doc.rust-lang.org/std/ops/trait.Deref.html), которая позволяет писать чуть меньше ручных преобразований ссылок. Так, например, мы можем передать `&String` в качестве аргумента функции, ожидающей `&str`. Потому что `String` реализует `Deref<Target = str>`. 
Для знакомых с хитростями операторов C++  ближайшей аналогией может быть перегрузка `operator ->`: запись `x->field` вызывает `->` у `x`. И продолжает применять ее снова и снова к каждому возвращенному типу, пока в конце-концов не получим сырой указатель. Или ошибку компиляции: [circular pointer delegation](https://godbolt.org/z/8WqsxrW81).
В Rust запись `x.field` тоже вызывает похожую цепочку обращений к `Deref` (или `DerefMut`).

[^3]: Вполне вероятно, пришедшие из Python и JavaScript тоже, но я с ними не работаю.