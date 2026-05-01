# Циклы событий и borrow checker

На Rust очень здорово и приятно писать простые консольные утилиты: прямолинейная логика, минимум абстракциий, вот данные, вот функция над ними -- все, красота.
И обычно при написании таких программ не возникает больших проблем в borrow checkerом: код простой, часто однопоточный -- спокойно таскаем `&mut Data` везде и наслаждаемся жизнью.

Асинхронные приложения на Rust тоже в целом здорово и приятно писать, если особо не задумываться об их производительности (ее из коробки большинству и так хватает): взял `axum`, `tokio`, `actix` и прочих друзей веб-программиста, завернул общее состояние в мьютекс (возможно, асинхронный) `Arc<Mutex<T>>` и поехал радоваться жизни, да через каналы сообщения передавать. А на все притензии borrow checkerа ответ простой: `.clone()` и ссылками поменьше пользуйся, а то там `tokio::spawn` в аргументах `'static` требует.

Но что если нам надо все-таки писать сложное, интерактивное приложение, расширяемое, над которым будет работать десяток человек + AI агенты. И ко всему прочему мы будет целится на ultra low latency?

## Чертова интерактивность

Что такое интерактивное приложение? Это приложение, которое интерактирует! То есть взаимодействует с пользователем. Причем взаимодействует не единоразово (прочитал аргументы командной строки и пошел по своим делам), а многоразово! Слушает команды, отвечает на них и так по кругу, пока пользователь не отключится или приложение не устанет от общения с бестолковым кожанным мешком...

Если взаимодействие с таким приложением полностью линеаризуемо, то есть его можно представить как запрос-ответ-запрос-ответ, такое приложение можно в целом реализовать как однопоточную, синхронную программу, которая выглядит как-то так

```rust
fn run() {
    let mut state = State::new();
    loop {
        let input = recv_input();
        let output = process(input, &mut state);
        send_output(output);
    }
}
```

Просто и понятно. И тоже никаких проблем с borrow checkerом тут не должно возникать.

Но очевидно что у такого приложения есть существенные ограничения:

- Пользователь должен ждать ответа прежде чем запросить следующую команду. И если `process` хоть сколько-нибудь заметно длителен, пользователь начнет нервничать, cуетиться и думать, что приложение зависло, ведь ничего же не приходит в ответ!
- Приложение не может никак сообщить пользователю, что оно занято, процесс идет, не волнуйтесь -- точка выдачи результата пользователю только одна и она еще не достигнута!
- И самое главное: такое приложение может обслуживать только одного пользователя.

Для устранения этих недостатков человечество придумало многозадачность: надо как-то организовать код программы в виде не (очень) зависимых задач, которые можно исполнять конкурентно. И как-то их конкурентно исполнять: хочешь и можешь параллельно -- исполняй параллельно, не можешь -- исполняй по очереди, переключайся между ними, делай же хоть что-нибудь!

Но что бы мы ни придумывали, в конечном итоге у нас будут

#### 1. Либо просто `N` физических или виртуальных потока исполнения, каждый из которых молотит свою задачу

```rust

fn input_routine(mut tx: Sender<Input>) {
    loop {
        let input = get_input();
        tx.send(input);
    }
}

fn process_routine(mut rx: Receiver<Input>, mut tx: Sender<Output>) {
    let mut state = State::new(); 
    loop {
        let input = rx.recv();
        // process будет отправлять результат по частям, если требуется
        process(input, &mut tx, &mut state);
    }
}

fn output_routine(mut rx: Receiver<Output>) {
    loop {
        let output = rx.recv();
        show_output(output)
    }
}

fn run() {
    let (in_tx, in_rx) = mpsc::channel(LIMIT);
    let (out_tx, out_rx) = mpsc::channel(LIMIT);
    thread::scope(move |mut scope|{
       scope.spawn(move || input_routine(in_tx));
       scope.spawn(move || process_routine(in_rx, out_tx));
       scope.spawn(move || output_routine(in_tx));
    });
}
```

Внимательный читатель, знакомый с `async` Rust, заметит, что этот примерный код легкой заменой блокирующих операций на асинхронные (например, из `tokio`), превращается 
в асинхронный, не блокирующий, потенциально работающий в одном лишь потоке, но структурно остающийся таким же -- потоки операционной системы заменяются на `async` стейт-машины, обрабатываемые `tokio` 

