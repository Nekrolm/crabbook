# Вы не хотите реализовывать Drop

В Rust, как и в C++, для так называемой resource safety -- защита от утечек ресурсов[^1] -- используется механизм автоматически вызываемых деструкторов

```C++

struct OperError{};

class FileDescriptor {
public:

    static std::expected<FileDescriptor, OpenError> open(const char* c_path) noexcept {
        FileDescriptor fd{c_path};
        if (!fd.is_valid()) {
            return std::unexpected(OpenError{});
        }
        return fd;
    }

    
    ~FileDescriptor() {
        if (!is_valid()) {
            return;
        }
        if (close(fd) != 0) {
            // хотя обычно ошибку тут игнорируют
            std::terminate(); 
        }
    }
    // Раз определили деструктор, то нужно определить и оставшиеся 4 специальных метода
    FileDescriptor(FileDescriptor&& other) noexcept {
        std::swap(fd, other.fd);
    }
    FileDescriptor(const FileDescriptor&) = delete;
    FileDescriptor& operator = (FileDescriptor&& other) noexcept {
        FileDescriptor tmp { std::move(other) };
        std::swap(fd, tmp.fd);
        return *this;
    }
    FileDescriptor& operator = (const FileDescriptor&) = delete;
private:

    bool is_valid() const {
        return fd >= 0;
    }

    explicit FileDescriptor(const char* c_path) : fd { open(c_path, FLAGS ... ) } {}


    int fd = -1;
};


void foo(const char* c_path) {
    auto fd = FileDescriptor::open(c_path).value();
    ...
    // деструктор fd автоматически будет вызван при выходе из области видимости
}

```

Почти то же самое в Rust

```rust

pub struct FileDescriptor(i32); 

pub struct OpenError; // детали сообщения опущены для простоты

impl FileDescriptor {
    pub fn open(path: &CStr) -> Result<Self, OpenError> {
        let fd = unsafe { libc::open(path.as_ptr(), ....) };
        if fd < 0 {
            return Err(OpenError);
        }
        Ok(Self(fd))
    }
}

impl Drop for FileDescriptor {
    fn drop(&mut self) {
        if unsafe { libc::close(self.fd) } != 0 {
            std::process::abort();
        }
    }
}


fn foo(c_path: &CStr) {
    let fd = FileDescriptor::open(c_path).unwrap();
    ...
    // std::mem::drop(fd) автоматически будет вызван при выходе из области видимости
}
```

Первое очевидное отличие: семантика перемещения вшита в язык изначально, а не пришита сбоку, так что никаких специальных методов для нее описывать не нужно.

Второе, менее очевидное: у `FileDescriptor` в Rust версии внутри нет невалидного состояния и проверки на него.

И также как в C++, для составных структур компилятор автоматически вызывает деструкторы полей, независимо от наличия пользовательского деструктора


```C++
// Деструктор MyData вызовет деструкторы полей, в обратном порядке.
// Сначала уничтожит fd, затем name
struct MyData {
    std::string name;
    FileDescriptor fd;
};
```

Но только есть отличие

```Rust
// В Rust деструкторы полей вызываются в прямом порядке!
// Cначала name, затем fd. Эта особенность справедлива только для полей структур. 
// Переменные в скоупах функций, как и в C++, как и должно быть, всегда уничтожаются в порядке обратном из созданию.
struct MyData {
    name: String,
    fd: FileDescriptor,
}
```

На этом отличия не заканчиваются.

В Rust есть возможность частичной разборки структур -- partial move.

```Rust
fn take_string(s: String) {}

fn foo(data: MyData) {
    take_string(data.name);
    ...

    // Только для поля data.fd будет вызван drop! 
    // Строка name уже была поглощена и уничтожена внутри функции `take_string`
}
```

```C++
void take_string(std::string s) {}

void foo(data: MyData) {
    take_string(std::move(data.name));
    ...
    // Дестурктор для всего объекта data будет вызван.
}
```

И если в C++ мы добавим пользовательский деструктор
```C++
struct MyData {
   
   ~MyData() { std::println("hello dctor");  }
   
   ...
}
```
То для функции foo ничего не изменится. Она всё также будет компилироваться. 

