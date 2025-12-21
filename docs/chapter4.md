# Chapter 4: ACPI、HPET、非同期実行基盤

このドキュメントでは、chapter4で実装した主要な機能について解説します。

## 目次

1. [ACPI (Advanced Configuration and Power Interface)](#acpi-advanced-configuration-and-power-interface)
2. [HPET (High Precision Event Timer)](#hpet-high-precision-event-timer)
3. [非同期実行基盤 (Executor, Task, Future)](#非同期実行基盤-executor-task-future)
4. [ミューテックス (Mutex)](#ミューテックス-mutex)

---

## ACPI (Advanced Configuration and Power Interface)

### ACPIとは

**ACPI** は、ハードウェアの構成情報や電源管理機能を提供する標準規格です。OSは ACPI テーブルを解析することで、システムに搭載されているハードウェアを発見できます。

### ACPIテーブルの階層構造

```
RSDP (Root System Description Pointer)
  ↓
XSDT (Extended System Description Table)
  ↓
  ├─ HPET (High Precision Event Timer)
  ├─ MCFG (Memory Mapped Configuration)
  ├─ FADT (Fixed ACPI Description Table)
  └─ その他のテーブル...
```

### RSDP (Root System Description Pointer)

UEFI 環境では、System Table の Configuration Table から ACPI の RSDP を取得できます。

```rust
#[repr(C)]
#[derive(Debug)]
pub struct AcpiRsdpStruct {
    signature: [u8; 8],      // "RSD PTR "
    checksum: u8,
    oem_id: [u8; 6],
    revision: u8,
    rsdt_address: u32,       // ACPI 1.0 (32ビット)
    length: u32,
    xsdt: u64,               // ACPI 2.0+ (64ビット)
}
```

**RSDP の役割:**
- XSDT のアドレスを保持
- ACPI テーブル階層のエントリーポイント

### XSDT (Extended System Description Table)

XSDT は、各種 ACPI テーブルへのポインタを格納した配列です。

```rust
#[repr(packed)]
struct Xsdt {
    header: SystemDescriptionTableHeader,  // 36バイト
    // この後に可変長の 64ビットポインタ配列が続く
}
```

**エントリ数の計算:**
```rust
fn number_of_entries(&self) -> usize {
    (self.header.length as usize - self.header_size()) / size_of::<*const u8>()
}
```

- `header.length`: XSDT全体のサイズ（ヘッダー + エントリ配列）
- `header_size()`: ヘッダーのサイズ (36バイト)
- エントリサイズ: 8バイト (64ビットポインタ)

### System Description Table Header

すべての ACPI テーブルは共通のヘッダーを持ちます。

```rust
#[repr(packed)]
#[derive(Clone, Copy, Debug)]
struct SystemDescriptionTableHeader {
    signature: [u8; 4],      // テーブルの種類 (例: "HPET")
    length: u32,             // テーブル全体の長さ
    _unused: [u8; 28],       // revision, checksum, OEM ID等
}
```

**シグネチャでテーブルを識別:**
- `"HPET"`: High Precision Event Timer
- `"MCFG"`: PCI Express Memory Mapped Configuration
- `"FADT"`: Fixed ACPI Description Table
- `"APIC"`: Multiple APIC Description Table

### XSDTの探索

XSDT 内のテーブルを走査してシグネチャで検索：

```rust
impl Xsdt {
    fn find_table(&self, sig: &'static [u8; 4]) -> Option<&'static SystemDescriptionTableHeader> {
        self.iter().find(|&e| e.signature() == sig)
    }

    unsafe fn entry(&self, index: usize) -> *const u8 {
        ((self as *const Self as *const u8).add(self.header_size()) as *const *const u8)
            .add(index)
            .read_unaligned()
    }
}
```

**メモリレイアウト:**
```
+------------------------+
| SystemDescriptionTable |
| Header (36 bytes)      |
+------------------------+  ← self.header_size()
| Entry[0]: u64 ptr      |  ← entry(0)
+------------------------+
| Entry[1]: u64 ptr      |  ← entry(1)
+------------------------+
| Entry[2]: u64 ptr      |  ← entry(2)
+------------------------+
| ...                    |
```

### Generic Address Structure

ACPI では、ハードウェアレジスタのアドレスを表現するために Generic Address Structure を使用します。

```rust
#[repr(packed)]
pub struct GenericAddress {
    address_space_id: u8,    // 0: System Memory, 1: System I/O, etc.
    _unused: [u8; 3],
    address: u64,            // 物理アドレス
}
```

**アドレス空間の種類:**
- `0`: System Memory (MMIO)
- `1`: System I/O (Port I/O)
- `2`: PCI Configuration Space
- その他: SMBus, CMOS, etc.

```rust
impl GenericAddress {
    pub fn address_in_memory_space(&self) -> Result<usize> {
        if self.address_space_id == 0 {
            Ok(self.address as usize)
        } else {
            Err("ACPI Generic Address is not in system memory space")
        }
    }
}
```

### HPET Descriptor

HPET テーブルには、HPET レジスタのベースアドレスが含まれています。

```rust
#[repr(packed)]
pub struct AcpiHpetDescriptor {
    _header: SystemDescriptionTableHeader,  // 36バイト
    _reserved0: u32,                         // 4バイト
    address: GenericAddress,                 // 12バイト
    _reserved1: u32,                         // 4バイト
}
```

**サイズ検証:**
```rust
const _: () = assert!(size_of::<AcpiHpetDescriptor>() == 56);
```

**HPET レジスタへのアクセス:**
```rust
impl AcpiHpetDescriptor {
    pub fn base_address(&self) -> Result<&'static mut HpetRegisters> {
        unsafe {
            self.address
                .address_in_memory_space()
                .map(|addr| &mut *(addr as *mut HpetRegisters))
        }
    }
}
```

### AcpiTableトレイト

各 ACPI テーブルに共通のインターフェースを提供：

```rust
trait AcpiTable {
    const SIGNATURE: &'static [u8; 4];
    type Table;

    fn new(header: &SystemDescriptionTableHeader) -> &Self::Table {
        header.expect_signature(Self::SIGNATURE);
        unsafe { &*(header as *const SystemDescriptionTableHeader as *const Self::Table) }
    }
}

impl AcpiTable for AcpiHpetDescriptor {
    const SIGNATURE: &'static [u8; 4] = b"HPET";
    type Table = Self;
}
```

### ACPI使用例

```rust
// UEFI System TableからACPI RSDPを取得
let acpi = efi_system_table.acpi_table().expect("ACPI table not found");

// XSDTからHPETテーブルを検索
let xsdt = acpi.xsdt();
let hpet_desc = xsdt.find_table(b"HPET").map(AcpiHpetDescriptor::new);

// HPETレジスタのベースアドレスを取得
let hpet_registers = hpet_desc.base_address()?;
```

---

## HPET (High Precision Event Timer)

### HPETとは

**HPET** は、高精度のハードウェアタイマーです。従来の PIT (Programmable Interval Timer) や RTC (Real-Time Clock) と比較して、より高精度な時刻測定が可能です。

**主な特徴:**
- **64ビットカウンタ**: オーバーフローまでの時間が長い
- **フェムト秒精度**: カウンタのインクリメント周期がフェムト秒単位で定義
- **複数のタイマー**: 最大32個の独立したタイマーを持つ

### HPETレジスタの構造

```rust
#[repr(C)]
pub struct HpetRegisters {
    capabilities_and_id: u64,       // オフセット 0x00
    _reserved0: u64,                 // オフセット 0x08
    configuration: u64,              // オフセット 0x10
    _reserved1: [u64; 27],           // オフセット 0x18-0xE8
    main_counter_value: u64,         // オフセット 0xF0
    _reserved2: u64,                 // オフセット 0xF8
    timers: [TimerRegister; 32],     // オフセット 0x100-0x4FF
}
```

**サイズ検証:**
```rust
const _: () = assert!(size_of::<HpetRegisters>() == 0x500);
```

### Capabilities and ID Register (オフセット 0x00)

このレジスタには HPET の能力情報が含まれます。

```
63          32  31        16 15   13 12         8  7        0
+-------------+------------+------+-------------+----------+
|   PERIOD    | RESERVED   | LEG  | NUM_TIM_CAP | REV_ID   |
+-------------+------------+------+-------------+----------+
```

**フィールドの意味:**
- `PERIOD (bits 63-32)`: カウンタ周期 (フェムト秒)
- `NUM_TIM_CAP (bits 12-8)`: タイマー数 - 1 (実際のタイマー数は +1)
- `REV_ID (bits 7-0)`: リビジョンID

**使用例:**
```rust
let fs_per_count = registers.capabilities_and_id >> 32;
let num_of_timers = ((registers.capabilities_and_id >> 8) & 0b11111) as usize + 1;
```

### 周波数の計算

```rust
let freq = 1_000_000_000_000_000 / fs_per_count;
```

**例:**
- `fs_per_count = 10,000,000` (10^7 fs) の場合
- `freq = 10^15 / 10^7 = 10^8 = 100 MHz`

### Configuration Register (オフセット 0x10)

HPET の動作を制御するレジスタです。

```
63      2  1  0
+------+--+--+
| RES  |LP|EN|
+------+--+--+
```

**ビットフィールド:**
- `ENABLE_CNF (bit 0)`: 1 で HPET 有効、0 で無効
- `LEG_RT_CNF (bit 1)`: レガシー置換モード

```rust
unsafe fn globally_enable(&mut self) {
    let config = read_volatile(&self.registers.configuration) | 0b01;
    write_volatile(&mut self.registers.configuration, config);
}

unsafe fn globally_disable(&mut self) {
    let config = read_volatile(&self.registers.configuration) & !0b11;
    write_volatile(&mut self.registers.configuration, config);
}
```

### Main Counter Value (オフセット 0xF0)

64ビットの自由走行カウンタです。HPET が有効化されると、周期的にインクリメントされます。

```rust
pub fn main_counter(&self) -> u64 {
    unsafe { read_volatile(&self.registers.main_counter_value) }
}
```

### Timer Register

各タイマーは専用のレジスタセットを持ちます。

```rust
#[repr(C)]
struct TimerRegister {
    configuration_and_capability: u64,  // オフセット +0x00
    _reserved: [u64; 3],                 // オフセット +0x08-0x18
}
const _: () = assert!(size_of::<TimerRegister>() == 0x20);
```

**Timer Configuration Bits:**
```rust
const TIMER_CONFIG_LEVEL_TRIGGER: u64 = 1 << 1;      // レベルトリガ
const TIMER_CONFIG_INT_ENABLE: u64 = 1 << 2;         // 割り込み有効
const TIMER_CONFIG_USE_PERIODIC_MODE: u64 = 1 << 3;  // 周期モード
```

### HPET初期化プロセス

```rust
impl Hpet {
    pub fn new(registers: &'static mut HpetRegisters) -> Self {
        // 1. Capabilities から情報を取得
        let fs_per_count = registers.capabilities_and_id >> 32;
        let num_of_timers = ((registers.capabilities_and_id >> 8) & 0b11111) as usize + 1;
        let freq = 1_000_000_000_000_000 / fs_per_count;

        let mut hpet = Self {
            registers,
            num_of_timers,
            freq,
        };

        unsafe {
            // 2. HPET を一時停止
            hpet.globally_disable();

            // 3. すべてのタイマーを無効化
            for i in 0..hpet.num_of_timers {
                let timer = &mut hpet.registers.timers[i];
                let mut config = read_volatile(&timer.configuration_and_capability);
                config &= !(TIMER_CONFIG_INT_ENABLE
                    | TIMER_CONFIG_USE_PERIODIC_MODE
                    | TIMER_CONFIG_LEVEL_TRIGGER
                    | (0b11111 << 9));  // IRQ routing をクリア
                timer.write_config(config);
            }

            // 4. メインカウンタをリセット
            write_volatile(&mut hpet.registers.main_counter_value, 0);

            // 5. HPET を有効化
            hpet.globally_enable();
        }

        hpet
    }
}
```

### タイムスタンプの取得

```rust
pub fn global_timestamp() -> Duration {
    HPET.lock()
        .as_ref()
        .map(|hpet| {
            let count = hpet.main_counter();
            let freq = hpet.freq();
            Duration::from_nanos((count * 1_000_000_000 / freq) as u64)
        })
        .unwrap_or(Duration::ZERO)
}
```

**計算例:**
- カウント値: 100,000,000
- 周波数: 100 MHz (10^8 Hz)
- 経過時間: 100,000,000 / 100,000,000 = 1秒

---

## 非同期実行基盤 (Executor, Task, Future)

### Rust の非同期プログラミング

Rust の `async/await` は、Future トレイトに基づいています。

```rust
pub trait Future {
    type Output;
    fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output>;
}

pub enum Poll<T> {
    Ready(T),    // 完了
    Pending,     // 未完了（再度 poll が必要）
}
```

### Future の状態遷移

```
    async fn が呼ばれる
           ↓
    Future オブジェクト生成
           ↓
    poll() ───→ Poll::Pending ──┐
      ↑                          │
      └──────────────────────────┘
           ↓
    Poll::Ready(result)
           ↓
         完了
```

### Task 構造体

Future を実行単位としてラップします。

```rust
pub struct Task<T> {
    future: Pin<Box<dyn Future<Output = Result<T>>>>,
    created_at_file: &'static str,
    created_at_line: u32,
}
```

**#[track_caller] によるデバッグ情報:**
```rust
#[track_caller]
pub fn new(future: impl Future<Output = Result<T>> + 'static) -> Task<T> {
    Task {
        future: Box::pin(future),
        created_at_file: Location::caller().file(),
        created_at_line: Location::caller().line(),
    }
}
```

これにより、タスクがどこで生成されたか追跡できます：
```
Task(src/main.rs:45)
```

### Waker の役割

通常の非同期ランタイムでは、Waker を使って Future の再実行をスケジュールします。

```
Future::poll(cx) → Poll::Pending
         ↓
    cx.waker().wake()  ← イベント発生時に呼ばれる
         ↓
    再度 poll される
```

### No-op Waker

現在の実装では、シンプルな polling ベースのため、Waker は何もしません。

```rust
fn no_op_raw_waker() -> RawWaker {
    fn no_op(_: *const ()) {}
    fn clone(_: *const ()) -> RawWaker {
        no_op_raw_waker()
    }
    let vtable = &RawWakerVTable::new(clone, no_op, no_op, no_op);
    RawWaker::new(null::<()>(), vtable)
}

pub fn no_op_waker() -> Waker {
    unsafe { Waker::from_raw(no_op_raw_waker()) }
}
```

### block_on: ブロッキング実行

単一の Future を完了まで実行：

```rust
pub fn block_on<T>(future: impl Future<Output = Result<T>> + 'static) -> Result<T> {
    let mut task = Task::new(future);
    loop {
        let waker = no_op_waker();
        let mut context = Context::from_waker(&waker);
        match task.poll(&mut context) {
            Poll::Ready(result) => break result,
            Poll::Pending => busy_loop_hint(),  // CPU にヒントを与えて待機
        }
    }
}
```

### Executor: 複数タスクの実行

```rust
pub struct Executor {
    task_queue: Option<VecDeque<Task<()>>>,
}
```

**ラウンドロビン方式:**
```rust
pub fn run(mut executor: Self) -> ! {
    info!("Executor starts running...");
    loop {
        let task = executor.task_queue().pop_front();
        if let Some(mut task) = task {
            let waker = no_op_waker();
            let mut context = Context::from_waker(&waker);
            match task.poll(&mut context) {
                Poll::Ready(result) => {
                    info!("Task completed: {:?}:{:?}", task, result);
                }
                Poll::Pending => {
                    executor.task_queue().push_back(task);  // キューの末尾に戻す
                }
            }
        }
    }
}
```

**実行フロー:**
```
キュー: [Task1, Task2, Task3]
    ↓
Task1.poll() → Pending
    ↓
キュー: [Task2, Task3, Task1]
    ↓
Task2.poll() → Pending
    ↓
キュー: [Task3, Task1, Task2]
    ↓
Task3.poll() → Ready(Ok(()))
    ↓
キュー: [Task1, Task2]
    ↓
繰り返し...
```

### Yield: 協調的マルチタスク

他のタスクに実行権を譲るための Future：

```rust
#[derive(Default)]
pub struct Yield {
    polled: AtomicBool,
}

impl Future for Yield {
    type Output = ();
    fn poll(self: Pin<&mut Self>, _: &mut Context<'_>) -> Poll<()> {
        if self.polled.fetch_or(true, Ordering::SeqCst) {
            Poll::Ready(())   // 2回目の poll で完了
        } else {
            Poll::Pending     // 1回目の poll は Pending
        }
    }
}

pub async fn yield_execution() {
    Yield::default().await
}
```

**使用例:**
```rust
async fn long_running_task() {
    for i in 0..1000 {
        do_work(i);
        yield_execution().await;  // 他のタスクに譲る
    }
}
```

### TimeoutFuture: 時間ベースの待機

```rust
pub struct TimeoutFuture {
    time_out: Duration,
}

impl TimeoutFuture {
    pub fn new(duration: Duration) -> Self {
        Self {
            time_out: global_timestamp() + duration,
        }
    }
}

impl Future for TimeoutFuture {
    type Output = ();
    fn poll(self: Pin<&mut Self>, _: &mut Context) -> Poll<()> {
        if self.time_out < global_timestamp() {
            Poll::Ready(())
        } else {
            Poll::Pending
        }
    }
}
```

**使用例:**
```rust
async fn periodic_task() {
    loop {
        info!("Tick!");
        TimeoutFuture::new(Duration::from_secs(1)).await;  // 1秒待機
    }
}
```

### 実行例

```rust
let task1 = Task::new(async move {
    for i in 100..=103 {
        info!("{i} timestamp = {:?}", global_timestamp());
        TimeoutFuture::new(Duration::from_secs(1)).await;
    }
    Ok(())
});

let task2 = Task::new(async move {
    for i in 200..=203 {
        info!("{i} timestamp = {:?}", global_timestamp());
        TimeoutFuture::new(Duration::from_secs(2)).await;
    }
    Ok(())
});

let mut executor = Executor::new();
executor.enqueue(task1);
executor.enqueue(task2);
Executor::run(executor)
```

**出力例:**
```
100 timestamp = 0ns
200 timestamp = 0ns
100 timestamp = 1000ms
100 timestamp = 2000ms
200 timestamp = 2000ms
100 timestamp = 3000ms
200 timestamp = 4000ms
...
```

---

## ミューテックス (Mutex)

### 排他制御の必要性

複数のタスクが同じデータにアクセスする場合、データ競合を防ぐために排他制御が必要です。

```rust
// 排他制御なし（データ競合の危険）
static mut COUNTER: u32 = 0;
unsafe { COUNTER += 1; }  // 複数タスクから同時実行で壊れる

// Mutexによる排他制御
static COUNTER: Mutex<u32> = Mutex::new(0);
*COUNTER.lock() += 1;  // 安全
```

### Mutexの構造

```rust
pub struct Mutex<T> {
    data: SyncUnsafeCell<T>,        // 保護対象のデータ
    is_taken: AtomicBool,           // ロック状態
    taker_line_num: AtomicU32,      // デバッグ用：ロック取得行番号
    created_at_file: &'static str,  // デバッグ用：生成元ファイル
    created_at_line: u32,            // デバッグ用：生成元行番号
}
```

### SyncUnsafeCell

`SyncUnsafeCell` は、排他制御を実装するための基本的なビルディングブロックです。

```rust
// 通常の不変参照からでも可変ポインタを取得できる
let cell: &SyncUnsafeCell<T> = ...;
let ptr: *mut T = cell.get();  // &self から *mut T を取得
```

**注意:** ポインタをデリファレンスする前に、アクセスが排他的であることを保証する必要があります。

### AtomicBool によるロック

```rust
#[track_caller]
fn try_lock(&self) -> Result<MutexGuard<T>> {
    if self.is_taken
        .compare_exchange(false, true, Ordering::SeqCst, Ordering::SeqCst)
        .is_ok()
    {
        // ロック成功
        self.taker_line_num
            .store(Location::caller().line(), Ordering::SeqCst);
        Ok(unsafe { MutexGuard::new(self, &self.data) })
    } else {
        // ロック失敗（既に他のタスクが保持中）
        Err("Lock failed")
    }
}
```

**compare_exchange の動作:**
```
is_taken が false の場合:
  → true にセット → ロック成功
is_taken が true の場合:
  → 変更せず → ロック失敗
```

### MutexGuard: RAII によるロック解放

```rust
pub struct MutexGuard<'a, T> {
    mutex: &'a Mutex<T>,
    data: &'a mut T,
    location: Location<'a>,
}

impl<'a, T> Drop for MutexGuard<'a, T> {
    fn drop(&mut self) {
        self.mutex.is_taken.store(false, Ordering::SeqCst)  // ロック解放
    }
}
```

**RAII (Resource Acquisition Is Initialization) パターン:**
- ロック取得 = MutexGuard 生成
- ロック解放 = MutexGuard の Drop

```rust
{
    let mut guard = MUTEX.lock();  // ロック取得
    *guard += 1;
    // ...
}  // ← スコープ終了で自動的にロック解放
```

### Deref/DerefMut トレイト

MutexGuard は透過的にデータへアクセスできます。

```rust
impl<'a, T> Deref for MutexGuard<'a, T> {
    type Target = T;
    fn deref(&self) -> &Self::Target {
        self.data
    }
}

impl<'a, T> DerefMut for MutexGuard<'a, T> {
    fn deref_mut(&mut self) -> &mut Self::Target {
        self.data
    }
}
```

**使用例:**
```rust
let mut guard = COUNTER.lock();
*guard += 1;  // DerefMut により *guard が &mut u32 として扱える
```

### スピンロック: リトライによるロック取得

```rust
#[track_caller]
pub fn lock(&self) -> MutexGuard<T> {
    for _ in 0..100000 {
        if let Ok(locked) = self.try_lock() {
            return locked;
        }
    }
    panic!(
        "Failed to lock Mutex at {}:{}, caller: {:?}, taker_line_num: {}",
        self.created_at_file,
        self.created_at_line,
        Location::caller(),
        self.taker_line_num.load(Ordering::SeqCst)
    )
}
```

**スピンロックの特徴:**
- ✅ シンプルな実装
- ✅ ロック保持時間が短い場合に効率的
- ❌ CPU サイクルを浪費（他のタスクが実行できない）
- ❌ 優先度逆転の問題

### under_locked: 関数型インターフェース

```rust
pub fn under_locked<R: Sized>(&self, f: &dyn Fn(&mut T) -> Result<R>) -> Result<R> {
    let mut locked = self.lock();
    f(&mut *locked)
}
```

**使用例:**
```rust
HPET.under_locked(&|hpet| {
    hpet.reset_counter();
    Ok(())
})?;
```

### グローバル Mutex の使用例

```rust
use crate::mutex::Mutex;

static HPET: Mutex<Option<Hpet>> = Mutex::new(None);

pub fn init_hpet(acpi: &AcpiRsdpStruct) {
    let hpet_desc = acpi.hpet().expect("HPET not found");
    let registers = hpet_desc.base_address().expect("Invalid HPET address");
    let hpet = Hpet::new(registers);

    *HPET.lock() = Some(hpet);
}

pub fn global_timestamp() -> Duration {
    HPET.lock()
        .as_ref()
        .map(|hpet| {
            let count = hpet.main_counter();
            let freq = hpet.freq();
            Duration::from_nanos((count * 1_000_000_000 / freq) as u64)
        })
        .unwrap_or(Duration::ZERO)
}
```

---

## まとめ

Chapter 4 では、OS カーネルに不可欠な基盤機能を実装しました：

### 実装した機能

1. **ACPI サポート**
   - ✅ RSDP/XSDT のパース
   - ✅ Generic Address Structure
   - ✅ HPET テーブルの取得

2. **HPET タイマー**
   - ✅ 高精度タイムスタンプ
   - ✅ ナノ秒精度の時刻測定
   - ✅ グローバルカウンタへのアクセス

3. **非同期実行基盤**
   - ✅ Future ベースのタスクシステム
   - ✅ Executor によるマルチタスク
   - ✅ TimeoutFuture による時間待機
   - ✅ yield による協調的スケジューリング

4. **ミューテックス**
   - ✅ スピンロック方式の排他制御
   - ✅ RAII によるロック管理
   - ✅ デバッグ情報の追跡

### 今後の拡張

- **割り込みベースの Executor**: タイマー割り込みで非同期タスクをスケジュール
- **プリエンプティブ・マルチタスク**: 強制的なコンテキストスイッチ
- **複数 CPU コアのサポート**: SMP (Symmetric Multi-Processing)
- **優先度ベースのスケジューリング**: リアルタイム性の向上

---

## 参考資料

- [ACPI Specification](https://uefi.org/specifications)
- [Intel® 64 and IA-32 Architectures Software Developer's Manual](https://www.intel.com/content/www/us/en/developer/articles/technical/intel-sdm.html)
  - Volume 3B: 14章 HPET
- [OSDev Wiki - HPET](https://wiki.osdev.org/HPET)
- [The Rust Async Book](https://rust-lang.github.io/async-book/)