```rust
async fn input_routine(mut tx: Sender<Input>) {
    loop {
        let input = get_input().await;
        tx.send(input).await;
    }
}

async fn process_routine(mut rx: Receiver<Input>, mut tx: Sender<Output>) {
    let mut state = State::new(); 
    loop {
        let input = rx.recv().await;
        // process будет отправлять результат по частям, если требуется
        process(input, &mut tx, &mut state).await;
    }
}

async fn output_routine(mut rx: Receiver<Output>) {
    loop {
        let output = rx.recv().await;
        show_output(output).await;
    }
}

async fn run() {
    let (in_tx, in_rx) = tokio::sync::mpsc::channel(LIMIT);
    let (out_tx, out_rx) = tokio::sync::mpsc::channel(LIMIT);
    
    tokio::join!{
       tokio::spawn(input_routine(in_tx));
       tokio::spawn(process_routine(in_rx, out_tx));
       tokio::spawn(output_routine(in_tx));
    };
}
```

#### 2. Либо мы организуем цикл обработки событий вручную.

```rust

fn run() {
    let mut events = VecDeque::new();
    let mut state = State::new();
    loop {
        try_get_input(&mut events);
        let Some(event) = events.pop_front() else {
            continue;
        }
        match event {
            Event::Input(input)  => start_process(input, &mut events, &mut state),
            Event::Processing(data) => continue_process(data, &mut events, &mut state),
            Event::Output(output) => show_output(output),
        }
    }
}
```

Если вы знакомы с внутренностями `tokio` и концепциями за типажом `Future`, то знаете, что c `#[tokio::main(flavor = "current_thread")]` пример из варианта 1 фактически внутри превращается в вариант 2, с поправкой на то, что именно передается между функциями. 

Поэтому предлагаю сфокусироваться на ручном цикле обработки событий. У него много полезных свойств! 
Его всегда можно отмасштабировать на несколько потоков (с разделяемым мутабельным состоянием придется повозиться). 
Но главное -- последовательность событий можно легко линеаризовать и, при необходимости, записать и проигрывать в одном и том же порядке -- тем самым существенно облегчить отладку.


## Проклятые абстракции и расширяемость

Итак, интерактивность будем обеспечивать циклом событий. Обобщенно, красиво, с абстракциями, которые нам точно помогут, он может выглядеть так

```rust
enum HandleStatus {
    Accepted,
    Ignored,
}

trait Handler {
    fn handle_event(&mut self, event: &Event) -> HandleStatus;
}


let mut handlers = Handlers::new();
// конфигурируем множество обработчиков -- накидываем необходимые impl Handler
//  
let mut event_queue = EventQueue::new();
loop {
    if event_queue.poll(&mut handlers) == Done {
        break;
    } 
}
```

Примерно так бы оно выглядело почти на любом серьезном современном языке программирования. В том числе и на Rust. Дальше для изоляции логики
определяем отдельные типажи, которые будут обрабатывать конкретные варианты событий -- и мы счастливы!

```rust

trait InputHandler {
    fn handle_mouse(&mut self, m: &Mouse);
    fn handle_keyboard(&mut self, key: &Key);
}

trait DisplayHandler {
    fn display(&mut self, m: &ShowData);
}


// с некоторыми манипуляциями можно написать макрос, который бы позволил делать следующее

#[derive(Handler)]
#[events(InputHandler, DisplayHandler)]
struct MyHandler { ... }

impl InputHandler for MyHandler { ... }

impl DisplayHandler for MyHandler { ... } 
```

И красота! Никаких огромных типажей с миллионом методов, которые нужно реализовать вручную -- отличная разбивка на отдельные компоненты и подсистемы...

Но есть проблема -- разделяемое состояние.

Если обработчикам нужно как-то друг с другом взаимодействовать, то вся эта красота моментально встречается с реальностьюю borrow checkerа.

Хочешь, чтоб у двух разных обработчиков было пересекающееся состояние, выбирай:
- Объединяй их в один и разрушай свой прекрасный дворец из изолированных абстракций.
- `Rc<Cell<T>` в лучшем случая, или `Rc<RefCell<T>>` в худшем [^1], и плати за каждое обращение к общему состоянию runtime проверкой заимствований -- всего пара сравнений и операций чтения/записи в память.


