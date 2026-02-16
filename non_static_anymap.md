# Не 'static AnyMap

Любите ли вы AnyMap так как люблю его я?
Замечательный тип и паттерн, который позволяет принести все счастье динамически типизированного ада в наш светлый статически типизированный рай. 

Популярные библиотеки, реализующие http-клиенты и серверы, очень часто снабжают структуры `Request` / `Response` особым полем `Extensions` -- в которое можно запихнуть что угодно.
Строители ECS[^1] фреймворков глубоко внутри используют AnyMap в том или ином виде, чтоб в него, как на свалку, пользователи могли закинуть (и достать) что им угодно, разделив данные и логику их обработки.

В простейшем виде `AnyMap` выглядит так:

```rust
#[derive(Default)]
struct AnyMap {
    table: HashMap<TypeId, Box<dyn Any>>
}

impl AnyMap {

    fn put<S: Any>(&mut self, x: S) {
        self.table.insert(TypeId::of::<S>(), Box::new(x));
    }

    fn get<S: Any>(&self) -> Option<&S> {
        self.table.get(&TypeId::of::<S>()).and_then(|s| s.downcast_ref())
    }

    fn get_mut<S: Any>(&mut self) -> Option<&mut S> {
        self.table.get_mut(&TypeId::of::<S>()).and_then(|s| s.downcast_mut())
    }
}
```

И все что реализует типаж `Any` можно в нее положить:

```rust
let mut m = AnyMap::default();
m.put("hello_world".to_string());
println!("{:?}", m.get::<String>());
```

Разумеется у этой красоты есть куча недостатков:
- Производительность из-за RTTI[^2] в любой форме -- нужны поиск и проверка типов во время исполнения
- Неиллюзорный шанс потратить пару часов отлаживая, почему вот в этой части проекта я ж положил в нее тип `X`, а в другой части проекта что-то его нету (В лучшем случае его просто кто-то вынул. А в худшем -- у вас две версии типа `X`, если он к вам пришел от какой-то из библиотек-зависимостей).

Но есть еще и проблема в применимости: `AnyMap` в таком виде не может ни в коем случае безопасно хранить не-`'static` типы: типаж `T: Any` требует `T: 'static`. И ничего с этим ты не сделаешь...

Или сделаешь?

Всегда можно соорудить свой собственный RTTI. Но что будет выступать в качестве TypeId? Кто их будет генерировать и поддерживать? Разрешить пользователям назначать их самостоятельно? Это все рискованно.

В Rust, со стабилизацией Generic Associated Types, можно сделать лучше.

Нам нужно всего лишь ввести пару взаимнообратных мета-функций:

`Owner:       E<'a> -> O`
`Element<'a>: O     -> E<'a>`

Таких что 

`for<'a> Element<'a> Owner E<'a> == E<'a>`
`for<'a> Owner Element<'a> O     == O`


Тогда тип `Owner` может однозначно определить тип `Element` и наоборот.


А дальше все просто:

```rust

// Хранилищу нужно лишь уметь элементы освобождать.
// Можно, конечно, использовать `Drop`, но не все типы реализуют Drop.
// Поэтому просто воткнем вспомогательный типаж, который реализуют все.
trait AnyDrop {}
impl <T> AnyDrop for T {}

struct AnyMap<'a> {
    // делаем lifetime параметр инвариантным -- это безопасный вариант,
    // чтоб никто в нашу AnyMap с долгоживущими ссылками короткоживущую не засунул
    invariant_lt: PhantomData<UnsafeCell<&'a mut ()>>,
    // ключ TypeId -- TypeId::of<Owner>
    // А хранимый элемент -- Element<'a>
    table: HashMap<TypeId, Box<dyn AnyDrop + 'a>>
}
```

Метафункции -- это будут два специальных типажа

```rust
trait Owner: 'static {
    type Element<'a>: Element<'a, Owner = Self>;
}

trait Element<'a>: 'a {
    type Owner: Owner<Element<'a> = Self>;
}
```

Советую приостановиться и помедитировать над этим. Это буквально запись уравнений выше, но на языке Generic Associative Types в Rust.

После медитации, реализовать методы `put` и `get` довольно просто

