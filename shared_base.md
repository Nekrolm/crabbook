# (Не)наследование имплементаций

В Rust, как известно, есть наследование интерфейсов, но нет наследования реализаций. Это вполне обоснованное дизайн решение для системного языка, поскольку наследование реализаций, на примере C++, тащит за собой прорву проблем, с которой не хочется связываться:
- Проблемы ромбовидного наследования
- Проблемы с переинициализацией vptr в конструкторах и деструкторах (виртуальные методы, вызванные в деструкторах/конструкторах работают как не виртуальные)
- Гонки за инициализацией vptr: нельзя при конструировании базового класса спавнить новый поток и отправлять в него ссылку на конструируемый объект
- Хаос и ошибки в переопределении поведения: например, по контракту `Derived::method()` всегда должен в конце/начале вызвать `Base::method()`, но кто-то забыл об этом на третьем или четвертом уровне наследников

Тем не менее у наследования реализаций есть и преимущества. И не только лишь для переиспользования кода.
Здесь и сейчас я продемонстрирую вам занимательный и при этом практически полезный пример. И самое главное -- что с ним можно сделать на Rust.

Допустим мы реализуем систему диспетчеризации сообщений из многих каналов. У всех этих каналов есть общий интерфейс, но реализации могут быть самые разные за исключением одной общей детали, обоснованной по соображением производительности

```C++
class alignas(std::hardware_destructive_interference_size) Source {    
protected:
    // у всех этих каналов первым полем будет лежать статус-флаг для быстрой проверки,
    // нужно ли нам читать из этого канала
    // этот статус-флаг обновляется конкурентно внутри конкретной реализации
    std::atomic_bool ready;
public:

    bool is_ready() const {
        // relaxed read
        // для простоты ordering параметр опущен
        return ready;
    }

    virtual ~Source() = default;

    // читаем согласно конкретной реализации -- долгая операция даже если в канале пусто
    // может сбросить is_ready
    // возвращает true, если что-то было прочитано
    virtual bool read(std::function_ref<void(std::span<uint8_t>)> receiver);
};
```

Пользоваться этим добром будет один читающий поток следующим образом

```C++

Arena channels_arena; // хранит в непрерывных буферах наборы конкретных реализации Source
std::vector<std::reference_wrapper<Source>> sources; // все элементы ссылаются на данные в арене
// любители сырых указателей могут использовать их вместо reference_wrapper

// ...
// поток повторяет цикл опроса, чередуя его с другими операциями
void process(
    std::span<std::reference_wrapper<Source>> sources,
    std::function_ref<void(std::span<uint8_t>)> recv
) {
    for (auto source: sources) {
        auto& src = source.get();
        if (src.is_ready()) {
            src.read(recv);
        }
    }
}
```

