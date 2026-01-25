# Chapter 6: USBデバイスドライバの実装

本章では、chapter5で実装したxHCIコントローラの基盤の上に、USBデバイスドライバを実装していく。具体的には、USBデバイスディスクリプタの読み取り、キーボードドライバ、およびタブレット（マウス）ドライバの実装を行う。

## 6.1 USB Descriptorの読み取り

### 6.1.1 Device Descriptor

USBデバイスに接続すると、まずDevice Descriptorを読み取ってデバイスの基本情報を取得する。

```rust
#[repr(packed)]
pub struct UsbDeviceDescriptor {
    pub desc_length: u8,      // ディスクリプタのサイズ（18バイト）
    pub desc_type: u8,        // ディスクリプタタイプ（1 = Device）
    pub version: u16,         // USBバージョン（BCD形式）
    pub device_class: u8,     // デバイスクラス
    pub device_subclass: u8,  // デバイスサブクラス
    pub device_protocol: u8,  // デバイスプロトコル
    pub max_packet_size: u8,  // エンドポイント0の最大パケットサイズ
    pub vendor_id: u16,       // ベンダーID
    pub product_id: u16,      // プロダクトID
    pub device_version: u16,  // デバイスバージョン（BCD形式）
    pub manufacturer_idx: u8, // 製造者文字列のインデックス
    pub product_idx: u8,      // 製品名文字列のインデックス
    pub serial_idx: u8,       // シリアル番号文字列のインデックス
    pub num_of_config: u8,    // 設定の数
}
```

Device Descriptorの取得は、Control Transfer（Setup Stage → Data Stage → Status Stage）を使用する。

### 6.1.2 Configuration/Interface/Endpoint Descriptor

デバイスの設定情報を取得するために、Configuration Descriptorとそれに続くInterface/Endpoint Descriptorを読み取る。

**Configuration Descriptor:**
```rust
#[repr(packed)]
pub struct ConfigDescriptor {
    desc_length: u8,         // ディスクリプタのサイズ（9バイト）
    desc_type: u8,           // ディスクリプタタイプ（2 = Config）
    total_length: u16,       // 設定全体の長さ（後続のディスクリプタを含む）
    num_of_interfaces: u8,   // インターフェースの数
    config_value: u8,        // 設定値（SET_CONFIGURATIONで使用）
    config_string_index: u8, // 設定名文字列のインデックス
    attribute: u8,           // 属性（自己電源/リモートウェイクアップ）
    max_power: u8,           // 最大消費電力（2mA単位）
}
```

**Interface Descriptor:**
```rust
#[repr(packed)]
pub struct InterfaceDescriptor {
    desc_length: u8,         // ディスクリプタのサイズ（9バイト）
    desc_type: u8,           // ディスクリプタタイプ（4 = Interface）
    pub interface_number: u8, // インターフェース番号
    pub alt_setting: u8,     // 代替設定
    num_of_endpoints: u8,    // エンドポイントの数
    interface_class: u8,     // インターフェースクラス
    interface_subclass: u8,  // インターフェースサブクラス
    interface_protocol: u8,  // インターフェースプロトコル
    interface_index: u8,     // インターフェース名文字列のインデックス
}
```

**Endpoint Descriptor:**
```rust
#[repr(packed)]
pub struct EndpointDescriptor {
    pub desc_length: u8,
    pub desc_type: u8,
    pub endpoint_address: u8,  // bit[0..=3]: エンドポイント番号, bit[7]: 方向
    pub attributes: u8,        // bit[0..=1]: 転送タイプ
    pub max_packet_size: u16,  // 最大パケットサイズ
    pub interval: u8,          // ポーリング間隔
}
```

### 6.1.3 ディスクリプタのイテレータ

Configuration Descriptorの`total_length`フィールドを使って、後続のディスクリプタを効率的にパースするイテレータを実装した：

```rust
pub struct DescriptorIterator<'a> {
    buf: &'a [u8],
    index: usize,
}

impl<'a> Iterator for DescriptorIterator<'a> {
    type Item = UsbDescriptor;
    fn next(&mut self) -> Option<Self::Item> {
        // desc_length と desc_type を読み取り、適切な構造体に変換
    }
}
```