Первый вариант плохой -- малейшие изменения будут разбегаться каскадом по огромному числу обработчиков, в конечном итоге все превратится в один большой божественный объект с гигантским состоянием и черт-знает-какой логикой -- какой метод что модифицирует?!

Второй вариант не blazing fast. А мы ведь на ultra low latency претендуем. Платить за бесполезную проверку, когда мы точно знаем, что у нас цикл событий, единовременно исполняется только один метод, состояние модифицирует ровно один объект -- зачем?

Что же делать?

## Отдельное состояние

Решение есть. Оно уже было в примерах выше -- нужно лишь отдать владение общим состоянием самому циклу обработки событий. А не пытаться его прикрутить к отдельным обработчикам -- лобовой подход, работающий в C++, Java, C#, Python, Go и прочих, противоестественен в Rust.


```rust
trait Handler {
    fn handle_event(&mut self, ctx: &mut State event: &Event) -> HandleStatus;
}


let mut handlers = Handlers::new();
let mut state = State::new();
// конфигурируем множество обработчиков -- накидываем необходимые impl Handler
//  
let mut event_queue = EventQueue::new();
loop {
    if event_queue.poll(&mut handlers, &mut state) == Done {
        break;
    } 
}
```
И все отдельные красивые обработчики тоже должны получить доступ к этому состоянию

```rust
trait InputHandler {
    fn handle_mouse(&mut self, ctx: &mut State, m: &Mouse);
    fn handle_keyboard(&mut self, ctx: &mut State, key: &Key);
}

trait DisplayHandler {
    fn display(&mut self, ctx: &mut State, m: &ShowData);
}
```

Красиво! Изящно! Никаких `RefCell` не потребуется и в целом код отражает ровно то, что задумывалось: каждый обработчик имеет эксклюзивный доступ к состоянию.

Но постойте... Мы же просто соорудили один огромный отдельный божественный объект... Да, это теперь не один большой обработчик, а одно большое состояние и отдельные операции над ними!

Да. Увы, но божественный объект -- это самый натуральный паттерн в Rust -- паттерн, который оставляет borrow checker счастливым. Тут главное определиться c природой его божественности. Каким богам он служит. А богов тут два.

### Всеми любимые ECS

Вот, скажем, есть Bevy -- игровой движок. Который внутри по сути, как и многие игровые движки, имеет цикл событий (ввод-вывод, таймеры), и предоставляет концептуально то же самое, что и мы тут изобретаем.

```rust
// вырезка из https://bevy.org/examples/camera/2d-screen-shake/

// обработчики объявляются как обычные функции.
// Этот запрашивает из разделяемого состояния некоторые компоненты
fn reset_transform(camera_shake: Single<(&CameraShakeState, &mut Transform)>) {
    let (camera_shake, mut transform) = camera_shake.into_inner();
    *transform = camera_shake.original_transform;
}

// этот обработчик инициализирует разделяемое состояние
fn setup_camera(mut commands: Commands) {
    // добавляет в него компоненты
    commands.spawn((
        Camera2d,
        CameraShakeConfig {
            ...
        },
    ));
}

fn main() {
    // инициализируем очередь
    App::new()
        // добаляем обрабочики
        .add_systems(Startup, (setup_scene ... ))
        ...
        // Добавляем обработчики
        .add_systems(PreUpdate, reset_transform)
         ...
         // запускаем цикл событий
        .run();
}
```

Есть обработчики, зовущиеся системами (System). Есть общее состояние, образованное сущностями (Entity), состоящими из разных компонентов (Components). Есть цикл событий. Entity - Component - System. ECS.

Каждая такая красивая система-функция всего-навсего превращается в объект, реализующий очень похожий типаж, почти наш `Handler`:

