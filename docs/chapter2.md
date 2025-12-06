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
  EFI_GET_MEMORY_MAP              GetMemoryMap;        // +56  (8バイト)
  EFI_ALLOCATE_POOL               AllocatePool;        // +64  (8バイト)
  EFI_FREE_POOL                   FreePool;            // +72  (8バイト)

  // Event & Timer Services
  EFI_CREATE_EVENT                CreateEvent;         // +80  (8バイト)
  // ... (他にも多数)

  // Protocol Handler Services
  // ... (約30個の関数)

  EFI_LOCATE_PROTOCOL             LocateProtocol;      // +320 (40番目)
} EFI_BOOT_SERVICES;
```

**重要**: `LocateProtocol` は **320バイト目** に配置されています。

### 3.3 Rustでの実装

```rust
#[repr(C)]
struct EfiBootServicesTable {
    _reserved0: [u64; 40],  // 最初の320バイトをスキップ (40 × 8 = 320)
    locate_protocol: extern "win64" fn(
        protocol: *const EfiGuid,
        registration: *const EfiVoid,
        interface: *mut *mut EfiVoid,
    ) -> EfiStatus,
}
const _: () = assert!(offset_of!(EfiBootServicesTable, locate_protocol) == 320);
```

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

### 4.4 2段階のポインタアクセス

```
[System Table at 0x8000_0000]
    +96バイト
    ↓
boot_services (ポインタ) → 値: 0x9000_0000
                              ↓
            [Boot Services Table at 0x9000_0000]
                +320バイト
                ↓
            locate_protocol (関数ポインタ) → 値: 0x0FFF_5678
                                              ↓
                            [関数コード at 0x0FFF_5678]
```

**アクセスの流れ:**
1. `system_table.boot_services` → System Tableの96バイト目を読む → `0x9000_0000` を取得
2. `boot_services.locate_protocol` → Boot Services Tableの320バイト目を読む → `0x0FFF_5678` を取得
3. 関数呼び出し → `0x0FFF_5678` のコードを実行

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
    // GOP取得
    let gop = locate_graphic_protocol(efi_system_table).unwrap();

    // VRAM情報を取得
    let vram_addr = gop.mode.frame_buffer_base;
    let vram_size = gop.mode.frame_buffer_size;

    // VRAMをスライスとして扱う
    let vram = unsafe {
        slice::from_raw_parts_mut(
            vram_addr as *mut u32,
            vram_size / size_of::<u32>()
        )
    };

    // 画面を白で塗りつぶす
    for pixel in vram {
        *pixel = 0xFFFFFF;
    }
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

## まとめ

### UEFIからGOPを取得するまでの流れ

```
1. UEFI ファームウェア起動
   ↓
2. efi_main(image_handle, system_table) が呼ばれる
   ↓
3. system_table.boot_services (96バイト目) を読む
   ↓
4. boot_services.locate_protocol (320バイト目) を読む
   ↓
5. locate_protocol(GOP_GUID, null, &mut gop) を呼ぶ
   ↓
6. UEFI が GOP を検索して gop に書き込む
   ↓
7. gop.mode.frame_buffer_base で VRAM アドレス取得
   ↓
8. VRAM に直接書き込んで画面表示
```

### 重要なポイント

1. **メモリレイアウト**: UEFI仕様と完全一致が必須
2. **reserved フィールド**: 使わないフィールドも正しい位置に配置
3. **ポインタチェーン**: System Table → Boot Services → 関数/プロトコル
4. **型安全性**: `const assert!` でコンパイル時検証
5. **プロトコル**: GUID で識別される機能の集合

### 次のステップ

- 文字表示（Simple Text Output Protocol）
- キーボード入力（Simple Text Input Protocol）
- ファイルアクセス（Simple File System Protocol）
- カーネルのロード（Image Protocol）

---

## 参考資料

- [UEFI Specification 2.10](https://uefi.org/specifications)
- [OSDev Wiki - UEFI](https://wiki.osdev.org/UEFI)
- [Rust UEFI Book](https://rust-osdev.github.io/uefi-rs/)