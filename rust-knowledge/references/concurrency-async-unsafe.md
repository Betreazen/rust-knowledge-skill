# Конкурентность, async и unsafe в Rust

Справочник для выбора модели конкурентности, примитивов синхронизации, каналов и рантайма, для обхода ловушек async (блокировка, отмена, futurelock, backpressure) и для безопасной работы с `unsafe` и FFI. Состояние на октябрь 2026 года (стабильный Rust 1.9x, edition 2024). Главная новая ловушка последних лет — отмена futures.

- [Канонические источники](#канонические-источники)
- [Потоки или async](#потоки-или-async)
- [Send и Sync](#send-и-sync)
- [Атомики и memory ordering](#атомики-и-memory-ordering)
- [Выбор блокировки](#выбор-блокировки)
- [Каналы](#каналы)
- [Rayon и параллелизм данных](#rayon-и-параллелизм-данных)
- [Async-рантаймы](#async-рантаймы)
- [Ловушки async](#ловушки-async)
- [Структурная конкурентность](#структурная-конкурентность)
- [Паттерн актора](#паттерн-актора)
- [Статус стабилизации на октябрь 2026](#статус-стабилизации-на-октябрь-2026)
- [Unsafe: практика](#unsafe-практика)
- [FFI](#ffi)
- [Чеклист review: конкурентный и unsafe-код](#чеклист-review-конкурентный-и-unsafe-код)

## Канонические источники

**Ключевые выводы**
- Минимальный набор: Rust Atomics and Locks, Rustonomicon, туториал Tokio, два поста Alice Ryhl (акторы и блокировка), Oxide RFD 400 и 609 (отмена и futurelock), статья о Miri и заметки к релизам 1.84/1.85.
- without.boats и Niko Matsakis объясняют, почему async устроен именно так. Это чтение для понимания, а не рецепты.
- Async Book переписывается: новые главы (структурная конкурентность, pinning) полезны, старые главы 2019–2021 годов [устарело] в части трейтов и замыканий.
- Async-главы книг до 2024 года (Rust for Rustaceans и др.) хороши для модели Future/Poll/Waker/Pin, но обходные пути для трейтов и замыканий в них [устарело].

| Ресурс | Автор, год | Зачем читать |
|---|---|---|
| [Rust Atomics and Locks](https://mara.nl/atomics/) | Mara Bos, 2023 | Атомики, memory ordering, мьютексы и condvar изнутри: свои spin lock, каналы, Arc. Бесплатно онлайн |
| [The Rustonomicon](https://doc.rust-lang.org/nomicon/) | rust-lang, живой | Aliasing, variance, drop check, `PhantomData`, неинициализированная память, FFI, смысл Send/Sync |
| [Unsafe Code Guidelines](https://rust-lang.github.io/unsafe-code-guidelines/) | rust-lang UCG WG, живой | Layout и validity-инварианты |
| [Tokio tutorial](https://tokio.rs/tokio/tutorial) | Tokio team, живой | Spawn, shared state, каналы, framing, `select!`, streams |
| [Async Book](https://rust-lang.github.io/async-book/) | rust-lang, переписывается | Новые главы о структурной конкурентности и pinning |
| [Catching up with async Rust](https://fasterthanli.me/articles/catching-up-with-async-rust) | Amos Wenger, 2024 | Что дал async fn в трейтах и чего не дал: `dyn`, Send bounds |
| [Pin](https://without.boats/blog/pin/), [FuturesUnordered and the order of futures](https://without.boats/blog/futures-unordered/), [Asynchronous clean-up](https://without.boats/blog/asynchronous-clean-up/) | without.boats, 2024 | Зачем Pin, ловушки внутризадачной конкурентности, почему async drop сложен |
| [Ralf's Ramblings](https://www.ralfj.de/blog/) | Ralf Jung, живой | UB, provenance, Miri |

## Потоки или async

**Ключевые выводы**
- Async — для большого числа одновременных ожиданий ввода-вывода (тысячи соединений, fan-out запросов). Для CPU-работы async ничего не даёт.
- CPU-работа с параллелизмом данных — rayon. Несколько долгоживущих воркеров — `std::thread` или `std::thread::scope` с каналами.
- Блокирующий ввод-вывод внутри async — в `spawn_blocking`. Бесконечный блокирующий цикл — в отдельный `std::thread`.
- Встраиваемые системы без std — embassy.

| Задача | Выбор |
|---|---|
| Тысячи сетевых соединений, I/O fan-out | tokio |
| CPU-параллелизм по коллекции | rayon `par_iter` |
| Несколько воркеров с заимствованием локальных данных | `std::thread::scope` |
| Долгий блокирующий вызов из async-кода | `tokio::task::spawn_blocking` |
| Бесконечный блокирующий цикл | отдельный `std::thread` + канал |
| Embedded, `no_std` | embassy |

```rust
let mut data = vec![1, 2, 3, 4];
let (left, right) = data.split_at_mut(2);
std::thread::scope(|s| {
    s.spawn(|| left.iter_mut().for_each(|x| *x *= 2));
    s.spawn(|| right.iter_mut().for_each(|x| *x += 1));
}); // все потоки присоединены здесь, заимствования корректны
```

| Ресурс | Автор, год | Зачем читать |
|---|---|---|
| [Async: What is blocking?](https://ryhl.io/blog/async-what-is-blocking/) | Alice Ryhl, 2020 | Где граница между async, `spawn_blocking`, rayon и отдельным потоком |
| [Two Beautiful Rust Programs](https://matklad.github.io/2020/07/15/two-beautiful-programs.html) | matklad, 2020 | Scoped threads и Mutex под контролем компилятора |

## Send и Sync

**Ключевые выводы**
- `Send`: значение можно передать в другой поток. `Sync`: `&T` можно разделить между потоками (`T: Sync` тогда и только тогда, когда `&T: Send`).
- `Rc` и сырые указатели — ни `Send`, ни `Sync`. `RefCell`/`Cell` — `Send`, но не `Sync`. `MutexGuard` — не `Send`. `Arc<T>` — `Send + Sync` только при `T: Send + Sync`.
- Future является `Send`, только если `Send` всё, что живёт через `.await`. Guard, `Rc` и заимствования `RefCell` уничтожать до `.await`, ограничив их блоком.
- `tokio::spawn` требует `Send + 'static`. Для не-`Send` futures есть `LocalSet` / `spawn_local`.
- `unsafe impl Send/Sync` писать только с комментарием `// SAFETY:`, который доказывает отсутствие гонок.

```rust
async fn bump(state: &std::sync::Mutex<u64>) {
    {
        let mut g = state.lock().unwrap();
        *g += 1;
    } // guard уничтожен до .await, future остаётся Send
    tokio::task::yield_now().await;
}
```

| Ресурс | Автор, год | Зачем читать |
|---|---|---|
| [Rustonomicon: Send and Sync](https://doc.rust-lang.org/nomicon/) | rust-lang | Формальный смысл маркерных трейтов |
| [Crust of Rust](https://www.youtube.com/@jonhoo) | Jon Gjengset, 2020–2022 | Эпизоды «Send, Sync and their implementors», «Channels», «Smart Pointers and Interior Mutability» |

## Атомики и memory ordering

**Ключевые выводы**
- Начинать с `Mutex` или `SeqCst`. Ослаблять до `Acquire`/`Release` только с записанным обоснованием happens-before.
- `Relaxed` — для счётчиков и статистики, когда от значения не зависит видимость других данных.
- Публикация данных: запись `Release`, чтение `Acquire`. Read-modify-write в обеих ролях — `AcqRel`.
- Код на атомиках проверять через loom (перебор чередований) и Miri (гонки данных, эмуляция слабой памяти).
- Lock-free структуры (crossbeam-epoch, `arc-swap`) брать, только когда профилирование показало конкуренцию за lock.

| Ordering | Когда |
|---|---|
| `Relaxed` | Счётчики, метрики, генераторы ID |
| `Release` (store) / `Acquire` (load) | Флаг «данные готовы», передача владения, реализация lock |
| `AcqRel` | RMW-операции (`fetch_add`, `compare_exchange`), которые и публикуют, и потребляют |
| `SeqCst` | Нужен единый глобальный порядок для нескольких атомиков; вариант по умолчанию, если нет уверенности |

```rust
use std::sync::atomic::{AtomicBool, AtomicU64, Ordering};
static HITS: AtomicU64 = AtomicU64::new(0);
static READY: AtomicBool = AtomicBool::new(false);

HITS.fetch_add(1, Ordering::Relaxed);           // только счётчик
READY.store(true, Ordering::Release);           // всё записанное ранее становится видимым...
if READY.load(Ordering::Acquire) { /* ... */ }  // ...потоку, который увидел true
```

| Ресурс | Автор, год | Зачем читать |
|---|---|---|
| [Rust Atomics and Locks](https://mara.nl/atomics/) | Mara Bos, 2023 | Главы о memory ordering и о построении своих примитивов |
| [Crust of Rust: Atomics and Memory Ordering](https://www.youtube.com/watch?v=rMGWeSjctlY) | Jon Gjengset, ~2021 | Ошибки порядка вживую и их поиск через loom |
| [loom](https://github.com/tokio-rs/loom) | tokio-rs | Перебор чередований потоков в тестах |

## Выбор блокировки

**Ключевые выводы**
- По умолчанию `std::sync::Mutex`, в том числе в async-коде, если guard не живёт через `.await`. Критическую секцию держать короткой.
- Если guard нужен через `.await`, сначала перестроить код (скопировать данные, актор). `tokio::sync::Mutex` — крайний вариант: его `lock()` не cancel-safe, и Oxide советует не держать его через await вовсе.
- `RwLock` — только при преобладании чтений с долгими читающими секциями; следить за голоданием писателей.
- Разросшийся `Arc<Mutex<T>>` — сигнал передать состояние одной задаче (актору).
- Poisoning: после паники под lock `std::sync::Mutex` возвращает `PoisonError`. Решить явно: `unwrap()` (паника распространится) или `into_inner()` (данные считаются согласованными).

| Потребность | Выбор |
|---|---|
| Короткая секция, sync или async без `.await` под guard | `std::sync::Mutex` |
| Guard через `.await` (после попытки перестроить код) | `tokio::sync::Mutex` |
| Много читателей, долгие чтения | `std::sync::RwLock` / `parking_lot::RwLock` |
| Малый размер, без poisoning, контроль fairness | `parking_lot` |
| Конкурентная хеш-таблица | `dashmap` (шардирование) |
| Конфигурация, которую часто читают и редко целиком заменяют | `arc-swap` |
| Флаги и счётчики | атомики |
| Ленивая инициализация | `OnceLock` / `LazyLock` |

`std::sync::Mutex` на Linux построен на futex начиная с 1.62, поэтому разрыв с `parking_lot` сократился [не проверено]. Свежих независимых бенчмарков `parking_lot` против std нет.

| Ресурс | Автор, год | Зачем читать |
|---|---|---|
| [Tokio discussion #7627](https://github.com/tokio-rs/tokio/discussions/7627) | Alice Ryhl и др. | Почему std `Mutex` предпочтительнее в async |
| [Tokio tutorial: Shared state](https://tokio.rs/tokio/tutorial) | Tokio team | Тот же совет с примерами |
| [parking_lot](https://github.com/Amanieu/parking_lot), [dashmap](https://github.com/xacrimon/dashmap) | Amanieu, xacrimon | Альтернативные блокировки и конкурентная map |

## Каналы

**Ключевые выводы**
- Каналы делать ограниченными (bounded): ёмкость создаёт backpressure и не даёт памяти расти без предела.
- `std::sync::mpsc` с 1.67 работает на реализации crossbeam-channel. Совет «std mpsc медленный, всегда бери crossbeam» [устарело]. Crossbeam по-прежнему нужен для MPMC и `select!`.
- В async-коде брать каналы рантайма: блокирующий `recv` из std в задаче останавливает поток воркера.
- Ответ на запрос — `oneshot`. Последнее значение (конфигурация, состояние shutdown) — `watch`. Рассылка всем — `broadcast`.

| Канал | Модель | Когда |
|---|---|---|
| `std::sync::mpsc` (`sync_channel(n)` — bounded) | MPSC, sync | Потоки без лишних зависимостей |
| `crossbeam-channel` | MPMC, sync, `select!` | Пулы воркеров, выбор из нескольких каналов |
| `flume` | MPMC, sync + async | Мост между sync и async; крейт в режиме «casual maintenance» |
| `tokio::sync::mpsc` | MPSC, async, bounded/unbounded | Акторы, конвейеры задач |
| `tokio::sync::oneshot` | одно значение | Ответ на запрос |
| `tokio::sync::broadcast` | каждому получателю | Рассылка событий; отстающий получатель теряет сообщения |
| `tokio::sync::watch` | только последнее значение | Конфигурация, сигнал остановки |

| Ресурс | Автор, год | Зачем читать |
|---|---|---|
| [Announcing Rust 1.67.0](https://blog.rust-lang.org/2023/01/26/Rust-1.67.0/) | rust-lang, 2023 | std mpsc на основе crossbeam |
| [crossbeam](https://github.com/crossbeam-rs/crossbeam), [flume](https://github.com/zesterer/flume) | crossbeam-rs, zesterer | MPMC-каналы |

## Rayon и параллелизм данных

**Ключевые выводы**
- CPU-тяжёлую обработку коллекций распараллеливать через `par_iter`. Пул rayon по размеру равен числу ядер.
- Пул `spawn_blocking` в tokio содержит намного больше потоков, чем ядер, поэтому тяжёлым вычислениям он не подходит.
- Из async-кода CPU-работу отправлять в rayon (`rayon::spawn`), а результат получать через `tokio::sync::oneshot`.
- Мелкие задачи не распараллеливать: накладные расходы на разбиение съедают выигрыш. Измерять.

```rust
use rayon::prelude::*;

fn total_cost(items: &[u64]) -> u64 {
    items.par_iter().map(|x| x.pow(2) % 7).sum()
}
```

| Ресурс | Автор, год | Зачем читать |
|---|---|---|
| [rayon](https://github.com/rayon-rs/rayon) | rayon-rs | Параллельные итераторы, `join`, `scope` |
| [Async: What is blocking?](https://ryhl.io/blog/async-what-is-blocking/) | Alice Ryhl, 2020 | Как совмещать rayon и tokio |

## Async-рантаймы

**Ключевые выводы**
- tokio — выбор по умолчанию для сетевых сервисов: крупнейшая экосистема (hyper, axum, tonic, tokio-util).
- smol — маленький и простой рантайм, когда экосистема tokio не нужна.
- embassy — async для микроконтроллеров, `no_std`, без аллокатора.
- Библиотеки по возможности не привязывать к рантайму: принимать futures и трейты, а не вызывать `tokio::spawn` внутри.
- Не смешивать рантаймы в одном процессе без необходимости: future, использующий реактор tokio, паникует вне tokio-контекста.

| Ресурс | Автор, год | Зачем читать |
|---|---|---|
| [Tokio tutorial](https://tokio.rs/tokio/tutorial) | Tokio team | Основной рантайм |
| [smol](https://github.com/smol-rs/smol) | smol-rs | Минималистичный рантайм |
| [embassy](https://embassy.dev/) | embassy-rs | Async на embedded |

## Ловушки async

**Ключевые выводы**
- Блокировка: между `.await` не больше 10–100 мкс. Блокирующий I/O — `spawn_blocking`, CPU — rayon.
- Lock через `.await`: std guard делает future не-`Send`, tokio guard рискует дедлоком и не cancel-safe. Освобождать lock до await.
- Отмена происходит только в точках `.await`, но любой `select!`, `timeout` или drop задачи — точка отмены. Cancel safety — локальное свойство future; cancel correctness — свойство системы целиком.
- Futurelock: в `select!` по `&mut fut` не делать `.await` в обработчике другой ветки, если `fut` может владеть ресурсом (lock, место в очереди), который нужен обработчику.
- Backpressure: ограничивать каждую очередь и число одновременных задач (bounded `mpsc`, `Semaphore`, `buffer_unordered(n)`).
- Каждой внешней операции нужен таймаут (`tokio::time::timeout`).

**Блокирующий код.** Источник: [Alice Ryhl](https://ryhl.io/blog/async-what-is-blocking/).
```rust
async fn read_config(path: std::path::PathBuf) -> anyhow::Result<Vec<u8>> {
    let bytes = tokio::task::spawn_blocking(move || std::fs::read(path)).await??;
    Ok(bytes)
}
```

**Отмена в цикле `select!`.** Future, который нельзя терять, создавать и закреплять вне цикла и возобновлять, а не пересоздавать на каждой итерации. Отправку в канал делить на «reserve, потом send», чтобы отмена не теряла сообщение. Вместо `abort()` останавливать задачи явным сигналом (`CancellationToken` из tokio-util, канал). Неотменяемые API документировать. Источники: [RFD 400](https://rfd.shared.oxide.computer/rfd/0400), [tokio `select!`: Cancellation safety](https://docs.rs/tokio/latest/tokio/macro.select.html).
```rust
let work = do_work();
tokio::pin!(work);
let mut tick = tokio::time::interval(std::time::Duration::from_secs(1));
let result = loop {
    tokio::select! {
        res = &mut work => break res,          // future возобновляется, а не создаётся заново
        _ = tick.tick() => report_progress(),  // в обработчике нет .await
    }
};
```
Cancel-safe в tokio, например, `mpsc::Receiver::recv`. Не cancel-safe — `tokio::sync::Mutex::lock` (теряет место в очереди) и чтения, которые частично заполняют буфер.

**Futurelock.** Источники: [RFD 609](https://rfd.shared.oxide.computer/rfd/0609), [e6data: Deadlocking a Tokio Mutex without Holding a Lock](https://e6data.com/blog/deadlocking-tokio-mutex-without-holding-lock).
```rust
let lock = tokio::sync::Mutex::new(0);
let fut = async { let _g = lock.lock().await; /* ... */ };
tokio::pin!(fut);
tokio::select! {
    _ = &mut fut => {}
    _ = tokio::time::sleep(std::time::Duration::from_millis(10)) => {
        // fut больше не опрашивается, но держит lock или стоит за ним в очереди
        let _g = lock.lock().await; // ждёт вечно
    }
}
```
Как избежать: не делать `.await` в обработчиках `select!`, пока жив `&mut fut`; вынести `fut` в отдельную задачу (`spawn`); уничтожить `fut` до ожидания. Есть крейт [aselect](https://docs.rs/aselect/latest/aselect/) (select без async-обработчиков) и [идеи линтинга](https://farnoy.dev/posts/futurelock-linting).

**`FuturesUnordered` и внутризадачная конкурентность.** Futures внутри одной задачи могут голодать и выполняться не по порядку. Независимые единицы работы лучше запускать отдельными задачами (`JoinSet`). Источник: [without.boats](https://without.boats/blog/futures-unordered/).

| Ресурс | Автор, год | Зачем читать |
|---|---|---|
| [RFD 400](https://rfd.shared.oxide.computer/rfd/0400) | Rain (Oxide) | Систематический разбор отмены и приёмы защиты |
| [RFD 609: Futurelock](https://rfd.shared.oxide.computer/rfd/0609) | Oxide, 2025 | Разбор futurelock; [обсуждение на HN](https://news.ycombinator.com/item?id=45774086) |
| [tokio `select!`](https://docs.rs/tokio/latest/tokio/macro.select.html) | Tokio | Список cancel-safe и не cancel-safe методов |
| [FuturesUnordered and the order of futures](https://without.boats/blog/futures-unordered/) | without.boats, 2024 | Внутризадачная и межзадачная конкурентность |

## Структурная конкурентность

**Ключевые выводы**
- Не запускать задачи по принципу «fire-and-forget» через `tokio::spawn`. Группу задач держать в `JoinSet`: при его уничтожении незавершённые задачи отменяются.
- Всегда обрабатывать `JoinError`: паника задачи и её отмена приходят именно так.
- Остановку делать сигналом (`CancellationToken`, `watch`), который задачи проверяют в `select!`, а не `abort()`.
- Для синхронных потоков то же даёт `std::thread::scope`.

```rust
async fn fetch_all(urls: Vec<String>) {
    let mut set = tokio::task::JoinSet::new();
    for url in urls {
        set.spawn(async move { fetch(&url).await });
    }
    while let Some(res) = set.join_next().await {
        match res {
            Ok(body) => handle(body),
            Err(e) => eprintln!("задача упала или отменена: {e}"),
        }
    }
}
```

| Ресурс | Автор, год | Зачем читать |
|---|---|---|
| [Async Book](https://rust-lang.github.io/async-book/) | rust-lang, 2025 | Новая глава о структурной конкурентности |
| [RFD 400](https://rfd.shared.oxide.computer/rfd/0400) | Rain (Oxide) | Почему явные сигналы отмены лучше `abort` |

## Паттерн актора

**Ключевые выводы**
- Состоянием владеет одна задача с циклом `while let Some(msg) = rx.recv().await`. Снаружи доступен клонируемый хэндл с `mpsc::Sender`.
- Ответы передаются через `oneshot`, вложенный в сообщение.
- Канал ограниченный: ёмкость задаёт backpressure.
- Актор завершается сам, когда уничтожены все хэндлы (все `Sender`). Это и есть корректное завершение.
- Актор заменяет `Arc<Mutex<_>>` там, где к состоянию обращается много задач.

```rust
use tokio::sync::{mpsc, oneshot};

enum Msg { Incr, Get { reply: oneshot::Sender<u64> } }

#[derive(Clone)]
pub struct Counter { tx: mpsc::Sender<Msg> }

impl Counter {
    pub fn spawn() -> Self {
        let (tx, mut rx) = mpsc::channel(64);
        tokio::spawn(async move {
            let mut n = 0u64;
            while let Some(msg) = rx.recv().await {
                match msg {
                    Msg::Incr => n += 1,
                    Msg::Get { reply } => { let _ = reply.send(n); }
                }
            }
        });
        Self { tx }
    }
    pub async fn get(&self) -> Option<u64> {
        let (reply, rx) = oneshot::channel();
        self.tx.send(Msg::Get { reply }).await.ok()?;
        rx.await.ok()
    }
}
```

| Ресурс | Автор, год | Зачем читать |
|---|---|---|
| [Actors with Tokio](https://ryhl.io/blog/actors-with-tokio/) | Alice Ryhl, 2021 | Полный разбор: где вызывать `spawn`, backpressure, завершение |

## Статус стабилизации на октябрь 2026

**Ключевые выводы**
- Для статической диспетчеризации использовать нативный `async fn` в трейтах. `#[async_trait]` или `dynosaur` — только когда нужен `dyn Trait`.
- Пока нет Return Type Notation, `Send`-вариант трейта генерировать через `trait-variant`.
- Шаблон `F: Fn() -> Fut, Fut: Future` заменять на `F: AsyncFn()`. Callback-и вида `Box<dyn Fn() -> Pin<Box<dyn Future>>>` там, где `dyn` не нужен, — [устарело].
- Последний проверенный отчёт о целях проекта (апрель 2026) стабилизаций dyn async traits, RTN и async drop не содержит. Перед опорой на это проверять свежие отчёты.

| Возможность | Статус | Версия / замена |
|---|---|---|
| `async fn` и `-> impl Trait` в трейтах | стабильно, только статическая диспетчеризация | 1.75 |
| Async-замыкания, `AsyncFn`/`AsyncFnMut`/`AsyncFnOnce` | стабильно | 1.85 |
| Edition 2024 (`Future`/`IntoFuture` в prelude, новые правила захвата RPIT) | стабильно | 1.85 |
| Strict provenance API | стабильно | 1.84 |
| `std::thread::scope` | стабильно | 1.63 |
| `dyn` async traits | не стабильно | `async-trait`, `dynosaur`; дизайн с явным `.box` в целях 2026 |
| Return Type Notation (`T::method(..): Send`) | не стабильно | `trait-variant`; стабилизация в целях 2026 |
| Async drop | не стабильно | явный `async fn close(self)` |
| Async-итераторы и генераторы | не стабильно | `futures::Stream` |
| Эргономика `&pin` | не стабильно | `pin!`, `Pin<&mut T>`, `pin-project` |

```rust
async fn call_twice(f: impl AsyncFn(u32) -> u32) -> u32 {
    f(1).await + f(2).await
}
// call_twice(async |x| x * 2).await
```

| Ресурс | Автор, год | Зачем читать |
|---|---|---|
| [Announcing Rust 1.85.0 and Rust 2024](https://blog.rust-lang.org/2025/02/20/Rust-1.85.0/) | rust-lang, 2025 | Async-замыкания и изменения edition 2024 |
| [RFC 3185: static async fn in trait](https://rust-lang.github.io/rfcs/3185-static-async-fn-in-trait.html) | rust-lang, 2023 | Что именно стабилизировано в 1.75 |
| [RFC 3935: Project Goals 2026](https://rust-lang.github.io/rfcs/3935-Project-Goals-2026.html) | rust-lang, 2026 | Box notation для dyn async, RTN, новый trait solver |
| [Project goals update, April 2026](https://blog.rust-lang.org/2026/05/18/project-goals-2026-04/) | rust-lang, 2026 | Последний проверенный статус |
| [Dyn async traits, part 10: Box box box](https://smallcultfollowing.com/babysteps/blog/2025/03/24/box-box-box/) | Niko Matsakis, 2025 | Направление дизайна |
| [async-trait](https://github.com/dtolnay/async-trait), [dynosaur](https://docs.rs/dynosaur/latest/dynosaur/) | dtolnay; dynosaur | Обходные пути для `dyn` |

## Unsafe: практика

**Ключевые выводы**
- Unsafe держать в маленьких модулях с безопасным API, который невозможно использовать неправильно.
- Перед каждым `unsafe {}` — комментарий `// SAFETY:`, объясняющий, почему соблюдены условия. У каждой `unsafe fn` — раздел документации `# Safety` с контрактом. Проверка — Clippy `undocumented_unsafe_blocks`.
- Внутри `unsafe fn` писать явные `unsafe {}` блоки: в edition 2024 `unsafe_op_in_unsafe_fn` предупреждает по умолчанию, на старых edition включать вручную.
- Тесты с unsafe гонять под Miri в CI (`cargo +nightly miri test`), в том числе с моделью Tree Borrows (`MIRIFLAGS="-Zmiri-tree-borrows"`). Miri также ловит гонки данных и эмулирует слабую память.
- Тегировать указатели через strict provenance API (`map_addr`, `with_addr`), а не кастами int↔ptr.
- Вместо `static mut` использовать атомики, `Mutex`, `OnceLock`; если нужен адрес — `&raw const`/`&raw mut`.

**Изменения edition 2024, касающиеся unsafe**: блоки `extern` пишутся как `unsafe extern`; ссылки на `static mut` — ошибка по умолчанию (`static_mut_refs`); `unsafe_op_in_unsafe_fn` — предупреждение по умолчанию; атрибуты `no_mangle`, `export_name`, `link_section` пишутся как `#[unsafe(...)]`; `std::env::set_var` и `remove_var` стали `unsafe`.

**Tree Borrows** заменяет стек Stacked Borrows деревом. На 30 000 самых популярных крейтов эта модель отвергает на 54% меньше тестов. Код, который проходит Tree Borrows, но не Stacked Borrows, не обязательно корректен навсегда: модель aliasing официально не зафиксирована.

```rust
/// # Safety
/// `idx` должен быть меньше `slice.len()`.
pub unsafe fn get_fast(slice: &[u32], idx: usize) -> u32 {
    // SAFETY: вызывающий гарантирует idx < slice.len() (контракт выше).
    unsafe { *slice.get_unchecked(idx) }
}

fn tag_pointer() {
    let p: *mut u64 = Box::into_raw(Box::new(0u64));
    let tagged = p.map_addr(|a| a | 1);          // тег в младшем бите, provenance сохранён
    let clean = tagged.map_addr(|a| a & !1);
    // SAFETY: clean совпадает с указателем из Box::into_raw и освобождается один раз.
    drop(unsafe { Box::from_raw(clean) });
}
```

| Ресурс | Автор, год | Зачем читать |
|---|---|---|
| [Tree Borrows](https://iris-project.org/pdfs/2025-pldi-treeborrows.pdf) | Villani, Hostert, Dreyer, Jung; PLDI 2025 | Новая модель aliasing ([ACM](https://dl.acm.org/doi/10.1145/3735592)) |
| [Miri (POPL 2026)](https://research.ralfj.de/papers/2026-popl-miri.pdf) | Ralf Jung и др., 2026 | Что ловит Miri и как |
| [Announcing Rust 1.84.0](https://blog.rust-lang.org/2025/01/09/Rust-1.84.0/) | rust-lang, 2025 | Strict provenance: `addr`, `with_addr`, `map_addr`, `expose_provenance`, `without_provenance`, `dangling` |
| [Announcing Rust 1.85.0 and Rust 2024](https://blog.rust-lang.org/2025/02/20/Rust-1.85.0/) | rust-lang, 2025 | Изменения unsafe в edition 2024 |
| [loom](https://github.com/tokio-rs/loom) | tokio-rs | Перебор чередований для кода на атомиках |
| [In-place initialization](https://ryhl.io/blog/in-place-initialization/) | Alice Ryhl, 2025 | Закреплённая инициализация на месте (низкоуровневый код) |

## FFI

**Ключевые выводы**
- Объявления чужих функций — в `unsafe extern "C" { ... }` (edition 2024). Функции, безопасные для вызова, можно пометить `safe fn`.
- Типы на границе — `#[repr(C)]`. Строки — `CStr`/`CString`, не `&str`.
- Не раскручивать панику через границу FFI. Паника, выходящая из `extern "C"` функции, завершает процесс. Если раскрутка нужна намеренно — `extern "C-unwind"`; иначе ловить через `std::panic::catch_unwind` и возвращать код ошибки.
- Сырой FFI оборачивать в безопасный объектный API (хэндл-структура с `Drop`).
- Генераторы: bindgen (C → Rust), cbindgen (Rust → C-заголовки), cxx (безопасный мост с C++).

```rust
unsafe extern "C" {
    safe fn abs(x: i32) -> i32; // объявлена безопасной: вызов без unsafe
}

#[repr(C)]
pub struct Point { pub x: f64, pub y: f64 }

#[unsafe(no_mangle)]
pub extern "C" fn point_len(p: Point) -> f64 { (p.x * p.x + p.y * p.y).sqrt() }
```

| Ресурс | Автор, год | Зачем читать |
|---|---|---|
| [The Rustonomicon: FFI](https://doc.rust-lang.org/nomicon/) | rust-lang | Layout, владение через границу, unwinding |
| [Rust Design Patterns: FFI](https://rust-unofficial.github.io/patterns/) | сообщество | Ошибки и строки в FFI, объектные API-обёртки |

## Чеклист review: конкурентный и unsafe-код

1. Выбор модели обоснован: async для I/O-конкурентности, rayon или потоки для CPU.
2. В async-задачах нет блокирующих вызовов (`std::fs`, `std::thread::sleep`, тяжёлые вычисления) вне `spawn_blocking` или rayon.
3. Ни один guard (`std` или `tokio`) не живёт через `.await`; там, где future должен быть `Send`, guard и `Rc` ограничены блоком.
4. `std::sync::Mutex` выбран по умолчанию; каждый `tokio::sync::Mutex` обоснован.
5. Поведение при poisoning выбрано явно.
6. Все очереди и каналы ограничены; число одновременных задач ограничено (`Semaphore`, `buffer_unordered`).
7. У внешних вызовов есть таймауты.
8. Каждый `select!` проверен на cancel safety: потерянные при отмене ветки не теряют данные; долгоживущие futures закреплены вне цикла.
9. В обработчиках `select!` нет `.await`, пока жив `&mut fut`, владеющий lock или местом в очереди (futurelock).
10. Задачи не запускаются «fire-and-forget»: есть `JoinSet` или сохранённый `JoinHandle`, `JoinError` обработан.
11. Остановка — явный сигнал (`CancellationToken`, `watch`), а не `abort()`.
12. Разделяемое изменяемое состояние, к которому обращается много задач, вынесено в актор или передаётся сообщениями.
13. Каждый ordering, отличный от `SeqCst`, сопровождён комментарием о happens-before; `Relaxed` только для независимых счётчиков.
14. Код на атомиках и lock-free структурах покрыт тестами loom и/или Miri.
15. В async-трейтах нет `#[async_trait]` без нужды в `dyn`; `Send`-bounds решены через `trait-variant` или явные bounds.
16. Каждый `unsafe {}` блок снабжён `// SAFETY:`, у каждой `unsafe fn` есть `# Safety`.
17. Внутри `unsafe fn` операции обёрнуты в явные `unsafe {}`; `unsafe_op_in_unsafe_fn` не подавлен.
18. Каждый `unsafe impl Send/Sync` доказан комментарием.
19. Unsafe изолирован в модуле с безопасным API; инварианты нельзя нарушить из безопасного кода.
20. Тесты с unsafe проходят под Miri (включая `-Zmiri-tree-borrows`).
21. Нет int↔ptr кастов для тегирования указателей; используются `map_addr`/`with_addr`.
22. Нет ссылок на `static mut`.
23. FFI: `unsafe extern`, `#[repr(C)]`, `#[unsafe(no_mangle)]`; паника не выходит за границу (`catch_unwind` или осознанный `C-unwind`).