```rust 
https://docs.rs/bevy/latest/bevy/prelude/trait.System.html
trait System : ... {
    ...
    fn run(
        &mut self,
        input: <Self::In as SystemInput>::Inner<'_>,
        world: &mut World,
    ) -> Result<Self::Out, RunSystemError> { ... }
}

// примерно:
impl<F: FnMut(Args...)> System for F {
    ... 
    fn run(
        &mut self,
        input: <Self::In as SystemInput>::Inner<'_>,
        world: &mut World,
    ) -> Result<Self::Out, RunSystemError> { 
        // достать аргументы из world
        self(args...)
     }
}
```

`World` это наш `State` -- огромный, божественный объект. Но его содержимое специфично для конкретной программы и задается динамически. 
Это динамическая объектопомойка, которая скрыта от пользователей ECS причудливой эргономичной магией.

### Generic состояние.

ECSный `World` -- божественный объект динамической природы. А за динамическую природу нужно платить во время исполнения. Иногда значительно.

Мы можем запросить божественный объект другой природы -- статической. Просто надо снабдить обработчики дополнительным generic параметром

```rust

trait Handler<State> {
    fn handle_event(&mut self, ctx: &mut State event: &Event) -> HandleStatus;
}


trait InputHandler<State> {
    fn handle_mouse(&mut self, ctx: &mut State, m: &Mouse);
    fn handle_keyboard(&mut self, ctx: &mut State, key: &Key);
}

trait DisplayHandler<State> {
    fn display(&mut self, ctx: &mut State, m: &ShowData);
}

```

И теперь наше состояние может быть, как вы уже догадались, произвольной статической объектопомойкой.

И если некоторому обработчику ничего не нужно от состояния, его можно реализовать так

```rust
struct StandaloneInputHandler{...}

impl<S> InputHandler<S> for StandaloneInputHandler {
    fn handle_mouse(&mut self, _ : &mut S, m: &Mouse) { ... }
    ...
}
```

Можно описать некоторый дополнительный типаж для состояния, необходимого конкретному обработчику

```rust

trait TimeState {
    fn current_time(&mut self) -> Instant;
}

struct TimedDisplayHandler{...};

impl <S: TimeState> DisplayHandler<S> for TimedDisplayHandler {
    fn display(&mut self, ctx: &mut S, m: &ShowData) {
        if ctx.current_time() > ... {
            ...
        }
    }
}

```

Невероятные возможности по растаскиванию божественной радости на различные маленькие изолированные абстракции! [^2] Состояние при этом все равно останется огромным божественным объектом.

### А еще их можно объединить...

```rust

struct StateContext<S> {
    static_state: S,
    dynamic_state: ECSWorld,
}

impl<S> Deref for StateContext<S> {
    type Target = S;
    fn deref(&self) -> &S {
        &self.static_state
    }
}


impl<S> DerefMut for StateContext<S> {
    fn deref_mut(&mut self) -> &mut S {
        &mut self.static_state
    }
}


impl<S> StateContext<S> {
    fn dynamic(&self) -> &ECSWorld {
        &self.dynamic_state
    }
    fn dynamic_mut(&mut self) -> &mut ECSWorld {
        &mut self.dynamic_state
    }
}


trait Handler<State> {
    fn handle_event(&mut self, ctx: &mut StateContext<State> event: &Event) -> HandleStatus;
}

```

И теперь каждый обработчик может сам решить, кому из богов он поклоняется. Полная свобода.

Для горячего пути можно положить данные в `static_state`. Для всякой редкой ерунды - в `dynamic_state`. Красота.

## Проклятый async Rust  

Вот я не случайно провернул все это без `async`. Да, скажем, `tokio` отлично обставит нам свой внутренний цикл обработки событий и все будет замечательно... Но есть проблема -- общее состояние.

`async` в Rust был прикручен неестественным для Rust способом. И проблема не только в плясках с самореферентными типами и `Pin`, `Unpin`, `!Unpin`

```Rust
// асинхронная функция
async fn process(data: Input, state: &mut State) { ... }

// рассахаривается в
fn process<'s>(data: Input, state: &'s mut State) -> impl Future<Output = ()> + 's { async move {  ...  }  }
```

Ссылка на `state` будет захвачена на все протяжение существования объекта `impl Future`.
И если мы хотим запустить конкурентно две асинхронные задачи, которые мы будем по-очереди опрашивать в *однопоточном* цикле обработки событий, у нас нет никакой возможности **безопасно** разделить между ними состояние без привлечения `RefCell`, `Cell` и почих друзей с накладными runtime расходами. 
Каждая отдельная `impl Future` -- независимое вычисление, со своим собственным состоянимем. Почти как отдельный поток. Между ними всегда нужна какая-то синхронизация, если требуется разделять состояние.

