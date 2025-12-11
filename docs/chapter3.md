# Chapter 3: メモリアロケータの実装

## Header構造体のメモリレイアウト

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