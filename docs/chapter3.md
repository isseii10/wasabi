# Chapter 3: ページング、割り込み、メモリアロケータ

このドキュメントでは、chapter3で実装した主要な機能について解説します。

## 目次

1. [x86_64ページングの実装](#x86_64ページングの実装)
2. [割り込み処理の実装](#割り込み処理の実装)
3. [メモリアロケータの実装](#メモリアロケータの実装)

---

## x86_64ページングの実装

### ページング機構の概要

x86_64アーキテクチャでは、**4レベルページテーブル**を使用して仮想アドレスを物理アドレスに変換します。

#### ページテーブルの階層構造

```
仮想アドレス (48ビット使用)
┌────────┬────────┬────────┬────────┬─────────────┐
│ PML4   │ PDPT   │   PD   │   PT   │   Offset    │
│ (9bit) │ (9bit) │ (9bit) │ (9bit) │   (12bit)   │
│ 47-39  │ 38-30  │ 29-21  │ 20-12  │    11-0     │
└────────┴────────┴────────┴────────┴─────────────┘
```

**各レベルで使用するビット範囲とSHIFT値:**

| テーブル | LEVEL | 使用ビット | SHIFT値 |
|---------|-------|-----------|---------|
| PML4    | 4     | bits 47-39 | 39 |
| PDPT    | 3     | bits 38-30 | 30 |
| PD      | 2     | bits 29-21 | 21 |
| PT      | 1     | bits 20-12 | 12 |

### Rustでの実装

```rust
#[repr(align(4096))]
pub struct Table<const LEVEL: usize, const SHIFT: usize, NEXT> {
    entry: [Entry<LEVEL, SHIFT, NEXT>; 512],
}

pub type PT = Table<1, 12, [u8; PAGE_SIZE]>;
pub type PD = Table<2, 21, PT>;
pub type PDPT = Table<3, 30, PD>;
pub type PML4 = Table<4, 39, PDPT>;

fn calc_index(&self, addr: u64) -> usize {
    ((addr >> SHIFT) & 0b1_1111_1111) as usize
}
```

### ページマッピングの実装

```rust
impl PML4 {
    pub fn create_mapping(
        &mut self,
        virt_start: u64,
        virt_end: u64,
        phys: u64,
        attr: PageAttr,
    ) -> Result<()> {
        // 4重ループでPML4→PDPT→PD→PTを辿る
        // ensure_populated()で必要なテーブルを自動作成
        // 連続したページをマッピング
    }
}
```

---

## 割り込み処理の実装

### IDT (Interrupt Descriptor Table)

各割り込み番号に対応するハンドラのアドレスを格納するテーブルです。

```rust
#[repr(C, packed)]
pub struct IdtDescriptor {
    offset_low: u16,
    segment_selector: u16,
    ist_index: u8,          // IST (Interrupt Stack Table) インデックス
    attr: IdtAttr,
    offset_mid: u16,
    offset_high: u32,
    _reserved: u32,
}
```

### IST (Interrupt Stack Table)

特定の例外用に専用スタックを確保する仕組みです。

**なぜ必要か:**
- Double Fault (#8) 等の重大な例外では、スタックが壊れている可能性がある
- 専用スタックを使うことで、スタック破損に関係なく確実にハンドラを実行できる

```rust
entries[8] = IdtDescriptor::new(
    segment_selector,
    2,  // IST[2]を使用（Double Fault専用スタック）
    IdtAttr::IntGateDPL0,
    interrupt_entrypoint8,
);
```

### 割り込みハンドラのエントリポイント

マクロで各割り込み番号のエントリポイントを生成：

```rust
macro_rules! interrupt_entrypoint {
    ($index:literal) => {
        global_asm!(concat!(
            ".global interrupt_entrypoint", stringify!($index), "\n",
            "interrupt_entrypoint", stringify!($index), ":\n",
            "push 0 // No error code\n",
            "push rcx\n",
            "mov rcx, ", stringify!($index), "\n",
            "jmp inthandler_common"
        ));
    };
}
```

---

## メモリアロケータの実装

### Header構造体のメモリレイアウト

### 構造体定義

```rust
struct Header {
    next_header: Option<Box<Header>>,  // 8バイト
    size: usize,                        // 8バイト
    is_allocated: bool,                 // 1バイト
    _reserved: usize,                   // 8バイト
}
```

### メモリ上での配置（32バイト）

```
+------------------+  ← Header開始アドレス (例: 0x1000)
| next_header      |  オフセット +0  (8バイト)
| (ポインタ)       |  次のHeaderへのポインタ or None
+------------------+  オフセット +8
| size             |  (8バイト)
|                  |  このチャンクのサイズ（Header含む）
+------------------+  オフセット +16
| is_allocated     |  (1バイト)
|                  |  true: 割り当て済み, false: 空き
+------------------+  オフセット +17
| (padding)        |  (7バイト) - アライメント調整
+------------------+  オフセット +24
| _reserved        |  (8バイト)
|                  |  予約領域（将来の拡張用）
+------------------+  オフセット +32 ← Header終了
| データ領域...    |
```

**重要なポイント:**
- Header全体は32バイト（2の累乗）
- `next_header`は`Option<Box<Header>>`型で8バイト
  - `Some(ptr)`: ポインタ値
  - `None`: 0x00（nullポインタ）
- `bool`型は1バイトだが、次のフィールドとのアライメントのため7バイトのパディングが入る
- `_reserved`フィールドで32バイトに調整

## フリーリストの構造

### 初期状態

UEFI Memory Mapから取得した空きメモリ領域から構築：

```
first_header
    ↓
+--------+------------------------+
| Header | Free Region 1          |  size: 0x100000 (1MB)
| @0x... | (is_allocated = false) |
+--------+------------------------+
    | next_header
    ↓
+--------+------------------------+
| Header | Free Region 2          |  size: 0x200000 (2MB)
| @0x... | (is_allocated = false) |
+--------+------------------------+
    | next_header
    ↓
  None
```

### メモリチャンクの構造

各メモリチャンクは以下の形式：

```
        チャンクの開始
           ↓
+----------+---------------------------+
| Header   | データ領域                |
| (32byte) | (size - 32 byte)          |
+----------+---------------------------+
           ↑                           ↑
      ユーザーが                  end_addr()
      受け取る                  (Header開始 + size)
      アドレス
```

## メモリ割り当ての仕組み（後方割り当て）

### 割り当て前の状態

要求: 64バイトのメモリ、アライメント32バイト

```
元のフリーチャンク:
+--------+----------------------------------------+
| Header | Free Region                           | size = 0x1000 (4096)
| @0x... |                                        | is_allocated = false
+--------+----------------------------------------+
0x1000   0x1020                                   0x2000
```

### 割り当て後の状態

**重要:** 割り当ては**後方**から行われる！

```
+--------+------------------+--------+---------+--------+-------+
| Header | Shrunk Free      | Header | Alloc'd | Header | Pad   |
| @0x... | Region           | @0x... | Region  | @0x... | Region|
+--------+------------------+--------+---------+--------+-------+
0x1000   0x1020             0x1F80   0x1FA0    0x1FE0   0x2000
                            (aligned)          (padding)

元のチャンク: size = 0x1000 → 縮小: size = 0xF60
新規割り当てチャンク: size = 0x60 (32 + 64)
パディングチャンク: size = 0x20 (末尾の余り)
```

### 詳細な割り当てプロセス

#### ステップ1: 割り当て領域の計算

```rust
// 要求サイズを2の累乗に切り上げ（最小32バイト）
let size = max(round_up_to_nearest_pow2(64).unwrap(), 32); // → 64

// アライメントも最小32バイト
let align = max(32, 32); // → 32

// 後方からアライメント調整して配置
let allocated_addr = (self.end_addr() - size) & !(align - 1);
// (0x2000 - 64) & !31 = 0x1FC0 & 0xFFFFFFE0 = 0x1FA0
```

#### ステップ2: 割り当て用Headerの作成

```
                                    allocated_addr
                                         ↓
+--------+------------------+--------+-------+
| Header | Free Region      | Header | Alloc |
|        |                  | NEW!   | 64B   |
+--------+------------------+--------+-------+
                            0x1F80   0x1FA0  0x1FE0
                            ↑
                       header_for_allocated
                       (allocated_addr - 32)
```

```rust
// 割り当てHeaderをallocated_addrの直前に配置
let header_for_allocated = Header::new_from_addr(0x1FA0 - 32); // 0x1F80
header_for_allocated.is_allocated = true;
header_for_allocated.size = 64 + 32; // 96 (0x60)
```

#### ステップ3: パディングHeaderの作成（必要な場合）

末尾に余りがある場合、パディング用のHeaderを作成：

```
                                              padding領域
                                              ↓
+--------+---------------+--------+-------+--------+----+
| Header | Free          | Header | Alloc | Header | Pad|
|        |               |        | 64B   | NEW!   | 0B |
+--------+---------------+--------+-------+--------+----+
                         0x1F80   0x1FA0  0x1FE0   0x2000
                                          ↑
                                    header_for_padding
                                    (0x1FE0 = 0x1F80 + 0x60)
```

```rust
if header_for_allocated.end_addr() != self.end_addr() {
    // パディングHeader作成
    let header_for_padding = Header::new_from_addr(0x1FE0);
    header_for_padding.is_allocated = false;
    header_for_padding.size = 0x2000 - 0x1FE0; // 32 (0x20)

    // リンクリストに挿入
    header_for_padding.next_header = header_for_allocated.next_header;
    header_for_allocated.next_header = Some(header_for_padding);
}
```

#### ステップ4: 元のフリーチャンクの縮小

```rust
// 元のチャンクのサイズを縮小
self.size -= (header_for_allocated.size + header_for_padding.size);
// 0x1000 - (0x60 + 0x20) = 0xF80

// リンクリストを更新
self.next_header = Some(header_for_allocated);
```

### 最終的なフリーリスト構造

```
first_header
    ↓
+--------+------------------+
| Header | Free Region      |  size: 0xF80
| @0x... | is_allocated=F   |
+--------+------------------+
    | next_header
    ↓
+--------+---------+
| Header | Alloc'd |  size: 0x60 ← ユーザーが使用中
| @0x... | is_all=T|
+--------+---------+
    | next_header
    ↓
+--------+-------+
| Header | Pad   |  size: 0x20
| @0x... | is_a=F|  (将来的に結合可能)
+--------+-------+
    | next_header
    ↓
   None
```

## 後方割り当ての利点

1. **アライメント調整が容易**
   - 末尾から逆算することで、アライメント境界に正確に配置できる
   - `(end_addr - size) & !(align - 1)` で一発計算

2. **フラグメンテーション削減**
   - パディングが末尾に集約される
   - 小さなパディング領域が散在しにくい

3. **実装の簡潔性**
   - 元のチャンクのサイズを減らすだけで済む
   - 前方割り当てだと複雑なポインタ操作が必要

## メモリ解放の仕組み

### 解放処理

```rust
unsafe fn dealloc(&self, ptr: *mut u8, _layout: Layout) {
    // ユーザーポインタからHeaderを逆算
    let mut region = Header::from_allocated_region(ptr);

    // 割り当てフラグをクリア
    region.is_allocated = false;

    // Headerをメモリ上にリーク（Drop防止）
    Box::leak(region);
}
```

### 解放前

```
+--------+-------+--------+---------+
| Header | Free  | Header | Alloc'd | ← dealloc(ptr)
|        |       |        | is_a=T  |
+--------+-------+--------+---------+
                 0x1F80   ptr=0x1FA0
```

### 解放後

```
+--------+-------+--------+---------+
| Header | Free  | Header | Free    | ← is_allocated = false
|        |       |        | is_a=F  |
+--------+-------+--------+---------+
                          ↑
                   次回の割り当てで再利用可能
```

**注意点:**
- 現在の実装では隣接する空きチャンクの**結合（coalesce）は行われない**
- 将来的な最適化として、解放時に隣接チャンクを結合する処理を追加可能

## サイズとアライメントの調整

### アライメントとは

**アライメント (Alignment)** とは、メモリアドレスが特定の倍数になるように配置する制約のことです。

#### なぜアライメントが必要か

1. **CPU命令の要求**
   - 多くのCPU命令は特定のアライメントを要求する
   - 例: x86_64の`movdqa`命令は16バイトアライメント必須

2. **パフォーマンス**
   - アライメントされていないアクセスは遅い（複数回のメモリアクセスが必要）
   - キャッシュライン境界をまたぐと性能が低下

3. **ハードウェア制約**
   - 一部のアーキテクチャではアライメント違反で例外が発生
   - ページ境界をまたぐと複数ページアクセスが必要（TLBミス増加）

#### データ型とアライメント

Rustの各型には**自然なアライメント**があります：

**基本型:**
```rust
u8   → align=1  (どのアドレスでもOK)
u16  → align=2  (2の倍数アドレス)
u32  → align=4  (4の倍数アドレス)
u64  → align=8  (8の倍数アドレス)
u128 → align=8  (x86_64では8、AArch64では16)
```

**複合型:**
```rust
// 配列 - 要素のアライメントを継承
[u8; 100]  → align=1
[u64; 10]  → align=8

// 構造体 - 最大メンバーのアライメント
struct Foo {
    a: u8,   // align=1
    b: u64,  // align=8
}
// → Foo全体のalign=8（最大メンバー）

// Wasabi Header
struct Header {
    next_header: Option<Box<Header>>,  // 8バイト, align=8
    size: usize,                        // 8バイト, align=8
    is_allocated: bool,                 // 1バイト, align=1
    _reserved: usize,                   // 8バイト, align=8
}
// → Header全体のalign=8（usize/ポインタのアライメント）
```

#### ページ境界とアライメント

アライメントが不十分だと**ページ境界をまたぐ**問題が発生：

```
align=8の場合（u128で16バイト確保）:
ページ境界
  ↓
  |...0xFF8|0x000|0x008|  ← 16バイトがページをまたぐ
  |Page 0  |  Page 1   |

結果:
- 2回のメモリアクセスが必要
- TLBミスのリスク増加
- キャッシュライン跨ぎの可能性
```

align=16にすれば問題を回避：
```
align=16の場合:
  |...0xFF0|███████████|  ← ページ内に収まる
  |Page 0  |  Page 1   |
```

#### Layout構造体のalignの決定

`alloc::alloc::Layout`のalignは**呼び出し側**が決定します：

**1. コンパイラの自動決定（型から）:**
```rust
let x = Box::new(123u64);  // コンパイラがLayout::new::<u64>()を生成
// → align=8 (u64の自然なアライメント)
```

**2. プログラマの明示的指定:**
```rust
let layout = Layout::from_size_align(1234, 64).unwrap();
// → align=64 (明示的に指定)
```

**3. 定数定義（システム要求）:**
```rust
pub const LAYOUT_PAGE_4K: Layout =
    unsafe { Layout::from_size_align_unchecked(4096, 4096) };
// → align=4096 (ページ境界アライメント)
```

allocator.rsのテストでは様々なアライメントを検証：
```rust
for align in [1, 2, 4, 8, 16, 32, 4096] {
    let layout = Layout::from_size_align(1234, align).unwrap();
    let ptr = ALLOCATOR.alloc_with_options(layout);
    assert!(ptr as usize % align == 0);  // アライメント検証
}
```

### サイズの2の累乗への切り上げ

```rust
pub fn round_up_to_nearest_pow2(v: usize) -> Result<usize> {
    1usize
        .checked_shl(usize::BITS - v.wrapping_sub(1).leading_zeros())
        .ok_or("Out of range")
}
```

**例:**
- 入力: 60 → 出力: 64 (2^6)
- 入力: 100 → 出力: 128 (2^7)
- 入力: 1000 → 出力: 1024 (2^10)

**理由:** 2の累乗サイズにすることで、アライメント調整が簡単になり、フラグメンテーションも削減される

### アライメント調整の計算

```rust
let allocated_addr = (end_addr - size) & !(align - 1);
```

**ビット演算の詳細:**

alignが32 (0b100000) の場合:
```
align - 1 = 31 = 0b011111
!(align - 1)   = 0b...11100000  (下位5ビットがマスク)

例: end_addr = 0x2000, size = 64
  0x2000 - 64 = 0x1FC0 = 0b...111111000000
  0x1FC0 & 0b...11100000 = 0x1FA0 = 0b...111110100000
                                        ↑↑↑↑↑
                                       下位5bitが0（32バイト境界）
```

## 初期化プロセス

### UEFI Memory Mapからの構築

```rust
pub fn init_with_mmap(&self, memory_map: &MemoryMapHolder) {
    for desc in memory_map.iter() {
        if desc.memory_type() == EfiMemoryType::CONVENTIONAL_MEMORY {
            self.add_free_from_descriptor(desc);
        }
    }
}
```

### メモリディスクリプタからの追加

```
UEFI Memory Map:
+-------------------+
| Descriptor 1      | Type: CONVENTIONAL_MEMORY
| start: 0x100000   | pages: 256 (1MB)
+-------------------+
| Descriptor 2      | Type: RESERVED
| start: 0x200000   | (スキップ)
+-------------------+
| Descriptor 3      | Type: CONVENTIONAL_MEMORY
| start: 0x300000   | pages: 512 (2MB)
+-------------------+

↓ init_with_mmap()

フリーリスト:
+--------+--------+    +--------+---------+
| Header | 1MB    | -> | Header | 2MB     | -> None
| @0x... | Free   |    | @0x... | Free    |
+--------+--------+    +--------+---------+
```

### アドレス0の回避

```rust
// アドレス0を含む場合は除外
if start_addr == 0 {
    start_addr += 4096;
    size = size.saturating_sub(4096);
}
```

**理由:** nullポインタ(0x0)との区別のため、アドレス0は使用しない

## メモリアロケータの特徴まとめ

### 設計の特徴

1. **First Fit方式**
   - フリーリストを先頭から探索
   - 最初に見つかった十分なサイズのチャンクを使用

2. **後方割り当て**
   - チャンクの末尾から割り当て
   - アライメント調整が容易

3. **Header埋め込み**
   - データ領域の直前にHeaderを配置
   - ポインタからO(1)でHeader取得可能

4. **2の累乗サイズ**
   - サイズとアライメントを2の累乗に調整
   - フラグメンテーション削減

### 制限事項

- **結合なし**: 隣接する空きチャンクの自動結合は未実装
- **ソートなし**: フリーリストはアドレス順でない
- **統計なし**: 使用量や断片化の追跡機能なし

### 将来の改善案

1. **空きチャンクの結合**
   ```rust
   // 解放時に隣接チャンクをチェック
   fn merge_adjacent_free_chunks(&mut self) { ... }
   ```

2. **Best Fit方式**
   ```rust
   // 最小の十分なチャンクを選択
   fn find_best_fit(&self, size: usize) -> Option<&mut Header> { ... }
   ```

3. **ビンベースの管理**
   ```rust
   // サイズ別にフリーリストを分類
   struct BinnedAllocator {
       small_bins: [Option<Box<Header>>; 64],
       large_bins: [Option<Box<Header>>; 32],
   }
   ```

## まとめ

Wasabiのメモリアロケータは、シンプルながら効率的な設計：

- ✅ **32バイトHeader**: 2の累乗サイズで効率的
- ✅ **後方割り当て**: アライメント調整が容易
- ✅ **First Fit**: 実装がシンプル
- ✅ **UEFI統合**: Memory Mapから自動構築

この実装は、OSカーネル開発の基礎として十分な機能を提供しつつ、将来的な拡張の余地も残しています。

---

## 参考資料

- [Intel® 64 and IA-32 Architectures Software Developer's Manual](https://www.intel.com/content/www/us/en/developer/articles/technical/intel-sdm.html)
  - Volume 3A: 4章 ページング
  - Volume 3A: 6章 割り込みと例外処理
- [OSDev Wiki - Paging](https://wiki.osdev.org/Paging)
- [OSDev Wiki - Interrupt Descriptor Table](https://wiki.osdev.org/Interrupt_Descriptor_Table)
- [OSDev Wiki - Memory Allocation](https://wiki.osdev.org/Memory_Allocation)