В приложениях с собственным циклом обработки событий иногда было бы очень удобно описать какой-то процесс с помощью async Rust -- доверить компилятору генерацию машины состояний намного лучше, чем делать это вручную. Но без привлечения `unsafe` или отравления всего остального кода примитивами синхронизации, такие стейт-машины (`impl Future`) не смогут взаимодействовать с общим состоянием. И это печально.

### Привлечение unsafe

```Rust
pub trait Future {
    type Output;

    // Тут у нас есть Context
    fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output>;
}

// А в контексте есть Waker.
// А в Waker есть data
impl Waker {
  pub fn data(&self) -> *const () {
      self.waker.data
  }
}
```
План прост: засунуть `&mut StateContext<State>` в качестве `Waker::data` и радоваться.

Достать его изнутри асинхронных функций можно будет следущим хитрым приемом

```rust
async fn my_routine<State>() {
    let ctx std::future::poll_fn(|ctx| {
        let raw = ctx.waker().data();
        let state = raw.cast::<StateContext<State>>();
        // поехали! unsafe brr
    }).await;
} 
```

Полную реализацию этого ужаса я оставляю дорогому читаелю в качестве домашнего задания.

### Привлечение unstable фич

У Контекста, передаваемого в Future, есть unstable метод

```rust
https://doc.rust-lang.org/std/task/struct.Context.html#method.ext

impl Context<'_> {
    pub const fn ext(&mut self) -> &mut (dyn Any + 'static)
}
```

Вы догадываетесь, что мы можем туда положить?

## Дивный nightly мир

Простите, но у `Future` нет будущего в этом направлении. Но спасение есть!

Генераторы! Они же корутины.


```Rust
https://dev-doc.rust-lang.org/beta/std/ops/trait.Generator.html
pub trait Generator<R = ()> {
    type Yield;
    type Return;

    // Required method
    fn resume(
        self: Pin<&mut Self>,
        arg: R
    ) -> GeneratorState<Self::Yield, Self::Return>;
}

```

Вы видите то, что вижу я? Спасительный generic параметр `R`!

```rust

fn my_routine<State>() -> for<'a> impl Generator< &'a mut StateContext<State>  > {
    gen {
        ....
    }
}

```
Мы сможем передавать разделяемое состояние кажды раз, когда возобновляем работу стейт-машины! Ровно то что нужно!

Как только генераторы стабилизируют, наступит счастье. 
Но вряд ли это случится скоро...

Но спасение есть! Ваш покорный слуга ночами не спал, одержимый буйными мыслями, пока не написал ЭТО!

Смотрите!

```Rust
// https://github.com/Nekrolm/generator-light/blob/master/examples/printer.rs
fn list_printer<D: Display>(
    sep: impl Display,
) -> impl Generator<D, Yield = (), Return = Infallible> {
    generator(async move |mut yielder: Yielder<_, _>| {
        let mut item = suspend_!(yielder);
        print!("{item}");
        loop {
            item = yield_!(yielder, ());
            print!("{sep}{item}")
        }
    })
}

```

Это оно. Генераторы. Именно такие, какие можно прикрутить к циклу обработки событий с разделяемым состоянием. [^3]



-------------------


[^1]: Не обязательно `Rc`. Может быть и просто разделяемая ссылка `&`. Зависит от того, как будут храниться обработчики, и что мы хотим с ними делать. В любом случае, `Rc` это не проблема. Ведь скорее всего новых ссылок в процессе работы не будет создаваться.

[^2]: Главное потом успешно собрать такое состояние, что удовлетворит всем добавленным обработчиком. И не захлебнуться в потоке ошибок компиляции с сообщениями trait solver'a. 

[^3]: Непосредственно с `async` сахаром это, увы, не выйдет -- lifetime параметр у `Yielder<_, &mut StateContext<State>>` опять-таки будет захвачен и ничего не выйдет. Но генераторы можно собирать из отдельных, не async функций, и тогда в целом все может сработать