## 6.2 Sliceableトレイト

バイト列からディスクリプタ構造体への変換を安全に行うため、`Sliceable`トレイトを導入した：

```rust
/// # Safety
/// バイト列との相互変換が安全な型（所有権や参照を持たない型）にのみ実装可能
pub unsafe trait Sliceable: Sized + Copy + Clone {
    fn as_slice(&self) -> &[u8] {
        unsafe {
            slice::from_raw_parts(
                self as *const Self as *const u8,
                size_of::<Self>()
            )
        }
    }
    fn copy_from_slice(data: &[u8]) -> Result<Self> {
        if size_of::<Self>() > data.len() {
            Err("data is too short")
        } else {
            Ok(unsafe { *(data.as_ptr() as *const Self) })
        }
    }
}
```

これにより、UEFIファームウェアから取得したバイト列を安全にRust構造体に変換できる。

## 6.3 USBキーボードドライバ

### 6.3.1 HIDクラスとBoot Protocol

USBキーボードはHID（Human Interface Device）クラスに属する。キーボードのインターフェーストリプルは `(3, 1, 1)`:
- クラス: 3 (HID)
- サブクラス: 1 (Boot Interface)
- プロトコル: 1 (Keyboard)

Boot Protocolを使用すると、デバイスはシンプルな8バイトのレポート形式でキー入力を報告する。

### 6.3.2 キーイベントの変換

```rust
pub enum KeyEvent {
    None,
    Char(char),
    Unknown(u8),
    Enter,
}

impl KeyEvent {
    pub fn from_usb_key_id(usage_id: u8) -> Self {
        match usage_id {
            0 => KeyEvent::None,
            4..=29 => KeyEvent::Char((b'a' + usage_id - 4) as char),  // a-z
            30..=39 => KeyEvent::Char((b'0' + (usage_id + 1) % 10) as char), // 0-9
            40 => KeyEvent::Enter,
            42 => KeyEvent::Char(0x08 as char), // Backspace
            44 => KeyEvent::Char(' '),
            // ... その他のキー
            _ => KeyEvent::Unknown(usage_id),
        }
    }
}
```

### 6.3.3 キーボード初期化シーケンス

1. **SET_CONFIGURATION**: デバイスの設定を有効化
2. **SET_INTERFACE**: インターフェースを選択
3. **SET_PROTOCOL**: Boot Protocolを設定
4. **GET_REPORT**: キー入力を取得（ループで継続的に呼び出し）

```rust
pub async fn start_usb_keyboard(
    xhc: &Rc<Controller>,
    slot: u8,
    ctrl_ep_ring: &mut CommandRing,
    descriptors: &Vec<UsbDescriptor>,
) -> Result<()> {
    let (config_desc, interface_desc, _) =
        pick_interface_with_triple(descriptors, (3, 1, 1))
            .ok_or("No USB KBD Boot interface found")?;

    xhc.request_set_config(slot, ctrl_ep_ring, config_desc.config_value()).await?;
    xhc.request_set_interface(slot, ctrl_ep_ring,
        interface_desc.interface_number, interface_desc.alt_setting).await?;
    xhc.request_set_protocol(slot, ctrl_ep_ring,
        interface_desc.interface_number, UsbHidProtocol::BootProtocol as u8).await?;

    // キー入力のポーリングループ
    let mut prev_pressed = BTreeSet::new();
    loop {
        let report = request_hid_report(xhc, slot, ctrl_ep_ring).await?;
        let pressed = BTreeSet::from_iter(report.into_iter().skip(2).filter(|id| *id != 0));
        // キーの押下/離上を検出
    }
}
```

## 6.4 HID Report Descriptorのパース

### 6.4.1 Report Descriptorの概要

Boot Protocolでは固定形式のレポートを使用するが、より高度なデバイス（タブレット等）ではReport Descriptorを解析して入力データの形式を理解する必要がある。