А вот в случае Rust
```rust 
impl Drop for MyData {
    fn drop(&mut self) {
        println!("hello drop");
    }
}
```
Добавление пользовательского деструктора отключает возможность раздербанивать структуру по частям!
`foo` [перестанет компилироваться](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&gist=b1b7c532e3ec28910e315b114143fd82).


```
error[E0509]: cannot move out of type `MyData`, which implements the `Drop` trait
  --> src/lib.rs:34:17
   |
34 |     take_string(data.name);
   |                 ^^^^^^^^^
   |                 |
   |                 cannot move out of here
   |                 move occurs because `data.name` has type `String`, which does not implement the `Copy` trait
   |
help: consider cloning the value if the performance cost is acceptable
   |
34 |     take_string(data.name.clone())
```

С одной стороны это хорошо: пользовательский drop может использовать все поля структуры, так что растаскивать нельзя.
А с другой -- если все-таки разбор на составные части необходим, придется добавлять и поддерживать пустое / невалидное состояние для нетривиальных полей. Как в C++. И в этот момент C++ники должны возгордиться и сказать, что раз так, то Rust не нужен.

-------------
Рассмотрим следующий занятный пример:

Пусть нам нужно смоделировать объект, который необходимо использовать один раз. А иначе это непоправимая ошибка. Конечно, идеально было бы отловить эту ошибку во время компиляции, но ни в Rust, ни в C++ линейных типов нет, использование опционально, так что придется довольствоваться ошибкой времени исполнения.

Что ж, известный факт, деструкторы хороши не только для того чтоб ресурсы очищать, но ими еще и можно описывать произвольную логику, которая должна быть гарантированно выполнена при выходе из области выхода. А это как раз подходит для реализации подобной проверки: завести индикатор использованности, и в деструкторе объекта проверить, а использовали ли его или нет.

```C++
struct ErrorHandle {
    void fail(std::string_view message) {};
};

class Gadget {
public:

    Gadget(std::string data, ErrorHandle err): data{ std::move(data) }, not_used { err } {}
    ~Gadget() {
        if (not_used) {
            not_used->fail("Destroyed before usage");
        }
    }

    // rvalue квалификатор чтоб подчеркнуть, что использование требует передачи владения
    std::string use_it() && {
        not_used = nullopt;
        return std::move(this->data); 
    }

private:
    std::string data;

    std::optional<ErrorHandle> not_used;
}
```

```Rust

struct ErrorHandle;

impl ErrorHandle {
    fn fail(&self, message: &str) {}
}

struct Gadget {
    data: String,
    not_used: Option<ErrHandle>
}

impl Drop for Gadget {
    fn drop(&mut self) {
        if let Some(handle) = self.not_used {
            handle.fail("Destroyed before usage");
        } 
    }
}

impl Gadget {
    pub fn new(data: String, err: ErrorHandle) -> Self {
        Self {
            data,
            not_used: Some(err),
        }
    }
    fn use_it(mut self) -> String {
        self.handle = None;
        // self.data -- не скомпилируется из-за Drop
        std::mem::take(&mut self.data)
    }
}
```


Ну, не так страшно, да?

А что если в качестве поля `data` взять что-то, у чего нет пустого состояния? 
И еще, перед `use_it`, надо разрешить вызывать какой-нибудь метод над `data`

```Rust
pub trait Data {
    fn inspect(&self);
}

struct Gadget {
    // Теперь нам придется использовать Option
    data: Option<Box<dyn Data>>,
    not_used: Option<ErrHandle>
}

// Drop без изменений

impl Gadget {
    fn new(data: Box<dyn Data>, err: ErrorHandle) -> Self {
        Self {
            data: Some(data),
            not_used: Some(err),
        }
    }

    fn inspect(&self) {
        // можно безопасно вызывть unwrap: тут всегда Some
        self.data.as_ref().unwrap().inspect(); 
    }

    fn use_it(mut self) -> Box<dyn Data> {
        self.not_used = None;
        self.data.take().unwrap();
    }
}
```

Сразу стало противно от `unwrap`. 
А в C++ у всех перемещаемых типов есть пустое состояние и unsafe доступ лаконичнее проверяемого, так что оно может выглядеть менее отвратительно.

Можно, конечно, оставить как есть, но зачем, если можно сделать лучше, красивее и безопаснее?

-------------

