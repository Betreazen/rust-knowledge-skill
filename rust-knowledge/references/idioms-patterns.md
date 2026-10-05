# Идиомы и паттерны Rust

Справочник по идиоматичному Rust для любых проектов (библиотеки, CLI, бэкенды, embedded, игры): каталог паттернов и антипаттернов с условиями применения и короткими эскизами кода, чеклист для code review, ключевые статьи и сдвиги терминологии на октябрь 2026 года. Общая идея: инварианты живут в типах, а не в проверках, разбросанных по коду.

- [Канонические источники](#канонические-источники)
- [Типы как носители инвариантов](#типы-как-носители-инвариантов)
- [Трейты и диспетчеризация](#трейты-и-диспетчеризация)
- [API: заимствование и конверсии](#api-заимствование-и-конверсии)
- [Владение, мутабельность и графы](#владение-мутабельность-и-графы)
- [Ошибки и паника](#ошибки-и-паника)
- [Антипаттерны](#антипаттерны)
- [Линты и сдвиги терминологии](#линты-и-сдвиги-терминологии)
- [Чеклист code review: идиоматичный Rust](#чеклист-code-review-идиоматичный-rust)
- [Влиятельные статьи](#влиятельные-статьи)

## Канонические источники

**Ключевые выводы**
- Для дизайна публичного API идти по чеклисту API Guidelines: именование `as_`/`to_`/`into_`, конверсии, future-proofing. Основной текст написан до edition 2021, поэтому читать его вместе с Effective Rust (2024) и Microsoft Pragmatic Rust Guidelines (2025).
- Единственный канонический источник с явным разделом антипаттернов — Rust Design Patterns (rust-unofficial). Книга поддерживается: последнее добавление — «custom traits для сложных bounds», декабрь 2025.
- Самый свежий обзор дизайна API — «Practical Rust API Design» (corrode, 2026-10-02).
- Классические эссе (King, Biffle, Hoverbear, Hertleif, floooh) почти все написаны до 2021 года. Идеи живы, синтаксис местами нет (`try!`, `extern crate`, `Box<Trait>` без `dyn`).

| Ресурс | Автор, год | Зачем читать |
|---|---|---|
| [Rust API Guidelines (checklist)](https://rust-lang.github.io/api-guidelines/checklist.html) | rust-lang libs team, ~2017–2019 | 11 категорий чеклиста: C-NEWTYPE, C-BUILDER, C-SEALED, C-CUSTOM-TYPE, C-GOOD-ERR и др. |
| [Rust Design Patterns](https://rust-unofficial.github.io/patterns/) | сообщество, живой | Идиомы, паттерны и антипаттерны (clone ради borrowck, Deref-полиморфизм, `deny(warnings)`) |
| [Effective Rust](https://effective-rust.com/) | David Drysdale, 2024 | 35 пунктов: типы, ошибки, конверсии, newtype, builder, RAII, generics vs `dyn`, Clippy |
| [Rust for Rustaceans](https://nostarch.com/rust-rustaceans) | Jon Gjengset, 2021 | Гл. 3 «Designing Interfaces»: предсказуемость, гибкость, очевидность, ограниченность API |
| [Pragmatic Rust Guidelines](https://microsoft.github.io/rust-guidelines/) | Microsoft, 2025 | Надстройка над API Guidelines: только правила, с которыми согласно большинство опытных разработчиков. Есть [чеклист](https://microsoft.github.io/rust-guidelines/guidelines/checklist/index.html) и сжатая версия для ИИ-ассистентов |
| [Clippy lint index](https://rust-lang.github.io/rust-clippy/master/index.html) | rust-lang | Поиск lint по имени и группе |
| [Learning Material for Idiomatic Rust](https://corrode.dev/blog/idiomatic-rust-resources/) | Matthias Endler, 2024 | Мета-список ресурсов |

## Типы как носители инвариантов

**Ключевые выводы**
- Любое значение с собственным смыслом (ID, единица измерения, проверенная строка) получает newtype. Перепутать аргументы станет невозможно, а инвариант проверяется один раз.
- Проверять ввод на границе системы и возвращать тип-доказательство, а не `bool`. Дальше по коду повторных проверок нет.
- Набор флагов и `Option`, которые должны согласоваться между собой, заменять enum с данными.
- Typestate стоит своей сложности, когда неверный порядок вызовов — реальная ошибка (протоколы, ресурсы, builder с обязательными полями).
- Освобождение ресурса привязывать к `Drop`: оно сработает при `return`, `?` и раскрутке паники.

**Newtype.** Когда применять: примитив несёт доменный смысл; нужен свой набор трейтов для чужого типа (orphan rule); нужно скрыть представление. Лечит «stringly typed» код. Источник: [Effective Rust, Item 6](https://effective-rust.com/newtype.html).
```rust
#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash)]
pub struct UserId(u64);
#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash)]
pub struct OrderId(u64);

fn cancel(order: OrderId, by: UserId) { /* перепутать аргументы нельзя */ }
```

**Parse, don't validate.** Когда применять: на любой границе (CLI, HTTP, файл конфигурации, FFI). Поле приватное, создать значение можно только через конструктор. Источники: [Alexis King](https://lexi-lambda.github.io/blog/2019/11/05/parse-don-t-validate/), [corrode: Practical Rust API Design](https://corrode.dev/blog/practical-rust-api-design/).
```rust
pub struct Email(String);

impl TryFrom<String> for Email {
    type Error = &'static str;
    fn try_from(s: String) -> Result<Self, Self::Error> {
        if s.contains('@') { Ok(Email(s)) } else { Err("в адресе нет @") }
    }
}
fn send(to: &Email) { /* корректность уже доказана типом */ }
```

**Невозможные состояния непредставимы.** Когда применять: у структуры есть поля, допустимые только при определённом значении другого поля. Источник: [corrode: Make Illegal States Unrepresentable](https://corrode.dev/blog/illegal-state/).
```rust
// Было: connected: bool, session: Option<Session>, error: Option<String>
enum Connection {
    Disconnected,
    Connecting { attempt: u32 },
    Connected { session: Session },
    Failed { reason: String },
}
```

**Typestate.** Когда применять: объект проходит фиксированные стадии и часть операций допустима только в одной из них. Переход потребляет `self`, старое состояние использовать нельзя. Источник: [Cliff Biffle](https://cliffle.com/blog/rust-typestate/).
```rust
use std::marker::PhantomData;
pub struct Closed;
pub struct Open;
pub struct Port<S> { id: u8, _s: PhantomData<S> }

impl Port<Closed> {
    pub fn open(self) -> Port<Open> { Port { id: self.id, _s: PhantomData } }
}
impl Port<Open> {
    pub fn write(&mut self, _b: &[u8]) {}
    pub fn close(self) -> Port<Closed> { Port { id: self.id, _s: PhantomData } }
}
```

**Builder.** Когда применять: много необязательных параметров или конструктор с 4+ аргументами. Обязательные поля передавать в `new`, либо проверять их при компиляции через typestate-builder ([greyblake](https://www.greyblake.com/blog/builder-with-typestate-in-rust/)). Если опций мало, хватит структуры-параметра с `Default` и `..Default::default()`. Источник: [Effective Rust, Item 7](https://effective-rust.com/builders.html).
```rust
#[derive(Default)]
pub struct ServerBuilder { port: Option<u16>, workers: Option<usize> }

impl ServerBuilder {
    pub fn port(mut self, p: u16) -> Self { self.port = Some(p); self }
    pub fn workers(mut self, n: usize) -> Self { self.workers = Some(n); self }
    pub fn build(self) -> Server {
        Server { port: self.port.unwrap_or(8080), workers: self.workers.unwrap_or(4) }
    }
}
```

**RAII и guards.** Когда применять: ресурс нужно освободить или состояние восстановить (файл, временный каталог, lock, транзакция, счётчик). Guard (как `MutexGuard`) даёт доступ ровно на время своей жизни. Сбой в `Drop` вернуть нельзя: для важных ошибок нужен явный `close(self) -> Result`. Источники: [Effective Rust, Item 11](https://effective-rust.com/raii.html), [rust-unofficial: RAII Guards](https://rust-unofficial.github.io/patterns/patterns/behavioural/RAII.html).
```rust
pub struct TempDir(std::path::PathBuf);

impl Drop for TempDir {
    fn drop(&mut self) {
        let _ = std::fs::remove_dir_all(&self.0); // сработает при return, `?` и раскрутке паники
    }
}
```

| Ресурс | Автор, год | Зачем читать |
|---|---|---|
| [Pretty State Machine Patterns in Rust](https://hoverbear.org/blog/rust-state-machine-pattern/) | Ana Hobden, 2016 | Автоматы на enum и generic-состояниях. Синтаксис [устарело], идеи актуальны |
| [Compile-Time Invariants in Rust](https://corrode.dev/blog/compile-time-invariants/), [Using Enums to Represent State](https://corrode.dev/blog/enums/) | corrode, 2023 | Короткие практичные варианты тех же идей |

## Трейты и диспетчеризация

**Ключевые выводы**
- Трейт, который не должны реализовывать снаружи, запечатывать: тогда методы можно добавлять без semver-break.
- Методы для чужих типов добавлять через extension trait с суффиксом `Ext`.
- Закрытое множество вариантов — enum. Открытое (плагины, разнородные коллекции) — `dyn Trait`. Горячий путь с типами, известными при компиляции, — generics.
- На границах крейтов в большом workspace generics раздувают время сборки. Там допустимы `dyn` или конкретные типы, внутри крейта — generics.
- Повторяющийся длинный where-клоз упаковывать в свой трейт с blanket impl.

**Sealed traits.** Когда применять: публичный трейт библиотеки, который пользователи вызывают и упоминают в bounds, но не реализуют. Есть три уровня запечатывания и частичное запечатывание отдельных методов. Источник: [Predrag Gruevski](https://predr.ag/blog/definitive-guide-to-sealed-traits-in-rust/).
```rust
mod private { pub trait Sealed {} }

pub trait Shape: private::Sealed {
    fn area(&self) -> f64;
}
pub struct Square(pub f64);
impl private::Sealed for Square {}
impl Shape for Square { fn area(&self) -> f64 { self.0 * self.0 } }
```

**Extension traits.** Когда применять: нужны удобные методы на типе из std или чужого крейта (`FutureExt`, `IteratorExt`). Пользователь подключает их явным `use`. Источник: [RFC 445](https://rust-lang.github.io/rfcs/0445-extension-trait-conventions.html).
```rust
pub trait StrExt {
    fn is_blank(&self) -> bool;
}
impl StrExt for str {
    fn is_blank(&self) -> bool { self.trim().is_empty() }
}
```

**Generics, `dyn` или enum.** Источники: [Effective Rust, Item 12](https://effective-rust.com/generics.html), [corrode: Understanding Dyn Compatibility](https://corrode.dev/blog/dyn-compatibility/), [rust-unofficial: on-stack dynamic dispatch](https://rust-unofficial.github.io/patterns/idioms/on-stack-dyn-dispatch.html).

| Вариант | Когда | Цена |
|---|---|---|
| Generics (`T: Trait`, `impl Trait`) | Типы известны при компиляции, горячий путь | Мономорфизация: код раздувается, сборка дольше |
| `dyn Trait` | Открытое множество типов, разнородные коллекции, стабильная граница крейта | Vtable, обычно аллокация (`Box`), трейт должен быть dyn compatible |
| Enum | Множество вариантов закрыто и известно | Добавление варианта трогает все `match`, зато компилятор находит все места |

```rust
fn draw_all<T: Draw>(items: &[T]) { for i in items { i.draw() } }      // generics
fn draw_any(items: &[Box<dyn Draw>]) { for i in items { i.draw() } }   // dyn
enum Widget { Button(Button), Label(Label) }                            // enum
```

**Свой трейт для сложных bounds.** Когда применять: одинаковый набор bounds повторяется в нескольких сигнатурах. Источник: [rust-unofficial](https://rust-unofficial.github.io/patterns/patterns/structural/trait-for-bounds.html).
```rust
pub trait Key: Eq + std::hash::Hash + Clone + Send + Sync + 'static {}
impl<T: Eq + std::hash::Hash + Clone + Send + Sync + 'static> Key for T {}

pub fn dedup<K: Key>(keys: &[K]) -> std::collections::HashSet<K> { keys.iter().cloned().collect() }
```

| Ресурс | Автор, год | Зачем читать |
|---|---|---|
| [Understanding Dyn Compatibility](https://corrode.dev/blog/dyn-compatibility/) | corrode, 2026 | Правила, которые раньше назывались object safety |
| [API Guidelines: Future proofing](https://rust-lang.github.io/api-guidelines/future-proofing.html) | rust-lang | C-SEALED, приватные поля, `#[non_exhaustive]` |

## API: заимствование и конверсии

**Ключевые выводы**
- Если функция только читает, принимать `&str`, `&[T]`, `&Path`, а не `&String`, `&Vec<T>`, `&PathBuf`. Если сохраняет значение, принимать `String` или `impl Into<String>`, чтобы вызывающий сам решил, клонировать ли.
- Generic-обёртку (`impl AsRef<Path>`) делать тонкой, а тело выносить в негенерик-функцию: мономорфизируется только обёртка.
- Реализовывать `From`, а не `Into` (`Into` получается бесплатно). Для конверсий, которые могут не сработать, — `TryFrom`.
- Именование: `as_` — дёшево, заимствование в заимствование; `to_` — дорого или с аллокацией; `into_` — потребляет `self`.
- `Cow` возвращать, когда в частом случае вход годится без изменений.
- Публичные enum и структуры, которые будут расти, помечать `#[non_exhaustive]`.
- Возвращать `impl Iterator`, принимать `impl IntoIterator`.

**Заимствование в аргументах.** Источники: [rust-unofficial: Use borrowed types for arguments](https://rust-unofficial.github.io/patterns/idioms/coercion-arguments.html), [matklad: Rust's Ugly Syntax](https://matklad.github.io/2023/01/26/rusts-ugly-syntax.html).
```rust
fn count_words(text: &str) -> usize { text.split_whitespace().count() } // только читаем

pub struct User { name: String }
impl User {
    pub fn new(name: impl Into<String>) -> Self { User { name: name.into() } } // сохраняем
}

pub fn load(path: impl AsRef<std::path::Path>) -> std::io::Result<String> {
    fn inner(path: &std::path::Path) -> std::io::Result<String> { std::fs::read_to_string(path) }
    inner(path.as_ref())
}
```

**From / Into / TryFrom / AsRef.** Когда применять: `From` — для очевидной конверсии без потерь; `TryFrom` — если вход может быть невалиден; `AsRef` — для дешёвого получения ссылки; `Borrow` — только когда `Eq`/`Hash`/`Ord` у заимствованной формы совпадают с исходной (ключи `HashMap`). Источники: [API Guidelines](https://rust-lang.github.io/api-guidelines/checklist.html), [Effective Rust, Item 5](https://effective-rust.com/casts.html).
```rust
pub struct Celsius(pub f64);
pub struct Fahrenheit(pub f64);

impl From<Celsius> for Fahrenheit {
    fn from(c: Celsius) -> Self { Fahrenheit(c.0 * 9.0 / 5.0 + 32.0) }
}
// let f: Fahrenheit = Celsius(100.0).into();
```

**`Cow`.** Когда применять: функция обычно возвращает вход как есть и лишь иногда его меняет. Источник: [Pascal Hertleif: The Secret Life of Cows](https://deterministic.space/secret-life-of-cows.html).
```rust
use std::borrow::Cow;

fn expand_tabs(input: &str) -> Cow<'_, str> {
    if input.contains('\t') {
        Cow::Owned(input.replace('\t', "    "))
    } else {
        Cow::Borrowed(input) // частый случай без аллокации
    }
}
```

**`#[non_exhaustive]` и escape-вариант.** Когда применять: публичный enum или структура будут пополняться. Снаружи крейта `match` обязан иметь `_`, а структуру нельзя собрать литералом, поэтому нужен конструктор или `Default`. Источник: [corrode: Practical Rust API Design](https://corrode.dev/blog/practical-rust-api-design/).
```rust
#[non_exhaustive]
pub enum Protocol { Http, Https, Unregistered(u16) }
```

**Итераторы.** Когда применять: почти всегда вместо индексных циклов. `collect()` в `Result<Vec<_>, _>` останавливается на первой ошибке. Источники: [Effective Rust, Item 9](https://effective-rust.com/iterators.html), [corrode: Thinking in Iterators](https://corrode.dev/blog/iterators/).
```rust
fn parse_all(lines: &[&str]) -> Result<Vec<u32>, std::num::ParseIntError> {
    lines.iter().map(|s| s.trim().parse::<u32>()).collect()
}
// В edition 2024 RPIT захватывает все lifetimes, `+ '_` писать не нужно
pub fn evens(v: &[u32]) -> impl Iterator<Item = u32> { v.iter().copied().filter(|x| x % 2 == 0) }
```

| Ресурс | Автор, год | Зачем читать |
|---|---|---|
| [Elegant Library APIs in Rust](https://deterministic.space/elegant-apis-in-rust.html) | Pascal Hertleif, 2016 | `impl Into`, builder, `Default`, iterator-friendly API. Синтаксис [устарело] |
| [We Have Named Arguments at Home](https://corrode.dev/blog/named-arguments-at-home/) | corrode, 2026 | Builder и структуры-параметры вместо именованных аргументов |

## Владение, мутабельность и графы

**Ключевые выводы**
- Брать самый слабый инструмент внутренней мутабельности, которого хватает: `Cell` → `RefCell` → `OnceCell`/`LazyLock` → `Mutex`/атомики.
- Граф объектов со взаимными ссылками строить на индексах или хэндлах в арене (`Vec`, slotmap), а не на `Rc<RefCell<_>>`. Это даёт локальность кэша, сериализуемость и отсутствие паник `BorrowMutError` в рантайме.
- Удаляемые элементы адресовать поколенческими индексами: устаревший хэндл обнаруживается, а не указывает на чужие данные.
- Долгоживущие структуры хранят owned-данные. Lifetimes в полях оставлять для коротких view (парсер над `&str`, итератор).
- Значение из `&mut` вынимать через `mem::take`/`mem::replace`, а не клонировать.

**Выбор внутренней мутабельности.** Источники: [std::cell](https://doc.rust-lang.org/std/cell/index.html), [Effective Rust, Item 8](https://effective-rust.com/references.html).

| Тип | Потоки | Когда |
|---|---|---|
| `Cell<T>` | один | Небольшие `Copy`-значения, get/set без заимствований |
| `RefCell<T>` | один | Нужна ссылка внутрь; конфликт заимствований — паника в рантайме |
| `OnceCell<T>` / `LazyCell<T>` | один | Ленивая инициализация один раз |
| `OnceLock<T>` / `LazyLock<T>` | много | Глобальные и статические ленивые значения; заменяют крейты `lazy_static` и `once_cell` |
| `Mutex<T>` / `RwLock<T>` | много | Составные данные; см. `concurrency-async-unsafe.md` |
| `Atomic*` | много | Счётчики, флаги, одиночные числа |

```rust
use std::sync::LazyLock;
static WORDS: LazyLock<Vec<&'static str>> = LazyLock::new(|| "a b c".split(' ').collect());
```

**Хэндлы и арены вместо ссылок.** Когда применять: графы, деревья с parent-ссылками, игровые сущности, AST, любые взаимные связи. Готовые крейты: `slotmap`, `generational-arena`, `petgraph`, `typed-arena`, `bumpalo`. Источники: [floooh: Handles are the better pointers](https://floooh.github.io/2018/06/17/handles-vs-pointers.html), [Niko Matsakis: Modeling graphs using vector indices](https://smallcultfollowing.com/babysteps/blog/2015/04/06/modeling-graphs-in-rust-using-vector-indices/).
```rust
#[derive(Clone, Copy, PartialEq, Eq, Hash, Debug)]
pub struct NodeId(usize);

pub struct Graph { nodes: Vec<Node> }
pub struct Node { value: i32, edges: Vec<NodeId> }

impl Graph {
    pub fn add(&mut self, value: i32) -> NodeId {
        self.nodes.push(Node { value, edges: Vec::new() });
        NodeId(self.nodes.len() - 1)
    }
}
```

**Не злоупотреблять `Rc<RefCell<_>>`.** Когда это нормально: небольшой однопоточный граф без горячего пути, например дерево UI-виджетов. Когда переделывать: `borrow_mut()` встречается во многих местах, появились `Weak` для разрыва циклов, бывают паники `BorrowMutError`. Варианты: один владелец передаёт `&mut` вниз по стеку; сообщения вместо общего состояния; индексы. Источники: [Catherine West, RustConf 2018](https://kyren.github.io/2018/09/14/rustconf-talk.html), [without.boats: References are like jumps](https://without.boats/blog/references-are-like-jumps/).

**`mem::take` / `mem::replace`.** Когда применять: смена варианта enum или перенос буфера из `&mut self`. Источник: [rust-unofficial](https://rust-unofficial.github.io/patterns/idioms/mem-replace.html).
```rust
enum State { Running { buf: Vec<u8> }, Done(Vec<u8>) }

fn finish(s: &mut State) {
    if let State::Running { buf } = s {
        *s = State::Done(std::mem::take(buf)); // без clone
    }
}
```

| Ресурс | Автор, год | Зачем читать |
|---|---|---|
| [RustConf 2018 closing keynote](https://kyren.github.io/2018/09/14/rustconf-talk.html) | Catherine West, 2018 | Как ООП и `Rc<RefCell>` воюют с borrowck и почему помогают ECS и индексы |
| [Don't Worry About Lifetimes](https://corrode.dev/blog/lifetimes/) | corrode, 2024 | Owned-данные по умолчанию, lifetimes по необходимости |
| [Ownership](https://without.boats/blog/ownership/) | without.boats, 2024 | Зачем вообще нужны правила владения |

## Ошибки и паника

**Ключевые выводы**
- Библиотеки возвращают типизированные ошибки, по которым можно делать `match` (`thiserror`; `snafu`, если нужен богатый контекст и backtrace). Варианты помечать `#[non_exhaustive]`.
- Типы ошибок делать мелкогранулярными, по одному на операцию или единицу сбоя. Один enum на весь крейт — лёгкий запах.
- Приложения используют `anyhow` (или `eyre`) и добавляют контекст через `.context(...)` на каждом уровне. Конверсия происходит на границе.
- Ошибка служит двум целям: управлению потоком (вызывающему коду нужен вариант) и отчёту (оператору нужна цепочка причин). Проектировать под обе.
- Паника — для багов и нарушенных инвариантов. `unwrap` уместен в тестах, примерах и там, где паника означает баг; в остальных случаях `expect("почему это невозможно")`.

Источники: [Luca Palmieri](https://www.lpalmieri.com/posts/error-handling-rust/), [Sabrina Jewson](https://sabrinajewson.org/blog/errors), [BurntSushi](https://burntsushi.net/unwrap/), [Effective Rust, Item 4](https://effective-rust.com/errors.html).
```rust
// Библиотека
#[derive(Debug, thiserror::Error)]
#[non_exhaustive]
pub enum LoadConfigError {
    #[error("строка {line}: неизвестный ключ `{key}`")]
    UnknownKey { line: usize, key: String },
    #[error("не удалось прочитать файл")]
    Io(#[from] std::io::Error),
}

// Приложение
use anyhow::Context;
fn run() -> anyhow::Result<()> {
    let _text = std::fs::read_to_string("app.toml").context("чтение app.toml")?;
    Ok(())
}
```

| Ресурс | Автор, год | Зачем читать |
|---|---|---|
| [Error handling in iroh](https://www.iroh.computer/blog/error-handling-in-iroh) | n0, 2025 | Ошибки и backtrace в реальной библиотеке, переход к snafu-стилю |
| [Study of std::io::Error](https://matklad.github.io/2020/10/15/study-of-std-io-error.html) | matklad, 2020 | `io::Error` как образец: kind + непрозрачное представление + payload |
| [Don't Unwrap Options](https://corrode.dev/blog/rust-option-handling-best-practices/) | corrode, 2024 | Комбинаторы `Option` и `let-else` вместо `unwrap` |

## Антипаттерны

**Ключевые выводы**
- `.clone()`, добавленный только чтобы замолчал borrowck, прячет проблему владения. Сначала сузить заимствование или переупорядочить код.
- `Deref` не использовать для имитации наследования: только для умных указателей и обёрток, которые прозрачно являются `T`.
- `#![deny(warnings)]` в исходниках ломает сборку на новом компиляторе. Строгость включать в CI.
- `String`, `&str` и `bool` вместо доменных типов — «stringly typed» код. Лечится enum и newtype.
- Не использовать glob-импорты и самодельные prelude в коде приложения: читателю не видно, откуда имя.

**Clone ради borrowck.** Источник: [rust-unofficial](https://rust-unofficial.github.io/patterns/anti_patterns/borrow_clone.html).
```rust
// Плохо: клон только ради тишины компилятора
let name = user.name.clone();
user.touch();
println!("{name}");
// Лучше: переупорядочить, чтобы заимствования не пересекались
user.touch();
println!("{}", user.name);
```

**Deref-полиморфизм.** Источник: [rust-unofficial](https://rust-unofficial.github.io/patterns/anti_patterns/deref.html).
```rust
pub struct Admin { user: User }
// Плохо: impl Deref<Target = User> for Admin — «Admin наследует User»
// Лучше: явная делегация или общий трейт
impl Admin { pub fn name(&self) -> &str { &self.user.name } }
```

**`#![deny(warnings)]`.** Источник: [rust-unofficial](https://rust-unofficial.github.io/patterns/anti_patterns/deny-warnings.html). Вместо него в CI: `cargo clippy --all-targets -- -D warnings` или `RUSTFLAGS="-D warnings"`; уровни lint — в `[workspace.lints]` (см. ниже).

**Stringly typed.** Источники: [API Guidelines C-CUSTOM-TYPE](https://rust-lang.github.io/api-guidelines/checklist.html), [Effective Rust, Item 1](https://effective-rust.com/use-types.html).
```rust
// Плохо: set_mode("fast", true) — что значит true?
pub enum Mode { Fast, Safe }
pub enum Verbosity { Quiet, Verbose }
pub fn set_mode(mode: Mode, verbosity: Verbosity) { /* ... */ }
```

| Ресурс | Автор, год | Зачем читать |
|---|---|---|
| [Rust Design Patterns: Anti-patterns](https://rust-unofficial.github.io/patterns/) | сообщество, живой | Канонический список с объяснениями |
| [Don't Use Preludes And Globs](https://corrode.dev/blog/dont-use-preludes-and-globs/) | corrode, 2024 | Почему glob-импорты вредят читаемости и semver |
| [Pitfalls of Safe Rust](https://corrode.dev/blog/pitfalls-of-safe-rust/) | corrode, 2025 | Переполнения, `as`-касты, паники в безопасном коде |
| [Sharp Edges In The Rust Standard Library](https://corrode.dev/blog/sharp-edges-in-rust-std/) | corrode, 2025 | Неочевидное поведение std |

## Линты и сдвиги терминологии

**Ключевые выводы**
- «Object safety» теперь называется **dyn compatibility**. Старый термин встречается в текстах до 2024 года и в старых сообщениях компилятора.
- Lint настраивать таблицей `[workspace.lints]` в корневом `Cargo.toml` (Cargo 1.74+); каждый крейт подключает её через `[lints] workspace = true`. Атрибуты `#![deny(...)]` в корне крейта — [устарело] как способ общей настройки.
- `clippy::pedantic` включать на уровне workspace с приоритетом `-1` и глушить шумные lint точечно.
- Подавлять lint через `#[expect(lint, reason = "...")]` (Rust 1.81+): компилятор предупредит, когда подавление станет лишним.
- Полезные restriction-lint: `undocumented_unsafe_blocks`, `dbg_macro`, `todo`, `unwrap_used`/`expect_used` (в приложениях), `print_stdout` (в библиотеках).
- Другие [устарело]: `lazy_static`/`once_cell` → `std::sync::LazyLock`/`OnceLock`; `#[async_trait]` без нужды в `dyn` → нативный `async fn` в трейтах; `try!` → `?`; `Box<Trait>` → `Box<dyn Trait>`.

```toml
# Корневой Cargo.toml
[workspace.lints.rust]
unsafe_op_in_unsafe_fn = "deny"

[workspace.lints.clippy]
pedantic = { level = "warn", priority = -1 }
missing_errors_doc = "allow"
missing_panics_doc = "allow"
undocumented_unsafe_blocks = "warn"

# Cargo.toml каждого крейта
[lints]
workspace = true
```

| Ресурс | Автор, год | Зачем читать |
|---|---|---|
| [Clippy pedantic в workspace](https://coreyja.com/notes/clippy-pedantic-workspace) | coreyja | Готовая конфигурация `[workspace.lints]` |
| [Useful Clippy Lints](https://rtaw.co.uk/posts/clippy-lints/) | rtaw | Подборка полезных lint за пределами default |
| [Effective Rust, Item 29: Listen to Clippy](https://effective-rust.com/clippy.html) | David Drysdale, 2024 | Как работать с Clippy в проекте |

## Чеклист code review: идиоматичный Rust

1. Доменные значения (ID, деньги, единицы, проверенные строки) — newtype или enum, а не голые `u64`/`String`/`bool`.
2. Внешний ввод разбирается на границе в типы с инвариантами; внутри нет повторных проверок одного и того же.
3. Нет согласуемых вручную флагов и `Option`: взаимоисключающие состояния — enum с данными.
4. Аргументы только для чтения — `&str`/`&[T]`/`&Path`, а не `&String`/`&Vec<T>`/`&PathBuf`.
5. Generic-функции с тяжёлым телом вынесли тело в негенерик-функцию.
6. Конверсии через `From`/`TryFrom`, а не самописные `to_x()`; имена `as_`/`to_`/`into_` соответствуют стоимости.
7. Нет `.clone()`, добавленного только ради borrowck; каждый clone на горячем пути обоснован.
8. Нет `Rc<RefCell<_>>`/`Arc<Mutex<_>>`, где хватило бы одного владельца, `&mut` или индексов.
9. Взаимосвязанные сущности адресуются индексами или хэндлами; при удалении — поколенческими.
10. Внутренняя мутабельность — самый слабый подходящий примитив.
11. Библиотека возвращает типизированные мелкогранулярные ошибки; приложение добавляет контекст (`.context(...)`).
12. Нет `unwrap()` на путях с внешними данными; `expect` объясняет, почему значение есть.
13. Ресурсы освобождаются через `Drop`; там, где важен результат закрытия, есть явный `close() -> Result`.
14. Публичные enum и структуры, которые будут расти, помечены `#[non_exhaustive]`; поля приватные, если есть инварианты.
15. Трейты, которые не должны реализовываться снаружи, запечатаны.
16. Выбор generics/`dyn`/enum осознан: закрытое множество не спрятано за `dyn`.
17. Нет `Deref` для имитации наследования.
18. Индексные циклы заменены итераторами там, где это читается лучше; API возвращает `impl Iterator`.
19. Долгоживущие структуры без лишних lifetime-параметров.
20. Нет glob-импортов, кроме `use super::*` в тестах и официальных prelude.
21. В исходниках нет `#![deny(warnings)]`; lint настроены в `[workspace.lints]`, подавления — через `#[expect(..., reason = "...")]`.
22. `cargo clippy --all-targets -- -D warnings` и `cargo fmt --check` проходят.
23. Публичные элементы документированы; у функций, возвращающих `Result` или способных паниковать, есть разделы `# Errors`/`# Panics`.

## Влиятельные статьи

| Статья | Автор, год | Зачем читать |
|---|---|---|
| [Parse, don't validate](https://lexi-lambda.github.io/blog/2019/11/05/parse-don-t-validate/) | Alexis King, 2019 | Тип как доказательство проверки |
| [The Typestate Pattern in Rust](https://cliffle.com/blog/rust-typestate/) | Cliff Biffle, ~2019 | Состояние в типе, переходы потребляют `self` |
| [Typed Design Patterns for the Functional Era](https://arxiv.org/abs/2307.07069) | Will Crichton, 2023 | Четыре типовых паттерна на Rust |
| [Builder with typestate in Rust](https://www.greyblake.com/blog/builder-with-typestate-in-rust/) | Serhii Potapov | Обязательные поля builder проверяются при компиляции |
| [Two Beautiful Rust Programs](https://matklad.github.io/2020/07/15/two-beautiful-programs.html) | matklad, 2020 | Что именно ловит borrowck. Сейчас вместо крейтов — `std::thread::scope` (1.63) |
| [Rust's Ugly Syntax](https://matklad.github.io/2023/01/26/rusts-ugly-syntax.html) | matklad, 2023 | Каждый элемент сигнатуры несёт смысл |
| [When Rust Gets Ugly](https://corrode.dev/blog/ugly/) | Matthias Endler, 2026 | Продолжение в духе matklad |
| [Error Handling In Rust: A Deep Dive](https://www.lpalmieri.com/posts/error-handling-rust/) | Luca Palmieri, ~2021 | Рамка для проектирования ошибок |
| [Modular Errors in Rust](https://sabrinajewson.org/blog/errors) | Sabrina Jewson, 2023 | Против одного enum на крейт |
| [Using unwrap() in Rust is Okay](https://burntsushi.net/unwrap/) | Andrew Gallant, 2022 | Паника для багов, не для ошибок |
| [A definitive guide to sealed traits](https://predr.ag/blog/definitive-guide-to-sealed-traits-in-rust/) | Predrag Gruevski, 2023 | Запечатывание трейтов и semver |
| [The Secret Life of Cows](https://deterministic.space/secret-life-of-cows.html) | Pascal Hertleif, 2018 | `Cow` в возврате и аргументах |
| [Handles are the better pointers](https://floooh.github.io/2018/06/17/handles-vs-pointers.html) | Andre Weissflog, 2018 | Хэндлы с поколениями вместо указателей |
| [References are like jumps](https://without.boats/blog/references-are-like-jumps/) | without.boats, 2024 | Ссылки как «структурный» aliasing |
| [Iterators and traversables](https://without.boats/blog/iterators-and-traversables/) | without.boats, 2024 | Внешние итераторы против внутреннего обхода |
| [Practical Rust API Design](https://corrode.dev/blog/practical-rust-api-design/) | Matthias Endler, 2026 | Самая свежая выжимка по API |
| [Make Illegal States Unrepresentable](https://corrode.dev/blog/illegal-state/), [Aim For Immutability](https://corrode.dev/blog/immutability/), [Thinking in Expressions](https://corrode.dev/blog/expressions/) | corrode, 2023–2025 | Короткие тексты на каждый день |
| [Type-Driven Development in Rust](https://www.ruggero.io/blog/rust_type_driven_development_guide/) | ruggero.io, 2025 | Рефакторинг в сторону типов |
| [Crust of Rust](https://www.youtube.com/@jonhoo) | Jon Gjengset, с 2020 | Лайвкодинг: lifetimes, итераторы, `Cow`, dispatch, interior mutability |
| [SE Radio 659: Idiomatic Rust](https://se-radio.net/2025/03/se-radio-659-brenden-matthews-on-idiomatic-rust/) | Brenden Matthews, 2025 | Подкаст: generics, трейты, паттерны и антипаттерны |