```rust

impl<'a> AnyMap<'a> {
    fn put<E: Element<'a>>(&mut self, x: E) {
        self.table.insert(TypeId::of::<E::Owner>(), Box::new(x));
    }

    fn get<E: Element<'a>>(&self) -> Option<&E> {
        let boxed = self.table.get(&TypeId::of::<E::Owner>())?;
        let any: &(dyn AnyDrop + 'a) = Box::deref(boxed);
         // Safety: E::Owner::Element == E
        unsafe { (any as *const (dyn AnyDrop + 'a) as *const <E::Owner as Owner>::Element<'a>).as_ref() }
    }
    
    fn get_mut<E: Element<'a>>(&mut self) -> Option<&mut E> {
        let boxed = self.table.get_mut(&TypeId::of::<E::Owner>())?;
        let any: &mut (dyn AnyDrop + 'a) = Box::deref_mut(boxed);
        // Safety: E::Owner::Element == E
        unsafe { (any as *mut (dyn AnyDrop + 'a) as *mut <E::Owner as Owner>::Element<'a>).as_mut() }
    }
}

```

Ну что, можем начинать запихивать в этот новый `AnyMap` все подряд?

К сожалению, теперь надо все-таки регистрировать типы прежде чем их засовывать куда попало. Но это не сложно


```rust
// Для Owned типов
impl Owner for String {
    type Element<'a> = Self;
}

impl<'a> Element<'a> for String {
    type Owner = Self
} 
```

```rust
// Для типов с лайфтаймами
struct CowStrOTag;

impl Owner for CowStrOTag {
    type Element<'a> = Cow<'a, str>;
}

impl<'a> Element<'a> for Cow<'a, str> {
    type Owner = CowStrOTag;
} 
```

Да, пользователь AnyMap будет должен зарегистрировать свои типы вручную (например, с помощью процедурного макроса), но он точно не сможет зарегистрировать неправильно и создать коллизию в typeid.

Вот теперь можно насладиться плодами наших трудов: `AnyMap` с не `'static` типами!

```rust
fn main() {

    let s = String::from("hello world");
    
    let mut m = AnyMap::default();
    let cs = Cow::from(&s);
    m.put(cs);
    {   
        // this won't pass borrow check
        // let s = String::from("hello again");
        // let cs = Cow::from(&s);
        // m.put(cs);
    }
    let cs_in_map = m.get::<Cow<str>>();
    println!("{cs_in_map:?}");
}
```

--------------


Применение трюка со сравнением типов `Owner` вместо типов `Element` куда более интересно выглядит если взглянуть на реализации экстракторов в ECS-фреймворках.

В них пользователям позволяют делать что-то такое:

```Rust
app.add_system(
    |res: Res<&MyAsset>, state: State<&mut MyState>| {  /* неявные лайфтайм параметры произвольны -- вы не можете никуда сохранить полученные ссылки */   }
)
```
И во время исполнения, когда будет нужно вызвать переданную функцию, `app` найдет в большой глобальной `AnyMap` пользовательские типы `MyAsset` и `MyState`, и их объкты в качестве аргументов.

Очевидно, что внутри оно каким-то образом делает те самые `AnyMap::get::<MyAsset>()` и `AnyMap::get_mut::<MyState>()` -- это просто и понятно. Но как оно достает сами типы? Ведь вместо них в аргументах ссылки, от которых еще нужно избавться -- мы ж знаем, что у ссылочных типов с лайфтаймами просто так `TypeId` не достанешь.

Делается это тем же самым способом, только сложнее

```rust


trait ArgsMarker {
    type RefErasedArgs: RefErasedArgs;
}

trait RefErasedArgs: 'static {
    type Args<'a>: ArgsMarker<RefErasedArgs = Self>;
}

// lifetime параметр для требования равенства в обратную сторону
// тут, во-первых, не обязателен, поскольку ECS обычно заполняется с помощью Extracted типов напрямую,
// а экстракторы в момент вызова получаются через <Extractor::Extracted as ExtraсtedType>::Extractor<'a>
// так что если кто-то злобный сделает `<Extractor::Extracted as ExtraсtedType>::Extractor<'a>  != Extractor`
// у него будет ошибка компиляции.
// А во-вторых, он потом при реализации IntoSystem и ArgsMarker заставит вас рыдать ошибкой
// the lifetime parameter is not constrained by the impl trait, self type, or predicates
//
trait Extractor {
    // type Mutability: 'static; для простоты ограничимся лишь shared ссылками
    type Extracted: ExtractedType;