Report Descriptorはアイテムの列で構成され、各アイテムは以下の形式：
- **プレフィックスバイト**: サイズ(2bit) + タイプ(2bit) + タグ(4bit)
- **データ**: 0〜4バイト

### 6.4.2 アイテムタイプ

```rust
enum UsbHidReportItemType {
    Main = 0,    // Input, Output, Collection等
    Global = 1,  // Usage Page, Report Size等
    Local = 2,   // Usage, Usage Min/Max等
    Reserved = 3,
}
```

### 6.4.3 Usage Page と Usage

HIDデバイスの入力項目は「Usage」で識別される：

```rust
pub enum UsbHidUsagePage {
    GenericDesktop,  // 0x01: 汎用デスクトップ（マウス、キーボード等）
    Button,          // 0x09: ボタン
    UnknownUsagePage(usize),
}

pub enum UsbHidUsage {
    Pointer,    // ポインタデバイス
    Mouse,      // マウス
    X,          // X座標
    Y,          // Y座標
    Wheel,      // ホイール
    Button(usize), // ボタン
    Constant,   // 定数（パディング）
    UnknownUsage(usize),
}
```

### 6.4.4 入力アイテム構造体

```rust
pub struct UsbHidReportInputItem {
    pub usage: UsbHidUsage,    // この項目が表すもの
    pub bit_size: usize,       // ビット幅
    pub is_array: bool,        // 配列形式かどうか
    pub is_absolute: bool,     // 絶対座標かどうか
    pub bit_offset: usize,     // レポート内のビットオフセット
    pub logical_min: u32,      // 論理最小値
    pub logical_max: u32,      // 論理最大値
}
```

## 6.5 USB Tablet（マウス）ドライバ

### 6.5.1 QEMUのUSB Tablet

QEMUは`usb-tablet`デバイスをエミュレートしており、以下の特徴がある：
- Vendor ID: 0x0627
- Product ID: 0x0001
- インターフェーストリプル: (3, 0, 0) - HIDクラス、サブクラス/プロトコル = 0

このデバイスは絶対座標でマウス位置を報告するため、Boot Protocolではなく Report Descriptorをパースする必要がある。

### 6.5.2 タブレット初期化

```rust
pub async fn start_usb_tablet(
    xhc: &Rc<Controller>,
    slot: u8,
    ctrl_ep_ring: &mut CommandRing,
    device_descriptor: &UsbDeviceDescriptor,
    descriptors: &Vec<UsbDescriptor>,
) -> Result<()> {
    // デバイスの識別
    if device_descriptor.vendor_id != 0x0627 || device_descriptor.product_id != 0x0001 {
        return Err("Not a USB Tablet");
    }

    // HID Descriptorを取得
    let hid_desc = /* ... */;

    // Report Descriptorを取得してパース
    let report = request_hid_report_descriptor(
        xhc, slot, ctrl_ep_ring,
        interface_desc.interface_number,
        hid_desc.report_descriptor_length as usize,
    ).await?;

    let input_report_items = parse_hid_report_descriptor(&report)?;
    // ...
}
```

### 6.5.3 座標のマッピング

Report Descriptorから取得した論理範囲を画面解像度にマッピングする：

```rust
pub fn map_value_in_range_inclusive(
    from: RangeInclusive<i64>,
    to: RangeInclusive<i64>,
    v: i64,
) -> Result<i64> {
    if !from.contains(&v) {
        Err("v is not in range from")
    } else {
        let from_left = (v - *from.start()) as i128;
        let from_width = (from.end() - from.start()) as i128;
        let to_width = (to.end() - to.start()) as i128;
        if from_width == 0 {
            Ok(*to.start())
        } else {
            let to_left = from_left * to_width / from_width;
            Ok(to.start() + to_left as i64)
        }
    }
}
```

## 6.6 ビット操作ユーティリティ

### 6.6.1 リトルエンディアンバイト列からのビット抽出

HIDレポートはリトルエンディアン形式で、ビット単位でデータを格納している。これを効率的に抽出する関数を実装：