На x86 gcc 16 c -O3 и -std=c++26 [генерирует](https://godbolt.org/z/rc5hG4qa4) для этого цикла восхитительно прекрасное и очень компактное тело

```
.L4:
        mov     rdi, QWORD PTR [rbx]
        movzx   eax, BYTE PTR [rdi+8]  ; читаем атомик
        test    al, al
        je      .L3
        ; вызывает read()
        mov     rax, QWORD PTR [rdi]   
        mov     rsi, rbp
        mov     rdx, r12
        call    [QWORD PTR [rax+16]]   
.L3:
        add     rbx, 8
        cmp     r13, rbx
        jne     .L4
```
Обратите внимание! Чтение атомарного флага не требует никакого вызова функции. Мы просто напрямую берем и читаем. Красота! Это blazing fast[^1] и это имеет значение. Существенную часть времени каналы не готовы к чтению -- в них нет новых сообщений [^2]. И чем быстрее мы сможем проскочить холостые итерации, тем быстрее мы перейдем к выполнению чего-то более полезного в этом потоке.
Когда каналов несколько десятков или сотен, совокупная экономия циклов на этом прямом чтении -- 5-10 наносекунд с каждого канала быстро складываются в пару микросекунд, что для ultra low latency приложений вполне заметно.


Если же мы теперь попробуем переложим этот код напрямую на Rust, мы обнаружим проблему -- вот так просто взять и выразить то же самое с тем же результатом не выходит. Тут нет наследования реализаций, только интерфейсов. И красивый `dyn Source` с методом `is_ready()` в нем уже никуда от динамической диспетчеризации [не убежит](https://godbolt.org/z/PeKsPM13T).


```Rust

pub trait Source {
    fn is_ready(&self) -> bool;
    fn read(&mut self, recv: &mut dyn FnMut(&[u8])); 
}


pub fn process(
    sources: &mut [&mut dyn Source],
    recv: &mut dyn FnMut(&[u8]),
) {
    for src in sources {
        if src.is_ready() {
            src.read(recv)
        }
    }
}

```

```
.LBB0_4:
        add     r12, 16
        cmp     r12, r15
        je      .LBB0_5
.LBB0_2:
        mov     r13, qword ptr [r12]
        mov     rbp, qword ptr [r12 + 8]
        mov     rdi, r13
        call    qword ptr [rbp + 24] ; вызов is_ready через vtable
        test    al, al
        je      .LBB0_4
        mov     rdi, r13
        mov     rsi, r14
        mov     rdx, rbx
        call    qword ptr [rbp + 32]
        jmp     .LBB0_4
.LBB0_5:
```

Что же делать? Неужели C++ вот тут в этом специфическом случае так легко и просто разбивает Rust?!

Решение есть! 

Для подобной комбинации статической базы и динамической имплемантаци, сохраняя layout структуры, можно обратиться к пользовательским dynamically sized types (DST) -- штуке редко встречающейся в обычных проектах.

### Шаг первый

Разделяем интерфейс на статическую и динамическую части

```Rust
// динамическая часть
pub trait Source {
    // Тут появляется отличие: если мы следуем тому же паттерну 
    // и концепции: на поле ready будет ссылаться кто-то изнутри
    // конкретной имплементации. Значит, наша структура
    // вообще-то самореферентная и должна быть неперемещаемой
    // в С++ неперемещаемость автоматически наследовалась от наличия atomic_bool.
    //
    // Так что Pin<&mut Self> -- необходимость для безопасного интерфейса
    fn read(self: Pin<&mut Self>, recv: &mut dyn FnMut(&[u8])); 
}

// статическая часть
// в стандартной библиотеке stable Rust нет такой прекрасной константы 
// как в C++
#[repr(C, align(64))] 
pub struct SourceHandle<S: Source + ?Sized> {
    ready: AtomicBool,
    // Маркер для запрета перемещения (сделать !Unpin)
    _pinned: PhantomPinned,
    // динамическую часть оставляем в хвосте
    source: S,
}

impl<S: Source + ?Sized> SourceHandle<S> {
    // is_ready, как в С++ -- не виртуальный
    pub fn is_ready(&self) -> bool {
        self.ready.load(Ordering::Relaxed)
    }
}

```
### Шаг второй

Приколачиваем динамичесную часть к статической

```Rust
impl <S: Source + ?Sized> Source for SourceHandle<S> {
    fn read(self: Pin<&mut Self>, recv: &mut dyn FnMut(&[u8])) {
        self.source().read(recv)
    }
}


// Вспомогательные внутренние геттеры
impl <S: Source + ?Sized> SourceHandle<S> {
    fn source(self: Pin<&mut Self>) -> Pin<&mut S> {
        unsafe { self.map_unchecked_mut(|this| &mut this.source) }
    }

    fn ready(&self) -> NonNull<AtomicBool> {
        NonNull::from_ref(&self.ready)
    }
}

```

### Шаг третий

Изменяем функцию обработки

```Rust

// Делаем наш восхитительный кастомый DST
type DynSource = SourceHandle<dyn Source>; 

#[inline(never)]
pub fn process(
    sources: &mut [Pin<&mut DynSource>],
    recv: &mut dyn FnMut(&[u8]),
) {
    for src in sources {
        if src.is_ready() {
            src.as_mut().read(recv)
        }
    }
}

```

И [получаем](https://godbolt.org/z/W473YWWsx) то чего хотели

```
.LBB0_4:
        ; Стоит заметить что в Rust &mut DST это fat pointer
        ; так что элементы по 16 байт на x64
        add     r13, 16
        cmp     r15, r13
        je      .LBB0_5
.LBB0_2:
        mov     rdi, qword ptr [r12 + r13]
        ; Ура! Атомик проверяется напрямую!
        movzx   eax, byte ptr [rdi]
        test    al, al
        je      .LBB0_4
        mov     rax, qword ptr [r12 + r13 + 8]
        add     rdi, qword ptr [rax + 16]
        mov     rsi, r14
        mov     rdx, rbx
        call    qword ptr [rax + 24]
        jmp     .LBB0_4
.LBB0_5:
```

Победа!

--------------------------

Остается лишь один вопрос: а как же создать конкретный экземпляр `DynSource`?

Не очень сложно. 

```Rust

struct SourceExample {
    internal: [u8; 1024],
}

impl Source for SourceExample {
    fn read(self: Pin<&mut Self>, recv: &mut dyn FnMut(&[u8])) {}
}

impl SourceExample {

    fn new() -> Pin<Box<SourceHandle<dyn Source>>> {
        let handle = SourceHandle {
            ready: AtomicBool::new(false),
            _pinned: PhantomPinned,
            source: Self { internal: [0; 1024] },
        };
        // Вместо Box::pin можно использовать 
        // какой-нибудь bumpalo::Box::pin_in
        // Важно запинить, ведь структура самореферентная
        let mut pinned = Box::pin(handle);
        let ready = pinned.ready();
        unsafe { pinned.as_mut().source().init(ready); }

        // Это одно из немногих неявных автоматических преобразований,
        // которые делает Rust:
        //
        // Unsize coercion!
        //
        // SourceHandle<Self> приводится к SourceHandle<dyn Source>
        // так как 
        //  - поле типа Self в ней последнее
        //  - Self : Source
        pinned
    }

    // немножно unsafe исключительно для самореферетности и inplace инициализации
    // ведь в Rust нет конструкторов и placement new
    unsafe fn init(self: Pin<&mut Self>, is_ready: NonNull<AtomicBool>) {}
}

```


----------


[^1]: Можно пойти еще дальше по пути секты свидетелей подхода Struct Of Arrays (SAO): и хранить отдельно `vector<atomic_bool>` и `vector<Source*>`, где каждый элемент ссылкается на свой атомик. Тогда теоретически проверка на холостые циклы станет еще более blazing fast, ведь  Но я сомневаюсь, что это того стоит для этого конкретного случая. Придется превозмочь ограничения, что `atomic_bool` напрямую закидывать в вектор не удастся (он некопируем и неперемещаем), а также подумать о динамическом добавлении новых источников с учетом того, что вектор нельзя реаллоцировать для стабильности ссылок на атомики.

[^2]: Здесь можно подискутировать на тему, а не надо ли нам сюда push/notification вместо активного опроса. Не надо. У системы есть ряд требований, в том числе детерминизм и короткие хвосты распределения latency, которые отбивают варианты c механизмами оповещений от OS.