Вот в C++ есть "правило нуля" -- если не определять самому ни один из спец методов (деструктор, конструкторы и операторы копирования и перемещения), то будет тебе счастье и вечная благодать. И компилятор сам все сделает. И вообще все современные гайдлайны говорят, что не надо самому деструкторы писать.

Так вот в Rust история та же. Если не писать `Drop` для нашей структуры, то будет нам счастье и вечная благодать. [^2]

Ту же самую проверку на использованность можно выполнить [по-другому](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&gist=5ca715b411aaea4fa33a94079b4b9bd9), привязав ее только к конкретному полю:

```rust

struct NotUsedGuard(Option<ErrorHandle>);

impl NotUsedGuard {
    fn new(err: ErrorHandle) -> Self {
        Self(Some(err))
    }
    fn disarm(mut self) {
        self.0 = None;
    }
}

impl Drop for NotUsedGuard {
    fn drop(&mut self) {
        let Self(Some(err)) = self else { return; };
        err.fail("Destroyed before usage");
    }
}

struct Gadget {
    data: Box<dyn Data>, // нам больше не нужен Option здесь! Ура!
    not_used: NotUsedGuard,
}

impl Gadget {
    fn new(data: Box<dyn Data>, err: ErrorHandle) -> Self {
        Self {
            data,
            not_used: NotUsedGuard::new(err)
        }
    }
    
    fn inspect(&self) {
        // Нет Option -> нет unwrap. Ура!
        self.data.inspect(); 
    }

    fn use_it(self) -> Box<dyn Data> {
        self.not_used.disarm();
        // У Gadget нет пользовательского Drop -> partial move работает! Ура! 
        self.data
    }

}
```

Стало лучше. Почти как в C++ -- всего с одним `Option` -- только в Rust ненужные деструкторы на пустых состояниях не вызываются.
Но можно сделать еще лучше, чтоб горделивые C++ники совсем расстроились.

`sizeof(NotUsedGuard) >= sizeof(ErrorHandle)`. 
Равенство достигается в случаях, когда `ErrorHandle` имеет нишу под какое-нибудь невалидное представление. Например `nullptr`, если где-то внутри у него будет `Box<T>`.

С помощью щепотки `unsafe {}` мы можем добиться, что `NotUsedGuard` будет совершенно идентичен `ErrorHandle`. Ведь на самом деле нам не нужен никакой `Option`: переходы состояний `not_used` -> `used` выражены передачей владения: если последний владелец это функция `NotUsedGuard::disarm` -- то мы в `used` состоянии. Иначе -- `not_used`.

```rust
use std::mem::ManuallyDrop;

#[repr(transparent)]
struct NotUsedGuard(ManuallyDrop<ErrorHandle>);

impl NotUsedGuard {
    fn new(err: ErrorHandle) -> Self {
        Self(ManuallyDrop::new(err))
    }
    
    fn disarm(self) {
        // Хоп, и drop больше не будет вызван
        let mut this = ManuallyDrop::new(self);
        // Safety: это последний владелец.
        // Больше доступа не будет
        // Уничтожаем ErrorHandle
        let _ = unsafe { ManuallyDrop::take(&mut this.0) };
    }

}

impl Drop for NotUsedGuard {
    fn drop(&mut self) {
        // Safety: это drop. Мы -- последний владелец
        unsafe { ManuallyDrop::take(&mut self.0) }
            .fail("Destroyed before usage");
    }
}
```

Все, ничего сложного, и никто не пострадал.


Примерно так, кстати, и реализован вспомогательная структура [`DropGuard`](https://doc.rust-lang.org/beta/std/mem/struct.DropGuard.html) из стандартной библиотеки. Вот только она unstable. Но можно взять аналогичную из крейта [`scopegurad`](https://docs.rs/scopeguard/latest/scopeguard/).

Правда, вот так просто взять и приспособить для той же самой цели и с тем же самым эффектом -- не выйдет. Но это уже другая история.


--------------

[^1]: Памяти, файловых дескрипторов, соединений из пулов вашей любимой сетевой бибоиотеки
[^2]: Со структурами, реализующими `Drop`, есть еще разные странные приколы и баго-ограничения текущей реализации borrow checkerа. Например, https://github.com/rust-lang/rust/issues/154695#issuecomment-4187884639