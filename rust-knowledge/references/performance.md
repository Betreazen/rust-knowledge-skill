# Производительность Rust

Справочник для ускорения Rust-кода и сборки. Здесь есть порядок работы, таблица «симптом → техника → источник» для быстрого поиска, разделы по профилированию, памяти, циклам, профилям сборки, времени компиляции, хэшированию, параллелизму и I/O, разборы реальных случаев и антипаттерны. Опорный источник — [The Rust Performance Book](https://nnethercote.github.io/perf-book/) (Nicholas Nethercote, живой документ). Остальные ресурсы раскрывают отдельные темы подробнее. Актуально на октябрь 2026 года (stable Rust 1.90+, edition 2024).

- [Метод: устранить работу → измерить → тюнить](#метод-устранить-работу--измерить--тюнить)
- [Симптом → техника → источник](#симптом--техника--источник)
- [Профилирование и бенчмарки](#профилирование-и-бенчмарки)
- [Память, аллокации и layout](#память-аллокации-и-layout)
- [Итераторы, bounds checks и SIMD](#итераторы-bounds-checks-и-simd)
- [Профили сборки: LTO, codegen-units, PGO, target-cpu, panic=abort](#профили-сборки-lto-codegen-units-pgo-target-cpu-panicabort)
- [Скорость компиляции](#скорость-компиляции)
- [Хэширование](#хэширование)
- [Параллелизм и I/O](#параллелизм-и-io)
- [Разборы реальных случаев](#разборы-реальных-случаев)
- [Антипаттерны](#антипаттерны)

---

## Метод: устранить работу → измерить → тюнить

Сначала профилирование. Потом, примерно в порядке убывания отдачи:

1. алгоритмическое устранение работы;
2. сокращение аллокаций;
3. дешёвое хэширование;
4. раскладка данных и размеры типов;
5. форма циклов, при которой исчезают bounds checks и работает автовекторизация;
6. явный SIMD или rayon;
7. флаги сборки (LTO, codegen-units, PGO, target-cpu).

Самый наглядный пример — uv. Его скорость объясняется прежде всего отказом от легаси-форматов и опорой на новые стандарты, а не самим Rust: «Speed comes from elimination» ([nesbitt.io](https://nesbitt.io/2025/12/26/how-uv-got-so-fast.html)).

### Ключевые выводы

- Начинайте с вопроса «можно ли эту работу не делать вообще»: кэш, ранний выход, special-casing частого случая, другой алгоритм. Микротюнинг даёт проценты, устранение работы даёт порядки.
- Не оптимизируйте без профиля. Горячая точка почти никогда не там, где кажется.
- Меряйте только release-сборку. Debug-сборка медленнее в разы, в ней не убираются bounds checks и не работает инлайнинг, поэтому её цифры ничего не говорят.
- После каждой правки меряйте снова и сравнивайте с сохранённым baseline. Если улучшение меньше шума, это не улучшение.
- Флаги сборки и `unsafe` оставляйте напоследок. Выигрыш от них зависит от нагрузки, и его надо подтверждать бенчмарком.

### Как бенчмаркать честно

- **Профиль.** `cargo bench` по умолчанию собирает с профилем `bench`, который наследует `release`. Если меряете бинарник руками, используйте `--release` или свой профиль, унаследованный от release (см. раздел про профили).
- **`std::hint::black_box`.** Оборачивайте им входы и результат, иначе оптимизатор может посчитать значение на этапе компиляции или выбросить неиспользуемое вычисление. В criterion и divan он тоже есть, но `std::hint::black_box` стабилен и в std.
- **Baseline.** Сохраняйте замер до изменения и сравнивайте с ним, например `cargo bench -- --save-baseline before`, а после правки `cargo bench -- --baseline before` (criterion). Сравнивайте распределения и доверительные интервалы, а не одно число.
- **Шум.** Одна и та же машина, тот же тулчейн, закрытые фоновые задачи, по возможности фиксированная частота CPU и питание от сети. Повторяйте прогон и смотрите на разброс. Разницу в 1–3% на ноутбуке без повторов не стоит считать реальной.
- **Реалистичные входы.** Микробенчмарк на 10 элементах, у которых всё лежит в L1, не скажет ничего о 10 млн записей. Нужны и микро-, и end-to-end замеры на представительных данных.
- **CI.** Wall-clock на общих раннерах слишком шумит. Для гейтинга регрессий используйте подсчёт инструкций и кэш-промахов (Gungraun через Valgrind), а историю храните в Bencher.
- **Проверяйте codegen.** Если оптимизация должна убрать bounds check или включить векторизацию, подтвердите это в ассемблере (`cargo-show-asm`), а не только по времени.

| Ресурс | Автор, год | Зачем читать |
|---|---|---|
| [The Rust Performance Book](https://nnethercote.github.io/perf-book/) | Nicholas Nethercote и др., с 2020, живой | Единая точка входа: главы Benchmarking, Build Configuration, Linting, Profiling, Inlining, Hashing, Heap Allocations, Type Sizes, Standard Library Types, Iterators, Bounds Checks, I/O, Logging and Debugging, Wrapper Types, Machine Code, Parallelism, General Tips, Compile Times |
| [If you want performance, cheat!](https://vorner.github.io/2020/09/03/performance-cheating.html) | vorner (Michal Vaner), 2020 | Главный выигрыш даёт изменение задачи, а не микротюнинг. Принцип верен, конкретные примеры устарели |
| [How uv got so fast](https://nesbitt.io/2025/12/26/how-uv-got-so-fast.html) | Andrew Nesbitt, декабрь 2025 | Аргумент против тезиса «перепиши на Rust, и станет в 10 раз быстрее»: скорость даёт устранение работы |
| [Read Rust: Performance](https://readrust.net/performance) | агрегатор | Подборка статей. Многие записи старые, смотрите на даты |

---

## Симптом → техника → источник

Основная таблица для поиска. Начинайте с симптома, который показал профайлер или ревью.

| Симптом | Техника | Источник |
|---|---|---|
| Профиль не снят, непонятно, где время | `samply` или `cargo flamegraph` на release-сборке с `debug = "line-tables-only"` и `-C force-frame-pointers=yes` | [Perf Book: Profiling](https://nnethercote.github.io/perf-book/), [samply](https://github.com/mstange/samply), [cargo-flamegraph](https://github.com/flamegraph-rs/flamegraph) |
| Бенчмарки в CI шумят и ложно краснеют | Подсчёт инструкций через Gungraun, хранение истории в Bencher | [Gungraun guide](https://bencher.dev/learn/benchmarking/rust/gungraun/), [Bencher](https://bencher.dev/docs/explanation/adapters/) |
| В профиле много `malloc`/`free`, `alloc::raw_vec::grow` | `Vec::with_capacity`/`reserve`, переиспользование буфера через `clear()`, вынос аллокации из цикла | [Perf Book: Heap Allocations](https://nnethercote.github.io/perf-book/) |
| Много маленьких коллекций (обычно 0–8 элементов) | `SmallVec`/`ArrayVec`/`smallstr`: данные на стеке до порога | [Perf Book: Heap Allocations](https://nnethercote.github.io/perf-book/) |
| `clone()` ради borrow checker в горячем пути | Передавать `&T`/`&[T]`, перестроить заимствования, для общих неизменяемых данных `Arc`/`Rc` | [Perf Book: Heap Allocations](https://nnethercote.github.io/perf-book/) |
| Частые `to_string()`/`format!`/`String::from` | `&str` вместо `String`, `Cow<str>`, когда менять обычно нечего, `write!` в переиспользуемый `String` | [The Secret Life of Cows](https://deterministic.space/secret-life-of-cows.html), [Perf Book](https://nnethercote.github.io/perf-book/) |
| Одни и те же строки сравниваются и хэшируются снова и снова | Интернирование: строка один раз, дальше `u32`-идентификатор (общая практика) | — |
| Масса мелких объектов с общим временем жизни (AST, кадр, запрос) | Арена: `bumpalo`, освобождение разом | [bumpalo](https://github.com/fitzgen/bumpalo), [туториал по арене](https://blog.morj.men/posts/rust-arena.html) |
| Аллокатор горячий в многопоточном сервисе | `#[global_allocator]` с mimalloc или jemalloc, обязательно с бенчмарком | [mimalloc_rust](https://github.com/purpleprotocol/mimalloc_rust), [jemallocator](https://github.com/tikv/jemallocator) |
| Непонятно, кто и сколько аллоцирует | `dhat-rs` как глобальный аллокатор, проверка числа аллокаций в тесте | [dhat-rs](https://github.com/nnethercote/dhat-rs) |
| `HashMap` горячий, ключи доверенные | `rustc_hash::FxHashMap`, foldhash, ahash или `hashbrown` (у него foldhash по умолчанию) | [Perf Book: Hashing](https://nnethercote.github.io/perf-book/), [hashbrown PR #563](https://github.com/rust-lang/hashbrown/pull/563) |
| Хэш-ключи — строки или срезы байтов | rapidhash или foldhash, заранее посчитанный хэш, интернирование | [rapidhash](https://github.com/hoxxep/rapidhash) |
| Ключи плотные малые целые (0..N) | `Vec<T>` по индексу вместо хэш-карты (общая практика) | — |
| Ключи приходят от недоверенного клиента | Оставить SipHash (`std` `RandomState`), он устойчив к HashDoS | [Perf Book: Hashing](https://nnethercote.github.io/perf-book/) |
| В asm горячего цикла видны `panic_bounds_check` | Итераторы вместо индексов, `zip`, перенарезка `&v[..n]` до цикла, `assert!` длины заранее, `chunks_exact` | [bounds-check cookbook](https://github.com/Shnatsel/bounds-check-cookbook), [Shnatsel](https://shnatsel.medium.com/how-to-avoid-bounds-checks-in-rust-without-unsafe-f65e618b4c1e) |
| Цикл по числам не векторизуется | Обработка блоками (`as_chunks`, `chunks_exact`), без ранних выходов во внутреннем цикле, проверка в `cargo-show-asm` | [The state of SIMD in Rust in 2026](https://shnatsel.github.io/state-of-simd-rust-2026/), [cargo-show-asm](https://github.com/pacak/cargo-show-asm) |
| Сумма или редукция `f32`/`f64` не векторизуется | Несколько независимых аккумуляторов или algebraic float ops, которые разрешают переупорядочивание | [SIMD 2026](https://shnatsel.github.io/state-of-simd-rust-2026/) |
| Нужен явный SIMD на stable | `fearless_simd` (по умолчанию), `wide`, `pulp`; `std::simd` только на nightly | [SIMD 2026](https://shnatsel.github.io/state-of-simd-rust-2026/), [pythonspeed](https://pythonspeed.com/articles/simd-stable-rust/) |
| Хочется AVX2/AVX-512, но бинарник пойдёт на чужое железо | Runtime-мультиверсионирование (`multiversion`, `#[simd]` в fearless_simd) вместо `target-cpu=native` | [SIMD 2026](https://shnatsel.github.io/state-of-simd-rust-2026/) |
| Тип большой, кэш-промахи, `memcpy` в профиле | Боксить большие редкие варианты enum, `assert!(size_of::<T>() <= N)` в тесте, `-Zprint-type-sizes` | [Perf Book: Type Sizes](https://nnethercote.github.io/perf-book/) |
| Цикл читает одно поле из большой структуры | Struct-of-arrays: отдельные `Vec` на горячие поля (общая практика data-oriented design) | — |
| Граф из `Rc<RefCell<_>>`, плохая локальность, паники borrow | Индексы или хэндлы с поколениями (`u32`) в плоском `Vec` | [Handles are the better pointers](https://floooh.github.io/2018/06/17/handles-vs-pointers.html) |
| `Box<dyn Trait>`/`&dyn` в горячем цикле | Generics (мономорфизация) или enum с `match`, если набор типов закрыт (общая практика) | [Rust for Rustaceans](https://nostarch.com/rust-rustaceans) |
| Промежуточные `collect()` в `Vec` между стадиями | Склеить стадии в одну цепочку итераторов, `extend` в переиспользуемый буфер | [Perf Book: Iterators](https://nnethercote.github.io/perf-book/) |
| `println!` в цикле, медленный вывод | `let mut out = BufWriter::new(stdout().lock());` и `writeln!(out, ..)` | [Perf Book: I/O](https://nnethercote.github.io/perf-book/) |
| Медленное чтение файла построчно | `BufReader` и `read_line` в один переиспользуемый `String` вместо `lines()`, работа с `&[u8]`, если UTF-8 не нужен | [Perf Book: I/O](https://nnethercote.github.io/perf-book/) |
| Поиск разделителей в байтах | `memchr` (SIMD внутри) | [1BRC: naveenaidu](https://naveenaidu.dev/tackling-the-1-billion-row-challenge-in-rust-a-journey-from-5-minutes-to-9-seconds/) |
| Разбор огромного текстового файла | mmap или крупные чанки, разбор байтов, fixed-point вместо `f64::from_str`, разбиение по потокам на границах строк | [1BRC без зависимостей](https://rpallas92.github.io/1brc/) |
| Независимые итерации, загружено одно ядро | `rayon`: `par_iter()`; для мелкой работы `with_min_len` или чанки | [rayon](https://github.com/rayon-rs/rayon) |
| Потоки толкаются на общем `Mutex`-аккумуляторе | Локальное накопление в каждом потоке, затем слияние (`fold` + `reduce` в rayon) | [Perf Book: Parallelism](https://nnethercote.github.io/perf-book/), [1BRC](https://rpallas92.github.io/1brc/) |
| `thread_local!` в профиле | Читать один раз и передавать ссылку дальше, не обращаться к нему в каждой итерации | [Fast Thread Locals In Rust](https://matklad.github.io/2020/10/03/fast-thread-locals-in-rust.html) |
| Форматирование логов в горячем пути | Проверка уровня до форматирования, вывод логов из горячих циклов | [Perf Book: Logging and Debugging](https://nnethercote.github.io/perf-book/) |
| Нужны последние 5–20% без правок кода | `lto`, `codegen-units = 1`, `panic = "abort"`, затем PGO через `cargo-pgo` | [Perf Book: Build Configuration](https://nnethercote.github.io/perf-book/), [cargo-pgo](https://kobzol.github.io/rust/cargo/2023/07/28/rust-cargo-pgo.html) |
| Бинарник слишком большой | `opt-level = "z"`, `strip`, `panic = "abort"`, LTO | [min-sized-rust](https://github.com/johnthagen/min-sized-rust) |
| Долгая линковка при инкрементальной пересборке | Свежий stable (LLD по умолчанию на x86_64 Linux с 1.90), на других платформах mold или wild | [Rust Blog: LLD 1.90](https://blog.rust-lang.org/2025/09/01/rust-lld-on-1.90.0-stable), [corrode](https://corrode.dev/blog/tips-for-faster-rust-compile-times/) |
| Полная сборка долгая, непонятно почему | `cargo build --timings`, разрезать «узкое горлышко» графа крейтов, урезать фичи зависимостей, убрать тяжёлые proc-macro | [Fast Rust Builds](https://matklad.github.io/2021/09/04/fast-rust-builds.html) |
| Generic-функции раздувают код и время компиляции | Тонкая generic-обёртка, внутри не-generic функция; `&dyn` на границах крейтов | [Fast Rust Builds](https://matklad.github.io/2021/09/04/fast-rust-builds.html) |
| Долгий цикл правка → запуск в debug | `debug = "line-tables-only"` или `0` в dev, Cranelift-бэкенд (nightly), оптимизированные зависимости | [rustc_codegen_cranelift](https://github.com/rust-lang/rustc_codegen_cranelift), [corrode](https://corrode.dev/blog/tips-for-faster-rust-compile-times/) |
| Интеграционные тесты долго линкуются | Один бинарник `tests/it/main.rs` с модулями вместо множества `tests/*.rs` | [matklad: Delete Cargo Integration Tests](https://matklad.github.io/2021/02/27/delete-cargo-integration-tests.html) |
| CI каждый раз собирает зависимости с нуля | `Swatinem/rust-cache` или `sccache` | [Faster CI Builds](https://corrode.dev/blog/tips-for-faster-ci-builds/), [sccache](https://github.com/mozilla/sccache) |
| Огромный сгенерированный крейт компилируется в один поток | Разбить на много крейтов ради параллельного codegen | [Feldera: 30 → 2 минуты](https://www.feldera.com/blog/cutting-down-rust-compile-times-from-30-to-2-minutes-with-one-thousand-crates) |
| Clippy молчит, но код подозрителен | Прогнать `cargo clippy` и разобрать группу `clippy::perf` | [Perf Book: Linting](https://nnethercote.github.io/perf-book/) |

---

## Профилирование и бенчмарки

### Ключевые выводы

- Профилируйте release-сборку с символами. Удобнее завести отдельный профиль `profiling`, унаследованный от `release`, с `debug = "line-tables-only"`, и собирать с `RUSTFLAGS="-C force-frame-pointers=yes"`, чтобы стеки получались корректными.
- CPU-профиль: `samply` — основной кроссплатформенный выбор (Linux, macOS, Windows, интерфейс Firefox Profiler). `cargo flamegraph` удобен, когда есть `perf` или DTrace.
- Аллокации: `dhat-rs` показывает число аллокаций, байты и пик. Его же можно использовать в тесте как регрессионную проверку («не больше N аллокаций»).
- Wall-clock-бенчмарки: criterion — статистический де-факто стандарт с baseline и HTML-отчётами. divan проще (атрибут `#[divan::bench]`), поддерживает generic-бенчи и умеет считать аллокации.
- CI: Gungraun считает инструкции и кэш-промахи через Valgrind, результаты детерминированы, поэтому шумные раннеры ему не мешают. Тренды и алерты на регрессии даёт Bencher.
- Чтобы проверить конкретную функцию, смотрите её ассемблер через `cargo-show-asm` (`cargo asm`). Так проверяют исчезновение bounds checks, инлайнинг и векторизацию.

| Ресурс | Автор, год | Зачем читать |
|---|---|---|
| [criterion.rs](https://github.com/bheisler/criterion.rs) | Brook Heisler и мейнтейнеры | Статистический бенчмарк на stable, baseline, обнаружение регрессий |
| [divan](https://github.com/nvzqz/divan) | Nikolai Vazquez, 2023 | Лёгкий харнесс на атрибутах, generic-бенчи, подсчёт аллокаций |
| [Gungraun guide](https://bencher.dev/learn/benchmarking/rust/gungraun/) | bencher.dev | Детерминированные бенчмарки по инструкциям через Callgrind/Cachegrind/DHAT для CI. Это бывший iai-callgrind: крейт `iai-callgrind` [устарело] (последний релиз 0.16.1, новая работа идёт только в gungraun), исходный `iai` [устарело] → gungraun |
| [Bencher adapters](https://bencher.dev/docs/explanation/adapters/) | Bencher | Непрерывный бенчмаркинг с адаптерами для criterion, divan и gungraun |
| [samply](https://github.com/mstange/samply) | Markus Stange | Сэмплирующий профайлер, результаты открываются в Firefox Profiler |
| [cargo-flamegraph](https://github.com/flamegraph-rs/flamegraph) | flamegraph-rs | Flamegraph одной командой через perf/DTrace |
| [cargo-show-asm](https://github.com/pacak/cargo-show-asm) | pacak | Ассемблер, LLVM IR или MIR одной функции |
| [dhat-rs](https://github.com/nnethercote/dhat-rs) | Nicholas Nethercote | Heap-профайлер и проверки числа аллокаций в тестах |
| [How to Profile Rust Applications](https://oneuptime.com/blog/post/2026-02-03-rust-profiling/view) | OneUptime, февраль 2026 | Вторичный пошаговый гайд по perf, flamegraph и criterion |

---

## Память, аллокации и layout

### Ключевые выводы

- Задавайте ёмкость заранее (`Vec::with_capacity`, `String::with_capacity`, `reserve`), если размер известен или оценивается. Переиспользуйте буферы между итерациями: `clear()` сохраняет ёмкость.
- Возвращайте `&str`/`&[T]` или `Cow`, когда изменение нужно редко. Не клонируйте ради borrow checker в горячем пути: лучше перестроить заимствования.
- Для маленьких коллекций подходят `SmallVec`/`ArrayVec`/`smallstr`. Для объектов с общим временем жизни (фаза, кадр, запрос) — арена `bumpalo`: выделение почти бесплатно, освобождение одним махом.
- Держите горячие типы маленькими. Enum весит столько же, сколько его самый большой вариант, поэтому большие редкие варианты боксите. Размеры проверяйте через `-Zprint-type-sizes` (nightly) или `assert!(std::mem::size_of::<T>() <= N)` в тесте, чтобы ловить регрессии.
- Для графов используйте индексы (`u32`) или хэндлы с поколениями в плоском `Vec` вместо `Rc<RefCell<_>>`. Так данные лежат плотнее, сериализуются без усилий и не дают паник `BorrowMutError`.
- Если цикл читает 1–2 поля из широкой структуры, переходите на struct-of-arrays: кэш-линии будут заполнены полезными данными.
- Смену глобального аллокатора (mimalloc, jemalloc) стоит пробовать для нагрузок с частыми аллокациями, особенно многопоточных. Выигрыш зависит от нагрузки, а независимого воспроизводимого сравнения аллокаторов нет, так что решайте только по своему бенчмарку.

```rust
// Переиспользование буфера вместо аллокации на каждой итерации
let mut buf = String::with_capacity(256);
for item in items {
    buf.clear();
    write!(buf, "{}:{}", item.key, item.value)?; // use std::fmt::Write
    sink.consume(&buf);
}

// Страховка от роста горячего типа
#[test]
fn hot_type_stays_small() {
    assert!(std::mem::size_of::<Event>() <= 32);
}
```

| Ресурс | Автор, год | Зачем читать |
|---|---|---|
| Главы «Heap Allocations» и «Type Sizes» в [Performance Book](https://nnethercote.github.io/perf-book/) | Nicholas Nethercote | `with_capacity`, `clear()`, `SmallVec`/`ArrayVec`, `Cow`, бокс больших вариантов, `-Zprint-type-sizes` |
| [The Secret Life of Cows](https://deterministic.space/secret-life-of-cows.html) | Pascal Hertleif, 2018 | Классика про `Cow<str>`, по-прежнему точна |
| [bumpalo](https://github.com/fitzgen/bumpalo) | Nick Fitzgerald | Bump-аллокатор для фазовых аллокаций |
| [Your own little memory strategy](https://blog.morj.men/posts/rust-arena.html) | blog.morj.men | Туториал по арене в Rust |
| [Handles are the better pointers](https://floooh.github.io/2018/06/17/handles-vs-pointers.html) | Andre Weissflog, 2018 | Индексы с поколениями вместо указателей: локальность и защита от висячих ссылок |
| [mimalloc_rust](https://github.com/purpleprotocol/mimalloc_rust) / [tikv-jemallocator](https://github.com/tikv/jemallocator) | purpleprotocol / TiKV | Замена `#[global_allocator]` |
| [Fast Thread Locals In Rust](https://matklad.github.io/2020/10/03/fast-thread-locals-in-rust.html) | matklad, 2020 | Цена доступа к `thread_local!`. Частично [устарело]: std позже ускорила доступ, но приём «прочитать один раз и передать ссылку» остаётся в силе |
| [Rust for Rustaceans](https://nostarch.com/rust-rustaceans) | Jon Gjengset, 2021 | Layout, `repr`, цена generics и trait objects |

---

## Итераторы, bounds checks и SIMD

### Ключевые выводы

- Итерируйте, а не индексируйте: `iter()`, `zip`, `windows`, `chunks_exact` не дают bounds checks и упрощают векторизацию.
- Если индексы нужны, докажите компилятору границы заранее: перенарежьте `let a = &a[..n]; let b = &b[..n];` до цикла или поставьте `assert!(i_max < v.len())` перед ним. LLVM вынесет проверку из цикла или удалит её. `get_unchecked` — последнее средство, и только с комментарием `// SAFETY:`.
- Автовекторизацию пробуйте первой: обрабатывайте фиксированные блоки (`slice.as_chunks::<N>()`, `chunks_exact`), уберите ранние `break`/`return` из внутреннего цикла и проверьте ассемблер.
- Наивная сумма `f32`/`f64` не векторизуется: строгий порядок IEEE запрещает переупорядочивать операции. Помогают несколько независимых аккумуляторов или algebraic float ops (`algebraic_add` и т.п.), которые, по обзору Shnatsel, стабилизированы в Rust 1.98 [не проверено].
- Для явного SIMD на stable по умолчанию берите `fearless_simd`: минимум `unsafe`, мультиверсионирование через `#[simd]`, AVX-512 только там, где он помогает. `wide` (1.0) умеет тригонометрию, но не мультиверсионирует на x86. `pulp` (и построенный на нём `macerator`) ориентирован на математику и линейную алгебру. Для автовекторизованных циклов подходит крейт `multiversion`.
- `std::simd` (portable SIMD) на октябрь 2026 года по-прежнему только nightly. Его `reduce_sum` плох и по скорости, и по точности.

```rust
// Bounds checks убираются перенарезкой до цикла
fn dot(a: &[f32], b: &[f32]) -> f32 {
    let n = a.len().min(b.len());
    let (a, b) = (&a[..n], &b[..n]);
    a.iter().zip(b).map(|(x, y)| x * y).sum()
}
```

| Ресурс | Автор, год | Зачем читать |
|---|---|---|
| [How to avoid bounds checks in Rust (without unsafe!)](https://shnatsel.medium.com/how-to-avoid-bounds-checks-in-rust-without-unsafe-f65e618b4c1e) | Sergey Davidoff (Shnatsel) | Рецепты устранения bounds checks и их проверка в asm |
| [bounds-check-cookbook](https://github.com/Shnatsel/bounds-check-cookbook) | Shnatsel | Репозиторий с примерами до и после |
| [The state of SIMD in Rust in 2026](https://shnatsel.github.io/state-of-simd-rust-2026/) | Shnatsel, сентябрь 2026 | Текущая карта SIMD-крейтов и автовекторизации |
| [The state of SIMD in Rust in 2025](https://shnatsel.medium.com/the-state-of-simd-in-rust-in-2025-32c263e5f53d) | Shnatsel, 2025 | [устарело] → версия 2026 года выше |
| [Using portable SIMD in stable Rust](https://pythonspeed.com/articles/simd-stable-rust/) | Itamar Turner-Trauring | Практические альтернативы `std::simd` на stable |
| [RFC 2977 (stdsimd)](https://rust-lang.github.io/rfcs/2977-stdsimd.html) | rust-lang | Дизайн portable SIMD |
| [SIMD on GPU](https://www.vectorware.com/blog/simd-on-gpu/) | Vectorware, 2026 | Новое направление: `core::simd` на GPU |
| Глава «Iterators» в [Performance Book](https://nnethercote.github.io/perf-book/) | Nicholas Nethercote | Цепочки итераторов вместо промежуточных `collect` |

Крейт `packed_simd` [устарело] давно заброшен → `fearless_simd`/`wide`/`pulp` на stable или `std::simd` на nightly.

---

## Профили сборки: LTO, codegen-units, PGO, target-cpu, panic=abort

### Ключевые выводы

- Обычный `release` (`opt-level = 3`, thin-local LTO, 16 codegen-units) уже быстрый. Дополнительные настройки дают оставшиеся проценты ценой времени сборки, поэтому выносите их в отдельный профиль `dist`, а не в `release`, который нужен для итераций.
- `lto = "fat"` и `codegen-units = 1` дают межмодульный инлайнинг и обычно чуть более быстрый и меньший бинарник, но сильно удлиняют сборку. `lto = "thin"` — компромисс.
- `panic = "abort"` убирает таблицы раскрутки и немного уменьшает код, но ломает `catch_unwind` и то, что на нём держится (изоляция паник в пулах потоков и серверах, некоторые FFI-сценарии). Тесты, бенчмарки, build-скрипты и proc-macro игнорируют эту настройку.
- PGO (`cargo pgo build` → прогон представительной нагрузки → `cargo pgo optimize`) с опциональным BOLT поверх работает, только если профильная нагрузка похожа на реальную. На ней же rustc и Clippy получили заметные ускорения.
- `-C target-cpu=native` подходит только для бинарника, который запускается на той же машине. Для дистрибуции выбирайте уровень (`x86-64-v3`) осознанно или используйте runtime-мультиверсионирование.
- Типичного выигрыша от LTO, `codegen-units = 1` или `panic = "abort"` ни один первоисточник не называет: он зависит от нагрузки. Цифра «10–30% от дополнительных оптимизаций компилятора» взята из вторичного блога [не проверено]. Меряйте сами.
- Готовые пресеты (быстрая компиляция, быстрый рантайм, минимальный размер) применяет `cargo-wizard`.

```toml
# Cargo.toml

# [profile.release] оставляем стандартным: быстрые итерации

# профиль для профилирования (cargo build --profile profiling)
[profile.profiling]
inherits = "release"
debug = "line-tables-only"   # символы для samply/perf; на скорость не влияет

# профиль для дистрибуции (cargo build --profile dist)
[profile.dist]
inherits = "release"
lto = "fat"            # или "thin": почти тот же эффект, сборка быстрее
codegen-units = 1      # лучше оптимизация, сборка медленнее
panic = "abort"        # только если не нужен catch_unwind
strip = "symbols"      # меньше бинарник; для профилирования не подходит

# debug-сборка, где зависимости оптимизированы (игры, численные задачи):
[profile.dev.package."*"]
opt-level = 3
```

```toml
# .cargo/config.toml — целевой CPU для известного железа
[build]
rustflags = ["-C", "target-cpu=x86-64-v3"]   # "native" только для локального запуска
```

| Настройка | Плюс | Цена / риск |
|---|---|---|
| `lto = "fat"` | Межкрейтовый инлайнинг, меньше код | Долгая линковка, большой расход памяти при сборке |
| `lto = "thin"` | Большая часть выигрыша fat LTO | Чуть слабее fat |
| `codegen-units = 1` | Лучше оптимизация внутри крейта | Нет параллельного codegen, сборка медленнее |
| `panic = "abort"` | Меньше код, нет landing pads | Нет `catch_unwind`, паника валит процесс |
| `opt-level = "s"`/`"z"` | Меньший бинарник | Обычно медленнее, чем `3` |
| `target-cpu=native` | AVX2/AVX-512 и т.п. | `SIGILL` на другом CPU |
| PGO / BOLT | Лучшая раскладка кода и ветвлений | Нужна представительная нагрузка, сборка в несколько этапов |
| `debug = "line-tables-only"` в release | Символизация в профайлере | Больше размер бинарника, на скорость не влияет |

| Ресурс | Автор, год | Зачем читать |
|---|---|---|
| Глава «Build Configuration» в [Performance Book](https://nnethercote.github.io/perf-book/) | Nicholas Nethercote | Канонический список: LTO, codegen-units, panic, target-cpu, аллокаторы, PGO |
| [Optimizing Rust programs with PGO and BOLT using cargo-pgo](https://kobzol.github.io/rust/cargo/2023/07/28/rust-cargo-pgo.html) | Jakub Beránek (Kobzol), 2023 | PGO за три команды, BOLT поверх. Детали инструментов могли сместиться, схема та же |
| [Automating Cargo project configuration using cargo-wizard](https://kobzol.github.io/rust/cargo/2024/03/10/rust-cargo-wizard.html) | Kobzol, 2024 | Пресеты профилей одной командой |
| [Speeding up the Rust compiler without changing its code](https://kobzol.github.io/rust/rustc/2022/10/27/speeding-rustc-without-changing-its-code.html) | Kobzol, 2022 | Кейс: PGO, BOLT и LTO на rustc |
| [min-sized-rust](https://github.com/johnthagen/min-sized-rust) | johnthagen | Профиль под размер, контраст к профилю под скорость |
| [Performance tuning Rust in production](https://khimananda.com/blog/performance-tuning-rust-in-production) | khimananda.com, 2026 | Вторичный обзор продакшн-настроек; цифры [не проверено] |

---

## Скорость компиляции

### Ключевые выводы

- Обновляйте тулчейн: rustc ускоряется почти каждый релиз. С 1.90 на `x86_64-unknown-linux-gnu` по умолчанию используется LLD: инкрементальная линковка ripgrep стала примерно в 7 раз быстрее (около 40% end-to-end), сборки debug с нуля — примерно на 20%. Отключается через `-C linker-features=-lld`. За два месяца до сентября 2026 года среднее wall-time компилятора снизилось на 4.57%.
- Ручной `-C link-arg=-fuse-ld=lld` на x86_64 Linux [устарело] с 1.90 → ничего не делать. На других платформах берите mold или wild.
- Время сборки определяется формой графа крейтов не меньше, чем флагами. Граф должен быть плоским и широким, без одного «толстого» крейта, через который проходит всё. Найти узкое место помогает `cargo build --timings`.
- Держите generic-функции тонкими: generic-обёртка конвертирует аргументы и вызывает внутреннюю не-generic функцию, так что мономорфизируется только обёртка. На границах крейтов предпочтителен `&dyn`.
- Тяжёлые proc-macro и лишние фичи зависимостей стоят дорого. Отключайте `default-features`, держите proc-macro ближе к листьям графа.
- Для dev-сборок: `debug = "line-tables-only"` или `debug = 0`, Cranelift-бэкенд (nightly-компонент `rustc-codegen-cranelift-preview` на Linux, macOS и x86_64 Windows) — сборка быстрее, код медленнее.
- Для CI: `Swatinem/rust-cache` или `sccache`, `cargo-nextest`, интеграционные тесты одним бинарником.
- Для огромного сгенерированного кода дробление на сотни крейтов окупается параллельным codegen (Feldera: с 30 до 2 минут). Цена — больше линковки и пересборки из-за унификации фич.

```rust
// Мономорфизируется только тонкая обёртка
pub fn read_config(path: impl AsRef<std::path::Path>) -> std::io::Result<Config> {
    fn inner(path: &std::path::Path) -> std::io::Result<Config> {
        todo!("вся логика здесь, компилируется один раз")
    }
    inner(path.as_ref())
}
```

| Ресурс | Автор, год | Зачем читать |
|---|---|---|
| [Fast Rust Builds](https://matklad.github.io/2021/09/04/fast-rust-builds.html) | matklad, 2021 | Лучший концептуальный текст: граф крейтов и пайплайнинг, приём с внутренней функцией, proc-macro, кэш в CI, `--timings` |
| [Tips For Faster Rust Compile Times](https://corrode.dev/blog/tips-for-faster-rust-compile-times/) | Matthias Endler (corrode.dev), живой | Самый полный актуальный чеклист: кэш, линкеры mold/lld/wild, codegen-опции, proc-macro, workspace, nextest, Docker |
| [Tips for Faster Rust CI Builds](https://corrode.dev/blog/tips-for-faster-ci-builds/) | Matthias Endler | CI: `Swatinem/rust-cache` и другое |
| [Faster linking times with 1.90.0 stable on Linux using the LLD linker](https://blog.rust-lang.org/2025/09/01/rust-lld-on-1.90.0-stable) | Rémy Rakic, сентябрь 2025 | LLD по умолчанию на x86_64 Linux, цифры и способ отключения |
| [rustc_codegen_cranelift](https://github.com/rust-lang/rustc_codegen_cranelift) | bjorn3 | Cranelift-бэкенд для быстрых debug-сборок |
| [Rust Project Primer: Linking](https://rustprojectprimer.com/building/linker.html) / [Codegen](https://rustprojectprimer.com/building/codegen.html) | Rust Project Primer | Практичный выбор линкера и бэкенда |
| [How to speed up the Rust compiler in September 2026](https://nnethercote.github.io/2026/09/30/how-to-speed-up-the-rust-compiler-in-september-2026.html) | Nicholas Nethercote, 2026 | Что даёт обновление тулчейна. Предыдущие выпуски: [Jul 2026](https://nnethercote.github.io/2026/07/31/how-to-speed-up-the-rust-compiler-in-july-2026.html), [Dec 2025](https://nnethercote.github.io/2025/12/05/how-to-speed-up-the-rust-compiler-in-december-2025.html), [May 2025](https://nnethercote.github.io/2025/05/22/how-to-speed-up-the-rust-compiler-in-may-2025.html), [Mar 2025](https://nnethercote.github.io/2025/03/19/how-to-speed-up-the-rust-compiler-in-march-2025.html) |
| [Delete Cargo Integration Tests](https://matklad.github.io/2021/02/27/delete-cargo-integration-tests.html) | matklad, 2021 | Один тестовый бинарник вместо многих: в Cargo это дало ускорение в 3 раза |
| [Cutting down Rust compile times from 30 to 2 minutes with one thousand crates](https://www.feldera.com/blog/cutting-down-rust-compile-times-from-30-to-2-minutes-with-one-thousand-crates) | Feldera | Кейс дробления сгенерированного кода |
| [sccache](https://github.com/mozilla/sccache) | Mozilla | Общий кэш компиляции для CI и команды |

---

## Хэширование

### Ключевые выводы

- `std::collections::HashMap` по умолчанию использует SipHash (`RandomState`). Он устойчив к HashDoS, но медленный. Если ключи доверенные (внутренние идентификаторы, данные компилятора, игровые сущности), меняйте хэшер.
- Варианты: `rustc_hash::FxHashMap` (очень быстрый на целых и коротких ключах), `foldhash` (`foldhash::fast` для карт, `foldhash::quality` для скетчей и HyperLogLog), `ahash`, `rapidhash` (по отзывам быстрее на строках, умеет хэширование на этапе компиляции). Самый простой путь — `hashbrown`, у которого foldhash стоит по умолчанию.
- Если ключи контролирует атакующий (HTTP-заголовки, пользовательский ввод в публичном сервисе), оставляйте SipHash или хэшер со случайным сидом.
- Лучший хэш — тот, который не считается. Плотные малые id лучше класть в `Vec` по индексу. Для двойных поисков используйте `entry`. Строки, которые часто хэшируются, можно интернировать.
- Порядок итерации `HashMap` не детерминирован. Для воспроизводимости (реплеи, снапшот-тесты) используйте `BTreeMap` или `IndexMap`, либо хэшер с фиксированным сидом.
- Цифры скорости из вторичного бенчмарка: около 13 GiB/s у Fx и foldhash против около 1 GiB/s у SipHash на 8-байтных ключах [не проверено]. Это ориентир, а не обещание.

```rust
use rustc_hash::FxHashMap;
let mut counts: FxHashMap<u32, u64> = FxHashMap::default();
*counts.entry(id).or_insert(0) += 1;
```

| Ресурс | Автор, год | Зачем читать |
|---|---|---|
| Глава «Hashing» в [Performance Book](https://nnethercote.github.io/perf-book/) | Nicholas Nethercote | Когда и на что менять SipHash |
| [hashbrown PR #563: Change the default hasher to foldhash](https://github.com/rust-lang/hashbrown/pull/563) | Amanieu | foldhash по умолчанию в hashbrown; в `std::HashMap` по-прежнему SipHash |
| [hashbrown на lib.rs](https://lib.rs/crates/hashbrown) | — | Текущая версия и API |
| [foldhash summary](https://openapps.pro/packages/foldhash) | Orson Peters (автор крейта) | Варианты `fast` и `quality` |
| [rapidhash](https://github.com/hoxxep/rapidhash) | hoxxep | Альтернатива для строковых ключей, хэширование на этапе компиляции |
| [Hashing algorithms for HashMap in Rust](https://blog.devgenius.io/hashing-algorithms-for-hashmap-in-rust-a-deep-dive-into-performance-and-security-3ae181798bb9?gi=0ca073f7de4a) | devgenius | Вторичное сравнение скорости и безопасности хэшеров [не проверено] |

---

## Параллелизм и I/O

### Ключевые выводы

- Для data-parallel работы используйте `rayon`: часто достаточно заменить `iter()` на `par_iter()`. Единица работы должна быть не слишком мелкой: для дешёвых элементов задайте `with_min_len` или обрабатывайте чанки.
- Не накапливайте результат под общим `Mutex` в горячем пути. Накапливайте локально в каждом потоке и сливайте в конце (`fold` + `reduce` в rayon, отдельные карты по потокам в 1BRC).
- Каналы и примитивы для ручного параллелизма даёт `crossbeam`. Для глубокого понимания атомиков и memory ordering читайте Rust Atomics and Locks.
- Не запускайте CPU-тяжёлые вычисления прямо в async-задачах: они блокируют executor. Выносите их в `spawn_blocking` или в пул rayon.
- Вывод: возьмите `stdout().lock()` один раз и оберните в `BufWriter`. `println!` в цикле на каждой строке берёт блокировку и пишет без буфера. Не забудьте, что `BufWriter` сбрасывается при `drop`, а ошибку сброса проверяет только явный `flush()`.
- Ввод: `BufReader` плюс `read_line` в один переиспользуемый `String` вместо `lines()`, который аллоцирует каждую строку. Если UTF-8 не нужен, работайте с `&[u8]` (`read_until`, `split`). Для поиска байтов есть `memchr`.
- Для больших файлов подходят mmap или чтение крупными чанками с разбором прямо из буфера.

```rust
use std::io::{self, BufWriter, Write};
let mut out = BufWriter::new(io::stdout().lock());
for row in rows {
    writeln!(out, "{row}")?;
}
out.flush()?;
```

| Ресурс | Автор, год | Зачем читать |
|---|---|---|
| Глава «Parallelism» в [Performance Book](https://nnethercote.github.io/perf-book/) | Nicholas Nethercote | Указатели на rayon и crossbeam |
| [rayon](https://github.com/rayon-rs/rayon) | Josh Stone, Niko Matsakis | `par_iter`, `join`, гранулярность работы |
| Главы «I/O» и «Standard Library Types» в [Performance Book](https://nnethercote.github.io/perf-book/) | Nicholas Nethercote | Блокировка stdout, `BufReader`/`BufWriter`, переиспользование буфера, байты вместо `String` |
| [Rust Atomics and Locks](https://mara.nl/atomics/) | Mara Bos, 2023 | Атомики, memory ordering, устройство блокировок |
| [Tackling 1BRC in Rust: from 5 minutes to 9 seconds](https://naveenaidu.dev/tackling-the-1-billion-row-challenge-in-rust-a-journey-from-5-minutes-to-9-seconds/) | naveenaidu | Пошаговое применение I/O- и parallel-приёмов |

---

## Разборы реальных случаев

### Ключевые выводы

- **1BRC** (One Billion Row Challenge, январь 2024). Rust-решения сходятся на одном рецепте: mmap или крупные чанки; разбор байтов, а не `str`; fixed-point целые вместо разбора `f64`; дешёвый хэш по срезам байтов; разбиение входа по потокам на границах строк и слияние локальных карт; по желанию SIMD-поиск `\n` и `;` в стиле `memchr`. Результаты: 5.16 с без единой зависимости и ускорение более чем в 33 раза при пошаговой оптимизации (с 5 минут до 9 секунд).
- **uv**. При запуске заявлено ускорение в 8–10 раз относительно pip без кэша и в 80–115 раз с тёплым кэшем. Разбор Несбитта показывает, что главный источник скорости — устранение работы: отказ от `.egg`, игнорирование конфигов pip, обязательные venv, статические метаданные PEP 658. Rust — необходимое, но не достаточное условие.
- **rustc и Clippy**. PGO, BOLT и LTO ускорили сам компилятор без правок кода. В 2026 году Clippy получил 10–30% за счёт удаления пустой виртуальной диспетчеризации и включения PGO, rustdoc тоже заметно ускорился.
- **Feldera**. Сгенерированный код разбили на тысячу крейтов, и сборка ускорилась с 30 до 2 минут за счёт параллельного codegen.
- Общий урок: самые крупные выигрыши структурные (меньше работы, другие данные, другой граф крейтов). Флаги компилятора добавляют проценты поверх.

| Ресурс | Автор, год | Зачем читать |
|---|---|---|
| [1BRC results are in](https://www.morling.dev/blog/1brc-results-are-in/) | Gunnar Morling, 2024 | Условия и итоги челленджа (исходно для Java) |
| [Rust 1BRC without dependencies](https://rpallas92.github.io/1brc/) | rpallas92 | 5.16 с без крейтов |
| [Tackling 1BRC in Rust: from 5 minutes to 9 seconds](https://naveenaidu.dev/tackling-the-1-billion-row-challenge-in-rust-a-journey-from-5-minutes-to-9-seconds/) | naveenaidu | Пошаговый путь с замером каждого шага |
| [One Billion Row Challenge](https://pncnmnp.github.io/blogs/one-billion-row-challenge.html) | pncnmnp | Rust и Python, меньше 10 с |
| [1BRC](https://barrcodes.dev/posts/1brc/) | BarrCodes | Ещё один подробный разбор |
| [The One Billion Row Challenge in Rust](https://baarse.substack.com/p/the-one-billion-row-challenge-in-rust-1) | baarse | Серия на Substack |
| [How uv got so fast](https://nesbitt.io/2025/12/26/how-uv-got-so-fast.html) | Andrew Nesbitt, декабрь 2025 | «Speed comes from elimination» ([обсуждение на HN](https://news.ycombinator.com/item?id=46393992)) |
| [uv: Python packaging in Rust](https://astral.sh/blog/uv) | Astral, февраль 2024 | Анонс и заявленные цифры. Есть также [доклад в Jane Street](https://www.janestreet.com/tech-talks/uv-an-extremely-fast-python-package-manager/) |
| [Speeding up the Rust compiler without changing its code](https://kobzol.github.io/rust/rustc/2022/10/27/speeding-rustc-without-changing-its-code.html) | Kobzol, 2022 | PGO, BOLT, LTO на rustc |
| [How to speed up the Rust compiler in September 2026](https://nnethercote.github.io/2026/09/30/how-to-speed-up-the-rust-compiler-in-september-2026.html) | Nicholas Nethercote, 2026 | Ускорения Clippy и rustdoc |
| [1160 PRs to improve Rust in 2025](https://kobzol.github.io/rust/rustc/2026/01/05/my-rust-contributions-in-2025.html) | Kobzol, январь 2026 | Инфраструктура производительности компилятора (rustc-perf, CI) |

---

## Антипаттерны

### Ключевые выводы

- Ловите их профайлером и группой `clippy::perf`. Большинство из них исправляется локально, без перестройки архитектуры.
- Если антипаттерн не в горячем пути, его не обязательно трогать: читаемость тоже имеет цену.

| Антипаттерн | Почему плохо | Замена |
|---|---|---|
| Бенчмарк или выводы по debug-сборке | Нет оптимизаций, bounds checks на месте, цифры не переносятся на release | `--release` или профиль, унаследованный от release |
| Бенчмарк без `black_box` | Оптимизатор выбрасывает или константно сворачивает измеряемую работу | `std::hint::black_box` для входов и результата |
| «Переписать на Rust, и станет быстрее» | Язык не устраняет лишнюю работу (урок uv) | Сначала алгоритм и устранение работы |
| `clone()` ради borrow checker в горячем цикле | Аллокация и копирование на каждой итерации | Ссылки, перестройка заимствований, `Arc` для общего неизменяемого |
| `String` там, где хватит `&str`/`Cow` | Лишние аллокации | `&str`, `Cow<str>`, `write!` в переиспользуемый буфер |
| Рост `Vec` без `with_capacity` при известном размере | Повторные реаллокации и копирование | `with_capacity`/`reserve` |
| SipHash для внутренних карт с доверенными ключами | Хэширование в несколько раз медленнее нужного | FxHash/foldhash/ahash или `hashbrown` |
| Быстрый хэшер для ключей от атакующего | Уязвимость к HashDoS | SipHash/`RandomState` |
| `println!` в цикле, `lines()` на больших файлах | Блокировка и запись на каждую строку, аллокация на строку | `BufWriter` поверх `stdout().lock()`, `read_line` в общий буфер |
| Индексы в горячем цикле | Bounds checks мешают векторизации | Итераторы, перенарезка, `chunks_exact` |
| Наивная float-редукция с расчётом на автовекторизацию | IEEE запрещает переупорядочивание, цикл остаётся скалярным | Несколько аккумуляторов, algebraic ops, явный SIMD |
| `target-cpu=native` в дистрибутиве | Падение с `SIGILL` на других CPU | Фиксированный уровень или мультиверсионирование |
| Расчёт на мультиверсионирование `wide` на x86 | `wide` его не умеет | `fearless_simd` или `multiversion` |
| `Box<dyn Trait>` в tight loop при закрытом наборе типов | Косвенный вызов, нет инлайнинга | Generics или enum + `match` |
| Цепочка `collect()` между стадиями | Промежуточные аллокации | Одна цепочка итераторов |
| Паутина `Rc<RefCell<_>>` | Плохая локальность, runtime-паники, сложная сериализация | Индексы или хэндлы с поколениями в арене |
| Большой редкий вариант в горячем enum | Каждое значение весит как самый большой вариант | `Box` для большого варианта, проверка `size_of` |
| `Arc<Mutex<_>>`-аккумулятор, общий для всех потоков | Контеншн съедает параллелизм | Локальное накопление и слияние |
| Слишком мелкие задачи в `par_iter` | Накладные расходы планировщика больше работы | `with_min_len`, чанки |
| Форматирование логов в горячем пути | Аллокации и форматирование, даже если уровень отключён | Проверка уровня, вынос из цикла |
| Сверхобобщённые API и тяжёлые proc-macro | Раздувание мономорфизацией, долгая компиляция | Внутренняя не-generic функция, `&dyn` на границах, урезанные фичи |
| `unsafe` (`get_unchecked`) без замера и проверки asm | Риск UB без доказанного выигрыша | Сначала безопасные приёмы, `unsafe` только с бенчмарком и `// SAFETY:` |

| Ресурс | Автор, год | Зачем читать |
|---|---|---|
| [The Rust Performance Book](https://nnethercote.github.io/perf-book/) | Nicholas Nethercote | Большинство антипаттернов и их исправлений, включая главы Linting и Logging and Debugging |
| [bounds-check-cookbook](https://github.com/Shnatsel/bounds-check-cookbook) | Shnatsel | Индексные циклы и как их переписать |
| [Fast Rust Builds](https://matklad.github.io/2021/09/04/fast-rust-builds.html) | matklad, 2021 | Антипаттерны времени компиляции |
| [The state of SIMD in Rust in 2026](https://shnatsel.github.io/state-of-simd-rust-2026/) | Shnatsel, 2026 | Ловушки SIMD: `target-cpu`, float-редукции, `wide` |
| [Rust Design Patterns](https://rust-unofficial.github.io/patterns/) | rust-unofficial | Общий раздел антипаттернов, включая clone ради borrow checker |
