# Chapter 2: UEFI仕様とGraphics Output Protocolの取得

このドキュメントでは、UEFIの基本的な仕組みと、Graphics Output Protocol (GOP) を取得するまでのプロセスを解説します。

## 目次

1. [UEFIの基本概念](#1-uefiの基本概念)
2. [System Tableの構造](#2-system-tableの構造)
3. [Boot Services Tableの構造](#3-boot-services-tableの構造)
4. [メモリレイアウトとreservedフィールド](#4-メモリレイアウトとreservedフィールド)
5. [locate_protocol関数](#5-locate_protocol関数)
6. [Graphics Output Protocolの取得](#6-graphics-output-protocolの取得)
7. [実装のポイント](#7-実装のポイント)

---

## 1. UEFIの基本概念

### 1.1 UEFIとは

UEFI (Unified Extensible Firmware Interface) は、OSとファームウェアの間のインターフェース仕様です。従来のBIOSに代わる新しいファームウェア規格で、以下の特徴があります：

- **プロトコルベース**: 機能を「プロトコル」という単位で提供
- **拡張可能**: 新しい機能を動的に追加可能
- **標準化**: UEFI Forumが仕様を策定

### 1.2 プロトコルとは

プロトコルは、UEFIが提供する機能の単位です。各プロトコルは：

- **GUID (Globally Unique Identifier)** で識別される
- 関数ポインタとデータ構造の集合
- 実行時に検索・取得可能

例：
- Graphics Output Protocol (GOP): 画面描画
- Simple File System Protocol: ファイルアクセス
- Simple Text Input Protocol: キーボード入力

### 1.3 Boot ServicesとRuntime Services

UEFIは2種類のサービスを提供します：

| サービス | 利用可能期間 | 主な機能 |
|---------|------------|---------|
| **Boot Services** | OSブート前のみ | メモリ管理、プロトコル検索、イベント処理 |
| **Runtime Services** | OS実行中も利用可能 | 変数アクセス、時刻管理、リセット |

OSカーネルは通常、`ExitBootServices()` を呼んでBoot Servicesを終了します。

---

## 2. System Tableの構造

### 2.1 System Tableの役割

System Table は、UEFIファームウェアから渡される最も重要なデータ構造です。すべてのUEFIサービスへの入り口となります。

### 2.2 UEFI仕様でのSystem Table

UEFI仕様（C言語）では以下のように定義されています：

```c
typedef struct {
  EFI_TABLE_HEADER                  Hdr;                    // +0   (24バイト)
  CHAR16                            *FirmwareVendor;        // +24  (8バイト)
  UINT32                            FirmwareRevision;       // +32  (4バイト)
  EFI_HANDLE                        ConsoleInHandle;        // +36  (4バイト padding後 +40)
  EFI_SIMPLE_TEXT_INPUT_PROTOCOL    *ConIn;                 // +40  (8バイト)
  EFI_HANDLE                        ConsoleOutHandle;       // +48  (8バイト)
  EFI_SIMPLE_TEXT_OUTPUT_PROTOCOL   *ConOut;                // +56  (8バイト)
  EFI_HANDLE                        StandardErrorHandle;    // +64  (8バイト)
  EFI_SIMPLE_TEXT_OUTPUT_PROTOCOL   *StdErr;                // +72  (8バイト)
  EFI_RUNTIME_SERVICES               *RuntimeServices;      // +80  (8バイト)
  EFI_BOOT_SERVICES                  *BootServices;         // +88  (8バイト) → 実際は96
} EFI_SYSTEM_TABLE;
```

**重要**: `BootServices` は **96バイト目** に配置されています。

### 2.3 Rustでの実装

wasabiでは、必要な部分だけを定義しています：

```rust
#[repr(C)]
struct EfiSystemTable {
    _reserved0: [u64; 12],  // 最初の96バイトをスキップ (12 × 8 = 96)
    pub boot_services: &'static EfiBootServicesTable,  // 96バイト目
}
const _: () = assert!(offset_of!(EfiSystemTable, boot_services) == 96);
```

**ポイント:**
- `#[repr(C)]`: C言語互換のメモリレイアウト
- `_reserved0`: 使わないフィールドの場所取り
- `const assert!`: コンパイル時にオフセット検証

### 2.4 メモリレイアウト

```
[UEFIファームウェアが用意したSystem Table]
アドレス 0x8000_0000 (例)
┌────────────────────────┐
│ +0   Header (24)       │  ← Signature, Revision等
├────────────────────────┤
│ +24  FirmwareVendor    │
├────────────────────────┤
│ +32  FirmwareRevision  │
├────────────────────────┤
│ +40  ConsoleInHandle   │
├────────────────────────┤
│ +48  ConIn             │
├────────────────────────┤
│ +56  ConOut            │
├────────────────────────┤
│ +72  StdErr            │
├────────────────────────┤
│ +80  RuntimeServices   │
├────────────────────────┤
│ +96  BootServices      │ ← ここ！
└────────────────────────┘
     ↓ (ポインタ値: 例 0x9000_0000)
[Boot Services Table]
```

---

## 3. Boot Services Tableの構造

### 3.1 Boot Services Tableの役割

Boot Services Table は、UEFI起動中に利用できる機能（関数）の集合です。

### 3.2 UEFI仕様でのBoot Services Table

主要な関数ポインタ（一部）：

```c
typedef struct {
  EFI_TABLE_HEADER                Hdr;                 // +0   (24バイト)

  // Task Priority Services
  EFI_RAISE_TPL                   RaiseTPL;            // +24  (8バイト)
  EFI_RESTORE_TPL                 RestoreTPL;          // +32  (8バイト)

  // Memory Services
  EFI_ALLOCATE_PAGES              AllocatePages;       // +40  (8バイト)
  EFI_FREE_PAGES                  FreePages;           // +48  (8バイト)
  EFI_GET_MEMORY_MAP              GetMemoryMap;        // +56  (8バイト) ← 使用
  EFI_ALLOCATE_POOL               AllocatePool;        // +64  (8バイト)
  EFI_FREE_POOL                   FreePool;            // +72  (8バイト)

  // Event & Timer Services
  EFI_CREATE_EVENT                CreateEvent;         // +80  (8バイト)
  // ... (他にも多数の関数)

  // Boot Services
  EFI_EXIT_BOOT_SERVICES          ExitBootServices;    // +232 (8バイト) ← 使用

  // Protocol Handler Services
  // ... (さらに関数が続く)

  EFI_LOCATE_PROTOCOL             LocateProtocol;      // +320 (8バイト) ← 使用
} EFI_BOOT_SERVICES;
```

**重要**: wasabiで使用する3つの関数のオフセット：
- `GetMemoryMap`: **56バイト目**
- `ExitBootServices`: **232バイト目**
- `LocateProtocol`: **320バイト目**

### 3.3 Rustでの実装

wasabiで使用する3つの関数を含む完全な定義：

```rust
#[repr(C)]
struct EfiBootServicesTable {
    _reserved0: [u64; 7],              // 56バイトまでスキップ
    get_memory_map: extern "win64" fn(
        memory_map_size: *mut usize,
        memory_map: *mut u8,
        map_key: *mut usize,
        descriptor_size: *mut usize,
        descriptor_version: *mut u32,
    ) -> EfiStatus,                    // +56
    _reserved1: [u64; 21],             // 232バイトまでスキップ
    exit_boot_services: extern "win64" fn(
        image_handle: EfiHandle,
        map_key: usize,
    ) -> EfiStatus,                    // +232
    _reserved4: [u64; 10],             // 320バイトまでスキップ
    locate_protocol: extern "win64" fn(
        protocol: *const EfiGuid,
        registration: *const EfiVoid,
        interface: *mut *mut EfiVoid,
    ) -> EfiStatus,                    // +320
}

const _: () = assert!(offset_of!(EfiBootServicesTable, get_memory_map) == 56);
const _: () = assert!(offset_of!(EfiBootServicesTable, exit_boot_services) == 232);
const _: () = assert!(offset_of!(EfiBootServicesTable, locate_protocol) == 320);
```

**各関数の用途：**
- `get_memory_map` (+56): メモリマップの取得
- `exit_boot_services` (+232): ブートサービスの終了
- `locate_protocol` (+320): プロトコルの検索

### 3.4 関数ポインタの仕組み

`locate_protocol` フィールドには、関数の実体ではなく **関数のアドレス**（8バイトの整数値）が格納されています。

```
[Boot Services Table at 0x9000_0000]
┌────────────────────────────────┐
│ +0   RaiseTPL                  │ → 値: 0x0FFF_1000
│ +8   RestoreTPL                │ → 値: 0x0FFF_1100
│ +16  AllocatePages             │ → 値: 0x0FFF_1200
│ ...                            │
│ +320 LocateProtocol            │ → 値: 0x0FFF_5678
└────────────────────────────────┘
                ↓ (関数のアドレス: 0x0FFF_5678)
[UEFIファームウェアのコード領域]
┌────────────────────────────────┐
│ 0x0FFF_5678:                   │
│   48 89 5C 24 08  mov ...      │ ← 実際の機械語コード
│   48 89 74 24 10  mov ...      │
│   ...                          │
│   C3            ret            │
└────────────────────────────────┘
```

---

## 4. メモリレイアウトとreservedフィールド

### 4.1 なぜreservedが必要か

UEFIファームウェアは、メモリに**既に**データ構造を配置しています。Rustの構造体は、その既存のメモリを**解釈**するための型定義です。

**重要な前提:**
- メモリレイアウトは変更できない（UEFIが作成済み）
- Rustの構造体定義で、UEFIのレイアウトと完全一致させる必要がある
- オフセットが1バイトでもズレると、誤ったデータを読む

### 4.2 間違った実装例

```rust
// ❌ 間違い: reserved なし
#[repr(C)]
struct EfiSystemTableWrong {
    pub boot_services: &'static EfiBootServicesTable,
}

// Rustの解釈:
//   boot_services はオフセット 0 にある
// 実際のメモリ:
//   0x8000_0000 + 0 = 0x8000_0000 (Header！)
// 結果:
//   Header データを boot_services として読んでしまう → クラッシュ
```

### 4.3 正しい実装

```rust
// ✅ 正しい: reserved で位置調整
#[repr(C)]
struct EfiSystemTable {
    _reserved0: [u64; 12],  // 96バイト分スキップ
    pub boot_services: &'static EfiBootServicesTable,
}

// Rustの解釈:
//   boot_services はオフセット 96 にある
// 実際のメモリ:
//   0x8000_0000 + 96 = 0x8000_0060
// 結果:
//   正しい位置の BootServices ポインタを読む ✅
```

---

## 5. locate_protocol関数

### 5.1 関数の役割

`locate_protocol` は、指定したプロトコルを検索して、そのポインタを返す関数です。

**比喩:** 図書館で「この本(GUID)はどこですか？」と聞いて、場所(アドレス)を教えてもらう

### 5.2 関数のシグネチャ

```rust
locate_protocol: extern "win64" fn(
    protocol: *const EfiGuid,           // [IN]  検索するプロトコルのGUID
    registration: *const EfiVoid,       // [IN]  通知ハンドル（通常null）
    interface: *mut *mut EfiVoid,       // [OUT] 結果を格納する変数のアドレス
) -> EfiStatus,                         // 戻り値: 成功/失敗
```

### 5.3 各引数の詳細

#### 第1引数: protocol (何が欲しいか)

検索したいプロトコルのGUIDを渡します。

```rust
const EFI_GRAPHICS_OUTPUT_PROTOCOL_GUID: EfiGuid = EfiGuid {
    data0: 0x9042a9de,
    data1: 0x23dc,
    data2: 0x4a38,
    data3: [0x96, 0xfb, 0x7a, 0xde, 0xd0, 0x80, 0x51, 0x6a],
};
// この値はUEFI仕様で定義された、Graphics Output Protocolの識別子
```

#### 第2引数: registration (通知ハンドル)

プロトコルインストール通知を使う場合のハンドル。通常のアプリケーションでは使わないので `null` を渡します。

```rust
null_mut::<EfiVoid>()
```

#### 第3引数: interface (結果の格納先)

関数が見つけたプロトコルのポインタを書き込む場所。**出力パラメータ**です。

```rust
let mut graphic_output_protocol = null_mut::<EfiGraphicsOutputProtocol>();

locate_protocol(
    ...,
    &mut graphic_output_protocol as *mut *mut ...,
    //   ^^^                        ^^^^^^^^
    //   |                          |
    //   変数のアドレス               ポインタのポインタ型
);

// 関数実行後、graphic_output_protocol にはGOPのアドレスが入る
```

**なぜポインタのポインタ？**

関数が「変数の中身」を書き換えたいため、変数のアドレスを渡す必要があります。

```
呼び出し前:   graphic_output_protocol = null
                    ↓ (関数が書き込み)
呼び出し後:   graphic_output_protocol = 0xB000_0000 (GOPのアドレス)
```

### 5.4 内部動作

```
UEFI ファームウェア内部の処理:

1. GUID を受け取る
   ↓
2. インストール済みプロトコルのデータベースを検索
   ┌─────────────────────────────────┐
   │ UEFI Protocol Database          │
   ├─────────────────────────────────┤
   │ GUID: xxx → Disk I/O Protocol   │
   │ GUID: yyy → File System Protocol│
   │ GUID: 9042a9de → Graphics Output │ ← 発見！
   │ GUID: zzz → Network Protocol    │
   └─────────────────────────────────┘
   ↓
3. 見つかったプロトコルのアドレスを取得
   graphics_output_addr = 0xB000_0000
   ↓
4. 出力パラメータに書き込む
   *interface = graphics_output_addr
   ↓
5. 成功ステータスを返す
   return EFI_SUCCESS
```

---

## 6. Graphics Output Protocolの取得

### 6.1 全体の流れ

```rust
fn locate_graphic_protocol<'a>(
    efi_system_table: &EfiSystemTable,
) -> Result<&'a EfiGraphicsOutputProtocol<'a>> {
    // 1. 結果を受け取る変数を準備
    let mut graphic_output_protocol = null_mut::<EfiGraphicsOutputProtocol>();

    // 2. locate_protocol を呼び出し
    let status = (efi_system_table.boot_services.locate_protocol)(
        &EFI_GRAPHICS_OUTPUT_PROTOCOL_GUID,  // Graphics Output が欲しい
        null_mut::<EfiVoid>(),                // 通知は使わない
        &mut graphic_output_protocol as *mut *mut EfiGraphicsOutputProtocol as *mut *mut EfiVoid,
    );

    // 3. 成功チェック
    if status != EfiStatus::Success {
        return Err("Failed to locate graphics output protocol");
    }

    // 4. ポインタを安全に参照に変換
    Ok(unsafe { &*graphic_output_protocol })
}
```

### 6.2 ステップバイステップ

| ステップ | 処理 | 状態 |
|---------|------|------|
| **1. 初期化** | `graphic_output_protocol = null` | 空の状態 |
| **2. 呼び出し** | `locate_protocol(GUID, null, &mut ...)` | UEFI に検索依頼 |
| **3. UEFI検索** | UEFIがプロトコルDBを検索 | 内部処理 |
| **4. 書き込み** | `*interface = 0xB000_0000` | 変数に結果を格納 |
| **5. 成功判定** | `status == Success` | エラーチェック |
| **6. 使用** | `&*graphic_output_protocol` | 安全に参照化 |

### 6.3 Graphics Output Protocolの構造

```rust
#[repr(C)]
struct EfiGraphicsOutputProtocol<'a> {
    reserved: [u64; 3],  // QueryMode, SetMode, Blt 関数（使わない）
    pub mode: &'a EfiGraphicsOutputProtocolMode<'a>,
}

#[repr(C)]
struct EfiGraphicsOutputProtocolMode<'a> {
    pub max_mode: u32,
    pub mode: u32,
    pub info: &'a EfiGraphicsOutputProtocolPixelInfo,
    pub size_of_info: u64,
    pub frame_buffer_base: usize,    // ← VRAMアドレス
    pub frame_buffer_size: usize,    // ← VRAMサイズ
}
```

### 6.4 取得後の使用例

```rust
fn efi_main(_image_handle: EfiHandle, efi_system_table: &EfiSystemTable) {
    // GOP取得とVRAM初期化
    let mut vram = init_vram(efi_system_table).unwrap();

    // 画面を黒で塗りつぶす
    fill_rect(&mut vram, 0x000000, 0, 0, vram.width, vram.height).unwrap();
}
```

---

## 7. 実装のポイント

### 7.1 型安全性の確保

```rust
// コンパイル時オフセット検証
const _: () = assert!(offset_of!(EfiSystemTable, boot_services) == 96);
const _: () = assert!(offset_of!(EfiBootServicesTable, locate_protocol) == 320);
```

**効果:**
- オフセットが間違っていたらコンパイルエラー
- 実行前にバグを発見できる

### 7.2 呼び出し規約

```rust
extern "win64" fn(...)
```

UEFIはx64環境で **Microsoft x64 calling convention** を使用します。`extern "win64"` で正しい呼び出し規約を指定します。

### 7.3 メモリレイアウト

```rust
#[repr(C)]
```

C言語互換のメモリレイアウトを使用します。Rustのデフォルトレイアウトは最適化されるため、UEFI仕様と一致しません。

### 7.4 安全性とunsafe

```rust
Ok(unsafe { &*graphic_output_protocol })
```

生ポインタから参照への変換は `unsafe` が必要です。UEFIファームウェアが正しいポインタを返すことを信頼します。

---

## 8. VRAMへの直接アクセスとグラフィック描画

### 8.1 フレームバッファの概念

Graphics Output Protocol (GOP) から取得できる最も重要な情報は、**フレームバッファ（VRAM）のアドレス**です。

```rust
#[repr(C)]
struct EfiGraphicsOutputProtocolMode<'a> {
    pub max_mode: u32,
    pub mode: u32,
    pub info: &'a EfiGraphicsOutputProtocolPixelInfo,
    pub size_of_info: u64,
    pub frame_buffer_base: usize,    // ← VRAMの物理アドレス
    pub frame_buffer_size: usize,    // ← VRAMのサイズ（バイト）
}
```

**フレームバッファとは:**
- 画面に表示される各ピクセルの色情報を格納するメモリ領域
- このメモリに書き込むと、そのまま画面に反映される
- OSがない環境でも、このメモリに直接書き込むだけで画面描画が可能

### 8.2 ピクセルフォーマット

UEFIのGOPは通常、**32ビット/ピクセル** (4バイト) のBGRX形式を使用します：

```
1ピクセル = 4バイト (32ビット)
┌─────────┬─────────┬─────────┬─────────┐
│  Blue   │  Green  │   Red   │ Reserved│
│ (8bit)  │ (8bit)  │ (8bit)  │ (8bit)  │
└─────────┴─────────┴─────────┴─────────┘

例: 白色 = 0xFFFFFF = 0x00FFFFFF (Reserved=0x00)
例: 赤色 = 0xFF0000
例: 緑色 = 0x00FF00
例: 青色 = 0x0000FF
```

### 8.3 VRAMバッファの抽象化

wasabiでは、VRAMを扱いやすくするため `Bitmap` トレイトを定義しています：

```rust
trait Bitmap {
    fn bytes_per_pixel(&self) -> i64;      // 1ピクセルのバイト数（通常4）
    fn pixels_per_line(&self) -> i64;      // 1行のピクセル数
    fn width(&self) -> i64;                 // 画面幅（ピクセル）
    fn height(&self) -> i64;                // 画面高さ（ピクセル）
    fn buf_mut(&mut self) -> *mut u8;      // VRAMバッファへのポインタ

    // ピクセルへのアクセス
    unsafe fn unchecked_pixel_at_mut(&mut self, x: i64, y: i64) -> *mut u32;
    fn pixel_at_mut(&mut self, x: i64, y: i64) -> Option<&mut u32>;
}
```

**なぜ `pixels_per_line` と `width` が別々？**

実際のVRAMは、画面幅より大きい場合があります（パディング）：

```
画面: 1920x1080
実際のVRAM: 2048ピクセル/行（アライメントのため）

メモリレイアウト:
行0: [1920ピクセル（表示）][128ピクセル（未使用）]
行1: [1920ピクセル（表示）][128ピクセル（未使用）]
...
```

ピクセル (x, y) のアドレス計算：
```
address = base + (y * pixels_per_line + x) * bytes_per_pixel
```

### 8.4 VRAMへの描画

GOPから取得したVRAMアドレスに直接書き込むことで、画面表示が可能です。

**基本的な描画パターン:**
```rust
// ピクセル (x, y) のアドレス計算
address = frame_buffer_base + (y * pixels_per_line + x) * bytes_per_pixel

// そのアドレスに色データ（32ビット）を書き込む
*(address as *mut u32) = color;  // 例: 0xFFFFFF (白)
```

**VRAMの特徴:**
- メモリマップドI/O: メモリへの書き込みが直接画面に反映
- バッファリング不要: 書き込んだ瞬間に画面が変わる
- OS不要: UEFIファームウェアだけで描画可能

---

## 9. メモリマップの取得とブートサービスの終了

### 9.1 なぜメモリマップが必要か

OSカーネルを起動する際、**どのメモリ領域が使用可能か**を知る必要があります。

**メモリマップの役割:**
- システム全体のメモリ配置を把握
- UEFIファームウェアが使用中の領域を避ける
- OSが自由に使えるメモリ（CONVENTIONAL_MEMORY）を特定
- 予約済み領域（ハードウェア、ACPIテーブル等）の位置を確認

### 9.2 メモリディスクリプタの構造

UEFIは、メモリを**ページ単位**（4KiB = 4096バイト）で管理します。

```rust
#[repr(C)]
#[derive(Clone, Copy, PartialEq, Eq, Debug)]
struct EfiMemoryDescriptor {
    memory_type: EfiMemoryType,      // メモリの種類
    physical_start: u64,              // 物理アドレスの開始位置
    virtual_start: u64,               // 仮想アドレス（通常0）
    number_of_pages: u64,             // ページ数（1ページ=4KiB）
    attribute: u64,                   // メモリ属性フラグ
}
```

**フィールドの説明:**

| フィールド | 説明 | 例 |
|-----------|------|---|
| `memory_type` | メモリの用途 | CONVENTIONAL_MEMORY（使用可能） |
| `physical_start` | 開始物理アドレス | 0x0010_0000（1MiB地点） |
| `number_of_pages` | ページ数 | 256（= 1MiB） |
| `attribute` | 属性（読み書き可否等） | 0x0000_000F |

**サイズ計算:**
```
メモリ領域のサイズ = number_of_pages × 4096 バイト
```

### 11.3 メモリタイプの種類

```rust
#[repr(i64)]
pub enum EfiMemoryType {
    RESERVED = 0,                    // 予約済み
    LOADER_CODE,                     // ブートローダーコード
    LOADER_DATA,                     // ブートローダーデータ
    BOOT_SERVICES_CODE,              // UEFIブートサービスコード
    BOOT_SERVICES_DATA,              // UEFIブートサービスデータ
    RUNTIME_SERVICES_CODE,           // UEFIランタイムサービスコード
    RUNTIME_SERVICES_DATA,           // UEFIランタイムサービスデータ
    CONVENTIONAL_MEMORY,             // ← OSが自由に使える！
    UNUSABLE_MEMORY,                 // 使用不可（ハードウェアエラー等）
    ACPI_RECLAIM_MEMORY,             // ACPI終了後に再利用可能
    ACPI_MEMORY_NVS,                 // ACPI NVS（再利用不可）
    MEMORY_MAPPED_IO,                // メモリマップドI/O
    MEMORY_MAPPED_IO_PORT_SPACE,     // I/Oポート空間
    PAL_CODE,                        // プロセッサ固有コード
    PERSISTENT_MEMORY,               // 不揮発性メモリ
}
```

**重要なタイプ:**
- **CONVENTIONAL_MEMORY**: OSが自由に使える通常メモリ
- **BOOT_SERVICES_CODE/DATA**: `ExitBootServices()` 後に再利用可能
- **RUNTIME_SERVICES_CODE/DATA**: OS実行中もUEFIが使用（再利用不可）
- **ACPI_RECLAIM_MEMORY**: ACPIパース後に再利用可能

### 11.4 get_memory_map関数

#### 関数のシグネチャ

```rust
get_memory_map: extern "win64" fn(
    memory_map_size: *mut usize,        // [IN/OUT] バッファサイズ
    memory_map: *mut u8,                // [OUT] メモリマップバッファ
    map_key: *mut usize,                // [OUT] マップキー
    descriptor_size: *mut usize,        // [OUT] ディスクリプタサイズ
    descriptor_version: *mut u32,       // [OUT] ディスクリプタバージョン
) -> EfiStatus,
```

**各パラメータの役割:**

| パラメータ | 型 | 説明 |
|-----------|---|------|
| `memory_map_size` | IN/OUT | バッファサイズ（不足時は必要サイズが返る） |
| `memory_map` | OUT | メモリディスクリプタの配列を格納 |
| `map_key` | OUT | メモリマップのバージョン識別子 |
| `descriptor_size` | OUT | 各ディスクリプタのバイト数 |
| `descriptor_version` | OUT | ディスクリプタ構造のバージョン |

#### map_keyの重要性

`map_key` は**メモリマップのスナップショット識別子**です。

```
時刻 T1: get_memory_map() → map_key = 123
時刻 T2: （他の処理でメモリ状態が変化）
時刻 T3: get_memory_map() → map_key = 124（変化！）
```

**なぜ重要？**
- `ExitBootServices()` は正確なmap_keyを要求する
- メモリ状態が変わるとmap_keyも変わる
- 古いmap_keyでExitすると失敗する

### 11.5 メモリマップの管理構造

wasabiでは、メモリマップを管理する専用の構造体を定義しています：

```rust
const MEMORY_MAP_BUFFER_SIZE: usize = 0x8000;  // 32KiB

struct MemoryMapHolder {
    memory_map_buffer: [u8; MEMORY_MAP_BUFFER_SIZE],  // ディスクリプタ配列
    memory_map_size: usize,                           // 実際のサイズ
    map_key: usize,                                   // マップキー
    descriptor_size: usize,                           // ディスクリプタのサイズ
    descriptor_version: u32,                          // バージョン
}
```

**バッファサイズの決定:**
- 1ディスクリプタ ≒ 48バイト
- 32KiB ÷ 48 ≒ 682個のディスクリプタ
- 通常のシステムでは十分（実際は100〜200個程度）

#### メモリマップの反復処理

```rust
impl MemoryMapHolder {
    pub fn iter(&self) -> MemoryMapIterator {
        MemoryMapIterator { map: self, ofs: 0 }
    }
}

impl<'a> Iterator for MemoryMapIterator<'a> {
    type Item = &'a EfiMemoryDescriptor;

    fn next(&mut self) -> Option<&'a EfiMemoryDescriptor> {
        if self.ofs >= self.map.memory_map_size {
            None
        } else {
            // バッファ内の現在位置からディスクリプタを取得
            let e: &EfiMemoryDescriptor = unsafe {
                &*(self.map.memory_map_buffer.as_ptr().add(self.ofs)
                   as *const EfiMemoryDescriptor)
            };
            self.ofs += self.map.descriptor_size;  // 次のディスクリプタへ
            Some(e)
        }
    }
}
```

**仕組み:**
1. バッファの先頭から `descriptor_size` バイトずつ読む
2. 各チャンクを `EfiMemoryDescriptor` として解釈
3. `memory_map_size` に達するまで繰り返す

### 11.6 メモリマップの取得と利用

```rust
fn efi_main(image_handle: EfiHandle, efi_system_table: &EfiSystemTable) {
    let mut vram = init_vram(efi_system_table).unwrap();
    let mut w = VramTextWriter::new(&mut vram);

    // メモリマップを取得
    let mut memory_map = MemoryMapHolder::new();
    let status = efi_system_table.boot_services.get_memory_map(&mut memory_map);
    writeln!(w, "{status:?}").unwrap();  // "Success"

    // 使用可能メモリを集計
    let mut total_memory_pages = 0;
    for e in memory_map.iter() {
        if e.memory_type != EfiMemoryType::CONVENTIONAL_MEMORY {
            continue;
        }
        total_memory_pages += e.number_of_pages;
        writeln!(w, "{e:?}").unwrap();  // ディスクリプタを表示
    }

    // MiB単位で表示
    let total_memory_size_mib = total_memory_pages * 4096 / 1024 / 1024;
    writeln!(w, "Total: {total_memory_pages} pages = {total_memory_size_mib} MiB").unwrap();
}
```

**出力例:**
```
Success
EfiMemoryDescriptor { memory_type: CONVENTIONAL_MEMORY, physical_start: 0x100000, ... }
EfiMemoryDescriptor { memory_type: CONVENTIONAL_MEMORY, physical_start: 0x1000000, ... }
...
Total: 262144 pages = 1024 MiB
```

---

## 12. ExitBootServices - UEFIからOSへの移行

### 12.1 ExitBootServicesの役割

`ExitBootServices()` は、**UEFIブートサービスを終了し、OSに制御を渡す**重要な関数です。

**実行前:**
- Boot Servicesが利用可能（メモリ割り当て、プロトコル検索等）
- UEFIファームウェアがメモリやハードウェアを管理

**実行後:**
- Boot Servicesは全て無効化（関数呼び出し不可）
- OSが完全にハードウェアを制御
- Runtime Servicesのみ利用可能（変数アクセス、リセット等）

### 12.2 関数のシグネチャ

```rust
exit_boot_services: extern "win64" fn(
    image_handle: EfiHandle,    // [IN] アプリケーションハンドル
    map_key: usize,             // [IN] 現在のメモリマップキー
) -> EfiStatus,
```

**パラメータ:**
- `image_handle`: `efi_main` に渡されたハンドル
- `map_key`: `get_memory_map` で取得したmap_key

### 12.3 なぜmap_keyが必要か

UEFIは、**メモリ状態が変わっていないこと**を確認してからExitします。

```
シナリオ1: 成功
1. get_memory_map() → map_key = 100
2. （メモリ状態変化なし）
3. exit_boot_services(handle, 100) → Success ✅

シナリオ2: 失敗
1. get_memory_map() → map_key = 100
2. （UEFIがメモリを再配置）→ map_key内部的に101に
3. exit_boot_services(handle, 100) → Invalid Parameter ❌
```

**理由:**
- メモリマップが古いと、OSが誤った領域にアクセスする危険
- UEFIは最新のmap_keyでのみExitを許可
  - Exit後はUEFI管理からOS管理になるため、Exitが成功した後はメモリマップは変わらない
  - `EfiRuntimeServicesCode`と`EfiRuntimeServicesData`はOSが触っちゃいけない

### 12.4 リトライループの実装

`ExitBootServices` は**失敗する可能性が高い**ため、リトライが必須です。

```rust
fn exit_from_efi_boot_services(
    image_handle: EfiHandle,
    efi_system_table: &EfiSystemTable,
    memory_map: &mut MemoryMapHolder,
) {
    loop {
        // 1. 最新のメモリマップを取得
        let status = efi_system_table.boot_services.get_memory_map(memory_map);
        assert_eq!(status, EfiStatus::Success);

        // 2. Exitを試みる
        let status = (efi_system_table.boot_services.exit_boot_services)(
            image_handle,
            memory_map.map_key,  // 最新のmap_key
        );

        // 3. 成功したらループを抜ける
        if status == EfiStatus::Success {
            break;
        }

        // 4. 失敗したら再度メモリマップを取得してリトライ
    }
}
```

**ループの流れ:**

```
┌─────────────────────────────────┐
│ 1. get_memory_map()             │
│    → map_key = 100              │
├─────────────────────────────────┤
│ 2. exit_boot_services(100)      │
│    → Invalid Parameter ❌       │ ← メモリ状態が変わった
├─────────────────────────────────┤
│ 3. get_memory_map()（再取得）    │
│    → map_key = 101              │
├─────────────────────────────────┤
│ 4. exit_boot_services(101)      │
│    → Success ✅                 │
└─────────────────────────────────┘
```

### 12.5 なぜ失敗するのか

`get_memory_map()` と `exit_boot_services()` の間に、UEFIが内部でメモリを変更する可能性があります。

**失敗の原因例:**
- 割り込み処理でメモリ割り当て
- ファームウェアのタイマー処理
- デバイスドライバの動作
- 他のEFIアプリケーションの動作

**解決策:**
- リトライループで最新のmap_keyを取得し続ける
- 通常は1〜2回のリトライで成功

### 12.6 ExitBootServices後の制約

#### 使えなくなる機能

| 機能 | 状態 |
|------|------|
| **メモリ割り当て** | ❌ 不可（AllocatePages, AllocatePool） |
| **プロトコル検索** | ❌ 不可（LocateProtocol） |
| **イベント/タイマー** | ❌ 不可 |
| **コンソールI/O** | ❌ 不可（ConOut, ConIn） |
| **Graphics Output Protocol** | ❌ 関数は使えない（VRAMは直接アクセス可） |

#### 引き続き使える機能

| 機能 | 状態 |
|------|------|
| **VRAM** | ✅ 直接アクセス可能（取得済みアドレス） |
| **変数アクセス** | ✅ Runtime Services経由 |
| **時刻取得** | ✅ Runtime Services経由 |
| **システムリセット** | ✅ Runtime Services経由 |

### 12.7 実際の使用例

wasabiの実装では、以下のような流れでExitBootServicesを実行しています：

```rust
fn efi_main(image_handle: EfiHandle, efi_system_table: &EfiSystemTable) {
    // 1. Graphics Output Protocolを取得（Exit前に必須）
    let mut vram = init_vram(efi_system_table).expect("init_vram failed");
    fill_rect(&mut vram, 0x000000, 0, 0, vram.width, vram.height).unwrap();
    draw_test_pattern(&mut vram);

    let mut w = VramTextWriter::new(&mut vram);
    for i in 0..4 {
        writeln!(w, "i={i}").unwrap();
    }

    // 2. メモリマップを取得して情報を表示
    let mut memory_map = MemoryMapHolder::new();
    let status = efi_system_table.boot_services.get_memory_map(&mut memory_map);
    writeln!(w, "{status:?}").unwrap();  // "Success"

    // 使用可能メモリを集計して画面に表示
    let mut total_memory_pages = 0;
    for e in memory_map.iter() {
        if e.memory_type != EfiMemoryType::CONVENTIONAL_MEMORY {
            continue;
        }
        total_memory_pages += e.number_of_pages;
        writeln!(w, "{e:?}").unwrap();
    }
    let total_memory_size_mib = total_memory_pages * 4096 / 1024 / 1024;
    writeln!(w, "Total: {total_memory_pages} pages = {total_memory_size_mib} MiB").unwrap();

    // 3. ブートサービスを終了
    exit_from_efi_boot_services(image_handle, efi_system_table, &mut memory_map);

    // ここからはOS環境（Boot Services使用不可）
    writeln!(w, "Hello, Non-UEFI world!").unwrap();  // VRAMは引き続き使える

    loop { hlt(); }
}
```

**実行時の画面表示:**
```
i=0
i=1
i=2
i=3
Success
EfiMemoryDescriptor { memory_type: CONVENTIONAL_MEMORY, physical_start: 0x100000, ... }
EfiMemoryDescriptor { memory_type: CONVENTIONAL_MEMORY, physical_start: 0x1000000, ... }
...
Total: 262144 pages = 1024 MiB
Hello, Non-UEFI world!
```

この例では、ExitBootServices前にメモリマップの情報を画面に表示してから、OSモードに移行しています。

### 12.8 Boot Services Tableのオフセット

```rust
#[repr(C)]
struct EfiBootServicesTable {
    _reserved0: [u64; 7],              // 56バイトまでスキップ
    get_memory_map: extern "win64" fn(...),  // +56
    _reserved1: [u64; 21],             // 232バイトまでスキップ
    exit_boot_services: extern "win64" fn(...),  // +232
    _reserved4: [u64; 10],             // 320バイトまでスキップ
    locate_protocol: extern "win64" fn(...),  // +320
}

const _: () = assert!(offset_of!(EfiBootServicesTable, get_memory_map) == 56);
const _: () = assert!(offset_of!(EfiBootServicesTable, exit_boot_services) == 232);
const _: () = assert!(offset_of!(EfiBootServicesTable, locate_protocol) == 320);
```

---

## 13. まとめ：UEFI環境からOSカーネルへ

### 13.1 全体の流れ

```
1. UEFI ファームウェア起動
   ↓
2. efi_main() が呼ばれる
   ├─ image_handle: アプリケーション識別子
   └─ system_table: UEFIサービスへのポインタ
   ↓
3. Graphics Output Protocol を取得
   ├─ locate_protocol(GOP_GUID) でGOPを検索
   └─ VRAM アドレスを取得（Exit後も使用）
   ↓
4. メモリマップを取得
   ├─ get_memory_map() でシステムメモリ情報を取得
   ├─ CONVENTIONAL_MEMORY を集計（OS使用可能領域）
   └─ map_key を保存（ExitBootServices用）
   ↓
5. UEFIブートサービスを終了
   ├─ リトライループで exit_boot_services() を呼ぶ
   ├─ 成功するまで get_memory_map() を再取得
   └─ Boot Services 完全無効化
   ↓
6. OS環境に移行
   ├─ VRAMへの直接アクセスは継続
   ├─ Runtime Services のみ利用可能
   └─ カーネルが完全にハードウェアを制御
```

### 13.2 重要なポイント

| トピック | 重要事項 |
|---------|---------|
| **メモリマップ** | OSが使える領域を把握するため必須 |
| **map_key** | メモリ状態の一貫性を保証する識別子 |
| **リトライループ** | ExitBootServicesは失敗する可能性が高い |
| **Exit前の準備** | 必要なプロトコルは全てExit前に取得 |
| **VRAM継続利用** | GOPは無効化されるがVRAMは直接使える |

### 13.3 次のステップ

- **ページングの有効化**: 仮想メモリ管理
- **割り込みハンドラ**: IDT（Interrupt Descriptor Table）の設定
- **カーネルヒープ**: 動的メモリアロケータの実装
- **タスク管理**: マルチタスク機能の実装
- **デバイスドライバ**: キーボード、ディスク等のドライバ実装

---

## 参考資料

- [UEFI Specification 2.10](https://uefi.org/specifications)
  - 12.9: Graphics Output Protocol
  - 11: Protocols - Console Support
- [OSDev Wiki - UEFI](https://wiki.osdev.org/UEFI)
- [OSDev Wiki - Drawing In a Linear Framebuffer](https://wiki.osdev.org/Drawing_In_a_Linear_Framebuffer)
- [Rust UEFI Book](https://rust-osdev.github.io/uefi-rs/)