    // unsafe потому что lifetime, который мы отбросили, прикручивать придется извне.
    // особенно для поддержки &mut ссылок
    unsafe fn from_world(world: &AnyMap) -> Self;
}

trait ExtractedType: 'static {
    type Extractor<'a>: Extractor<Extracted = Self>;
}


struct State<Ref>(Ref);
struct StateExtracted<S>(S);

impl<S: 'static> Extractor for State<&S> {
    type Extracted = StateExtracted<S>;

    unsafe fn from_world(world: &AnyMap) -> Self {
        // для простоты: отсутствие запрошенного типа в мире 
        // -- непоправимая ошибка.
        let s = world.get::<S>().expect("must be in world");
        let s = s as *const S; // detach lifetime
        unsafe { State(&*s) } 
    }
}

impl<S:'static> ExtractedType for StateExtracted<S> {
    type Extractor<'a> = State<&'a S>;
}


impl<A1, A2> ArgsMarker for (A1, A2)
where A1: Extractor, A2: Extractor {
    type RefErasedArgs = ( A1::Extracted, A2::Extracted );
}

impl<A1: ExtractedType, A2: ExtractedType> RefErasedArgs for (A1, A2) {
    type Args<'a> = ( A1::Extractor<'a>, A2::Extractor<'a> );
}

// Вот это ^ страдание с кортежами повторяется для большего числа аргументов с помощью макросов


// типаж со стертыми типами аргументов. Конкретные реализации знают, что именно им надо запросить
// из AnyMap.
// Также отмечу, что от System часто требуется быть 'static. Именно поэтому 
// и приходится возиться со всеми этими RefErasedArgs вместо ArgsMarker
trait System {
    fn run(&mut self, world: &AnyMap);
}

// Типаж для вычленения аргументов
trait IntoSystem<Args: ArgsMarker> {
    type System: System;
    fn into_system(self) -> Self::System;
}

// Оторванные аргументы нужно куда-то положить. В Rust же generic параметр не может
// быть просто так
struct FuncSystem<F, Args: RefErasedArgs>(F, PhantomData<Args>);

impl<F, A1: ExtractedType, A2: ExtractedType> System for FuncSystem<F, (A1, A2)> 
where F: for<'a> FnMut(A1::Extractor<'a>, A2::Extractor<'a>) {
    fn run(&mut self, world: &AnyMap) {
        // Мы гарантируем здесь, что нацепленный lifetime не 
        // переживет AnyMap.
        // И F, из-за требованию for<'a>, никуда полученные ссылки 
        // сохранить не может. 
        let a1 = unsafe { A1::Extractor::<'_>::from_world(world) };
        let a2 = unsafe { A2::Extractor::<'_>::from_world(world) };
        (self.0)(a1, a2);
    } 
}

impl<F, A1: Extractor, A2: Extractor> IntoSystem<(A1, A2)> for F
where F: FnMut(A1, A2),
      // Но не только для этих конкретных A1 и A2 с фиксированными лайфтаймами!
      // Функцию должно быть можно вызвать для любых!
      // Здесь как раз еще и прячется требование,
      // что существует 'a: <A1::Extracted as ExtractedType>::Extractor<'a> == A1
      F: for<'a> FnMut(<A1::Extracted as ExtractedType>::Extractor<'a>,  
                       <A2::Extracted as ExtractedType>::Extractor<'a>)
{
    type System = FuncSystem<F, (A1::Extracted, A2::Extracted)>;

    fn into_system(self) -> Self::System {
        FuncSystem(self, PhantomData)
    }
}

// Аналогично макросом генерятся варианты для большего числа аргументов

```

После чего вся магия сводится к:

```rust
struct App(AnyMap);

impl App {
    fn add_system<Args: ArgsMarker, F: IntoSystem<Args>>(&self, f: F) {
        let mut sys = f.into_system();
        sys.run(&self.0);
    }
}

```

И мы [поехали](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&gist=45b1362842ad79eab1131c3ce01e2f7a)

```rust

fn main() {
    let mut m = AnyMap::default();
    
    m.put("Hello world".to_string());
    m.put(42_u32);
    
    let app = App(m);
    app.add_system(|s: State<&String>, i: State<&u32>| {
        println!("{} {}", s.0, i.0);
    })
    
}

```

--------------
[^1]: ECS -- entity-component-system.

[^2]: RTTI -- runtime-type-information