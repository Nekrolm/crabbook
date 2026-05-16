# Бахнет ли impl Trait в аргументах функции? 

В [предыдущих сериях](impl_trait_references.md) я рассказывал, что значительно упростить generic-интерфейсы, избавить их от аннотаций времени жизни часто можно реализовав типаж для ссылок и дальше используя, `struct S<Iface: MyTrait> {  field: Iface }` вместо `struct <'a, Iface: MyTrait> { field : &'a mut Iface }`. 
И аналогично в методах можно будет принимать `impl Iface` вместо `&mut impl Iface`;

Однако, с удобством и упрощением вполне неплохо можно выстрелить себе в ногу. Не сильно, но неприятно

Есть, например, такой замечательный типаж `std::io::Write`. 
Он для удобства реализован и для мутабельный ссылок: `impl <W: Write + ?Sized> Write for &mut W { ... }`

И допустим мы пишем некоторую библиотеку бинарной сериализации (или маршалинга), в которой будет удобный типаж
```rust
pub trait Serialize {
    fn serialize(&self, out: impl io::Write) -> io::Result<()>;
}
```

Реализовали вы его для примитивов 
```Rust
// например 
impl Serialize for i32 {
    fn serialize(&self, mut out: impl io::Write) -> io::Result<()> {
        out.write_all(&self.to_le_bytes())
    }
}
// и так далее для остальных числовых типов
// ...


// И для строк, например
impl Serialize for str {
    fn serialize(&self, mut out: impl io::Write) -> io::Result<()> {
        let n = self.len();
        n.serialize(&mut out)?; // по значению передавать нельзя -- нам еще самими байты записывать! 
        out.write_all(self.as_bytes())
    }
}
```

Тут уже можно начать что-то подозревать, но давайте продолжим: наверняка, для удобства, нам прогодится generic реализация для контейнеров

```rust
impl <T: Serialize> Serialize for Vec<T> {
    fn serialize(&self, mut out: impl io::Write) -> io::Result<()> {
        let n = self.len();
        n.serialize(&mut out)?;
        for item in self.iter() {
            item.serialize(&mut out)?;
        }
        Ok(())
    }
}
// аналогично для всяких HashMap
```

Вроде ничего необычного. Можно перейти к самому главному!
Какая же библиотека сериализации без JSON-структур! В наше время без JSON никуда.

```Rust
enum JsonValue {
    Integer(i64),
    Array(Vec<JsonValue>),
    // остальные вариаенты ...
}

const INT_TAG: u8 = 0;
const ARRAY_TAG: u8 = 1;
// .... 

impl Serialize for JsonValue {
    fn serialize(&self, mut out: impl io::Write) -> io::Result<()> {
        match self {
            JsonValue::Integer(val) => {
                INT_TAG.serialize(&mut out)?;
                val.serialize(out)
            },
            JsonValue::Array(arr) => {
                ARRAY_TAG.serialize(&mut out)?;
                arr.serialize(out)
            }
        }
    }
}
```

Собираем это все [вместе](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&gist=2b687f33750998ea11c0a50e2127f354) -- вроде компилируется.
И все? И ничего?

Погодите. Код для релизации типажей компилируется на самом деле только тогда, когда используется. А так он всего лишь прошел проверку типов.
Давайте действительно скомпилируем!

Добавим использование
```rust
pub fn try_json(json: JsonValue, out: &mut dyn io::Write) {
    let _ = json.serialize(out);
}
```

И... 

```rust

   Compiling playground v0.0.1 (/playground)
error[E0275]: overflow evaluating the requirement `dyn std::io::Write: std::io::Write`
  |
  = help: consider increasing the recursion limit by adding a `#![recursion_limit = "256"]` attribute to your crate (`playground`)
  = note: required for `&mut dyn std::io::Write` to implement `std::io::Write`
  = note: 128 redundant requirements hidden
  = note: required for `&mut &mut &mut &mut &mut &mut &mut &mut &mut &mut &mut &mut &mut ...` to implement `std::io::Write`
  = note: the full name for the type has been written to '/playground/target/debug/deps/playground-bc4d385e38340b6d.long-type-3953689482779153601.txt'
  = note: consider using `--verbose` to print the full type name to the console


```

Красиво! 

```rust
note: required for `&mut &mut &mut &mut &mut &mut &mut &mut &mut &mut &mut &mut &mut ...` to implement `std::io::Write`
```

Мы получили бесконечный рекурсивный тип: каждый раз вызывая `arr.serialize(out)`, благодаря реализации для вектора
```rust
        for item in self.iter() {
            item.serialize(&mut out)?;
        }
```
мы навешиваем все больше и больше `&mut` ссылок.

Эта проблема специфична не только для рекурсивных конструкций как `JsonValue`, но в целом:

Если у нас есть две функции
```rust
// Iface реализован также для &mut impl Iface

fn bar(mut x: impl Iface) { ... }

fn foo(mut x: impl Iface) { 
    // ...
    bar(&mut x);
    // ...
}
```

И обе они вызываются всего один раз:

```rust

fn main() {
    let mut x = IfaceImpl;
    bar(&mut x);
    foo(&mut x);
}

```

То мы в итоге получим *три* мономорфизированные функции вместо двух:

```rust
fn bar_ref_mut_IfaceImpl(x: &mut IfaceImp) {...}
fn bar_ref_mut_ref_mut_IfaceImpl(x: &mut &mut IfaceImp) {...}
fn foo_ref_mut_IfaceImpl(x: &mut IfaceImpl) {...}
```

Ну и бесконечное число в паталогическом случае рекурсивных структур.

Так что иногда удобством стоит пожертвовать, чтоб не получить взрыв времени компиляции [^1]

```Rust

impl Serialize {
    fn serialize(&self, out: &mut impl io::Write) -> io::Result<()> 
}


impl <T: Serialize> Serialize for Vec<T> {
    fn serialize(&self, out: &mut impl io::Write) -> io::Result<()> {
        let n = self.len();
        n.serialize(out)?;
        for item in self.iter() {
            // больше не навешиваем дополнительных ссылок -- остаемся в с тем же типом
            item.serialize(out)?;
        }
        Ok(())
    }
}

```

Могу, кстати, порадовать любителей C++: в нем, в шаблонах, подобная проблема маловероятна -- в C++ нельзя взять ссылку на ссылку, да и по значению там сложные объекты в шаблоны редко передают:

```C++
template<class W>
concept Writer = true; // todo! :) 

struct JsonValue: std::variant<int64_t, std::vector<JsonValue>> {
    void serialize(this const JsonValue& self, Writer auto&& w) { /* ... */}
};
```
------------

[^1]: И не только. От слишком большого уровня индирекций `&mut &mut &mut Object`, образовавшихся таким нехитрым способом, может пострадать и время исполнения