```rust
pub fn extract_bits_from_le_bytes(bytes: &[u8], shift: usize, width: usize) -> Option<u64> {
    if width == 0 {
        return None;
    }
    let byte_range = (shift / 8)..((shift + width + 7) / 8);
    let mut value = 0u64;
    let bit_shift = shift - byte_range.start * 8;
    bytes.get(byte_range).map(|bytes_in_range| {
        for (i, v) in bytes_in_range.iter().enumerate() {
            value |= ((*v as u128) << (i * 8) >> bit_shift) as u64;
        }
        extract_bits(value, 0, width)
    })
}
```

### 6.6.2 符号付き値の処理

HIDレポートの値は符号付きの場合がある（相対座標など）。符号拡張を行う：

```rust
fn value_from_report(&self, report: &[u8]) -> Option<i64> {
    extract_bits_from_le_bytes(report, self.bit_offset, self.bit_size).map(|v| {
        // 最上位ビットが1なら負の値
        if self.bit_size >= 2 && extract_bits(v, self.bit_size - 1, 1) == 1 {
            -(!extract_bits(v, 0, self.bit_size - 1) as i64) - 1
        } else {
            v as i64
        }
    })
}
```

## 6.7 新規追加ファイル一覧

| ファイル | 説明 |
|---------|------|
| `src/usb.rs` | USBディスクリプタ定義、パース、リクエスト関数 |
| `src/keyboard.rs` | USBキーボードドライバ |
| `src/tablet.rs` | USB Tabletドライバ、HID Report Descriptorパーサ |
| `src/slice.rs` | Sliceableトレイト |
| `src/pin.rs` | IntoPinnedMutableSliceトレイト |
| `src/range.rs` | 範囲マッピング関数 |

## 6.8 xhci.rsの主な変更点

### 追加されたリクエスト関数

| 関数 | 説明 |
|-----|------|
| `request_descriptor` | 汎用ディスクリプタ取得 |
| `request_descriptor_for_interface` | インターフェース向けディスクリプタ取得 |
| `request_report_bytes` | HID入力レポート取得 |
| `request_set_config` | SET_CONFIGURATIONリクエスト |
| `request_set_interface` | SET_INTERFACEリクエスト |
| `request_set_protocol` | SET_PROTOCOLリクエスト（Boot/Report Protocol切り替え） |

### SetupStageTrbの定数追加

```rust
pub const REQ_TYPE_TYPE_CLASS: u8 = 1 << 5;      // クラス固有リクエスト
pub const REQ_TYPE_TO_INTERFACE: u8 = 1;         // インターフェースへのリクエスト
pub const REQ_GET_REPORT: u8 = 1;                // HID GET_REPORT
pub const REQ_SET_CONFIGURATION: u8 = 9;
pub const REQ_SET_INTERFACE: u8 = 11;
pub const REQ_SET_PROTOCOL: u8 = 0x0b;
```

## 6.9 デバイスドライバの選択ロジック

```rust
// xhci.rs run() 内
if start_usb_keyboard(&xhc, slot, &mut ctrl_ep_ring, &descriptors)
    .await.is_ok()
{
    return Ok(());
}
if start_usb_tablet(&xhc, slot, &mut ctrl_ep_ring, &device_descriptor, &descriptors)
    .await.is_ok()
{
    return Ok(());
}
info!("xhci: No available drivers...");
```

デバイスのディスクリプタを確認し、対応するドライバを順に試行する。最初に成功したドライバがデバイスを制御する。

## 6.10 まとめ

本章では以下を実装した：

1. **USBディスクリプタシステム**: Device/Configuration/Interface/Endpoint Descriptorの読み取りとパース
2. **USBキーボードドライバ**: Boot Protocolを使用したキー入力の取得
3. **HID Report Descriptorパーサ**: 汎用的なHIDデバイス対応の基盤
4. **USB Tabletドライバ**: Report Descriptorを解析してマウス座標を取得

これにより、xHCIコントローラを通じてUSB HIDデバイスと通信し、ユーザー入力を処理できるようになった。
