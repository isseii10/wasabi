# Chapter 5: Serial Port、PCI、xHCIドライバの実装

このドキュメントでは、1-chapter4から最新のコミットまでに実装された主要な機能について解説します。

---

## 全体像：何ができるようになったのか

Chapter 5のテーマは**「外部デバイスとの通信」**です。

### 達成したこと

```
[Chapter 4まで]
OS本体だけが動く状態
- 時間が測れる（HPET）
- 複数の作業を並行実行できる（Executor）

[Chapter 5で追加]
外部デバイスを見つけて使えるようになった
- デバッグログが出せる（Serial Port）
- デバイスを探せる（PCI）
- USBデバイスと通信できる（xHCI）
```

### 3つの階層

Chapter 5の実装は3階層に分かれています：

```
┌─────────────────────────────────────┐
│  レイヤー3: USBデバイスとの通信      │
│  (xHCI - USB管理者)                 │
│  「キーボード・マウスと話す」        │
└─────────────────────────────────────┘
              ↑
┌─────────────────────────────────────┐
│  レイヤー2: デバイスの発見           │
│  (PCI - デバイス住所録)             │
│  「何がつながってる？」             │
└─────────────────────────────────────┘
              ↑
┌─────────────────────────────────────┐
│  レイヤー1: デバッグ出力             │
│  (Serial Port - ログ出力)           │
│  「今何が起きてる？」               │
└─────────────────────────────────────┘
```

---

## 具体的なストーリー：ホテルの例え

Chapter 5全体を**ホテルの運営**に例えて説明します。

```
コンピュータ = ホテル全体
OS = ホテル支配人
USBデバイス = お客様（キーボード、マウス、USBメモリなど）
xHCI = フロントデスク（お客様の受付担当）
```

### ステップ1: 業務日誌の準備（Serial Port）

```
支配人「今日の業務を記録しよう」
  ↓
業務日誌を開く
  ↓
「9:00 ホテル開業しました」と記録
```

**Serial Port = 業務日誌**
- スタッフの作業を記録
- 後で振り返って問題を調査できる
- `info!("xHCI起動")`などのログを出力

### ステップ2: 館内施設の確認（PCI）

```
支配人「うちのホテルにはどんな施設がある？」
  ↓
館内を巡回
  ↓
発見:
  - フロントデスク（USBコントローラ）
  - レストラン（ネットワークカード）
  - 娯楽室（グラフィックカード）
```

**PCI = 館内施設マップ**
- どんな設備があるか調査
- 各施設の場所（メモリアドレス）を記録
- vendor ID = 施設の種類、device ID = 具体的な型番

### ステップ3: フロントデスクの準備（xHCI初期化）

```
支配人「フロントデスクを開けるぞ！」
  ↓
1. フロントをリセット（昨日の書類を片付け）
2. カウンターを設置
   - チェックインカウンター（Command Ring）
   - 完了通知ボード（Event Ring）
3. 「営業開始！」の札を出す
  ↓
フロント業務スタート！
```

**xHCI = フロントデスク**
- お客様（USBデバイス）の受付窓口
- Command Ring = チェックイン用紙を置くトレイ
- Event Ring = 「手続き完了しました」の通知ボード
- Doorbell = 呼び出しベル

### ステップ4: お客様の来館（USBデバイス検出）

```
フロント「玄関に誰か来た？」
  ↓
玄関3番（ポート3）をチェック
  ↓
フロント「お客様発見！」
  ↓
まず健康チェック（デバイスリセット）
  ↓
「問題なし。ロビーへどうぞ」
```

**USBポート = ホテルの玄関**
- 各ポートは異なる入口
- CCS（Current Connect Status）= 「誰かいます」センサー
- リセット = 健康チェックと本人確認

### ステップ5: 部屋の確保（Enable Slot Command）

```
フロント「お客様に部屋を用意します」
  ↓
チェックインカウンターに用紙を置く
「空き部屋ください」
  ↓
呼び出しベルをチーン♪
  ↓
支配人が処理して通知ボードに結果を貼る
「303号室が空いてます」
  ↓
フロント「303号室に決定！」
```

**Enable Slot Command = 部屋の予約**
- スロット番号 = 部屋番号
- Command Ring = チェックインカウンターのトレイ
- Doorbell = フロントの呼び出しベル
- Event Ring = 「手続き完了」の通知ボード

### ステップ6: 宿泊カードの記入（Address Device Command）

```
フロント「303号室のお客様情報を登録」
  ↓
宿泊カードに記入:
  - 玄関番号: 3番
  - 宿泊プラン: スタンダード（USB 2.0 High Speed）
  - 1日の予算: 64ドル（パケットサイズ）
  ↓
カウンターに提出「登録お願いします」
  ↓
支配人が処理して完了通知
  ↓
ルームキー発行！サービス利用開始！
```

**Address Device Command = 宿泊手続き**
- Input Context = 宿泊カード（お客様情報）
- Output Context = ホテル側の顧客管理台帳
- ルームキー発行 = USBデバイスとの通信開始

---

## アーキテクチャの進化

```
Chapter 4:
┌──────────────┐
│   OS本体     │
├──────────────┤
│   ACPI       │ ← ハードウェア情報の取得
│   HPET       │ ← 時間計測
│   Executor   │ ← タスク管理
└──────────────┘

Chapter 5:
┌──────────────────────────────┐
│         OS本体                │
├──────────────────────────────┤
│   Serial Port  ← デバッグ出力 │
├──────────────────────────────┤
│   PCI         ← デバイス発見  │
├──────────────────────────────┤
│   xHCI        ← USB通信       │
│     ↓                         │
│  USBキーボード                │
│  USBマウス                    │
│  USBメモリ                    │
└──────────────────────────────┘
```

---

## 重要な概念：Ring Bufferとは

xHCIの中心的な仕組みが**Ring Buffer（輪っかの配列）**です。

### 仕組み

```
回転寿司のレーンを想像してください：

Command Ring（注文レーン）:
  OS → → → xHC
┌───┬───┬───┬───┬───┐
│注文│注文│空 │空 │空 │ ← OSが注文を置く
└───┴───┴───┴───┴───┘
  ↑                 ↑
 書く場所          読む場所
                  (xHC)

Event Ring（配達レーン）:
  xHC → → → OS
┌───┬───┬───┬───┬───┐
│結果│結果│空 │空 │空 │ ← xHCが結果を置く
└───┴───┴───┴───┴───┘
  ↑                 ↑
 書く場所         読む場所
 (xHC)            (OS)
```

### Cycle Bit（世代フラグ）

問題：「この皿（TRB）は新しい？それとも前の周回の残り？」

解決策：Cycle Bitで世代を管理

```
1周目: Cycle Bit = 1 の皿だけ有効
┌───┬───┬───┬───┬───┐
│ 1 │ 1 │ 0 │ 0 │ 0 │
└───┴───┴───┴───┴───┘
  有効 有効 無効

2周目: Cycle Bit = 0 の皿だけ有効
┌───┬───┬───┬───┬───┐
│ 1 │ 1 │ 0 │ 0 │ 0 │
└───┴───┴───┴───┴───┘
  無効      有効 有効 有効
```

**例え:** 消費期限シール
「1月製造」「2月製造」とシールを貼り替えて、古いものと区別

---

## まとめ：Chapter 5で何を達成したか

### 機能面

| 機能 | 役割 | 例え |
|-----|------|------|
| Serial Port | デバッグログ出力 | 工事現場の掲示板 |
| PCI | デバイス発見 | 部屋の棚卸し |
| xHCI | USB通信管理 | 郵便配達システム |
| Global Executor | タスク一元管理 | 1人の司令塔で指揮 |

### 技術的進歩

1. **デバイス抽象化** - ハードウェアを見つけて制御できる
2. **非同期通信** - 待ち時間を効率的に処理
3. **メモリ管理** - デバイス用のメモリ領域を確保
4. **割り込みの代わり** - ポーリングで状態を監視

### 次のステップ

今はまだ「デバイスを発見して通信の準備ができた」段階。

次は：
- キーボードから文字を読み取る
- USBメモリからファイルを読む
- 画面に文字を表示する

---

## 目次（詳細説明）

1. [Serial Port (シリアルポート通信)](#serial-port-シリアルポート通信)
2. [PCI (Peripheral Component Interconnect)](#pci-peripheral-component-interconnect)
3. [xHCIドライバ (USB 3.0ホストコントローラ)](#xhciドライバ-usb-30ホストコントローラ)
4. [グローバルExecutorの導入](#グローバルexecutorの導入)
5. [ページテーブルの最適化](#ページテーブルの最適化)
6. [補助機能の実装](#補助機能の実装)

---

## コミット履歴概要

```
7a358fb Address Device Command
9dab9fd slotの割り当て
69efab7 EventRing
b34ca37 xHCIドライバの実装
f39c576 xHCIドライバの初期化
f62e7c9 xHCIドライバ
fd9ae50 ページテーブルの高速化
19df3f4 SHIFTを削除
87ab581 global_executor
a8566a5 xHCIの追加
b4d5b26 pciのデバイス一覧をmonitorに出す
b866dec pci
63acd4c serial portで入出力
```

---

## Serial Port (シリアルポート通信)

### コミット: 63acd4c

### Serial Portとは

**Serial Port** は、古典的なシリアル通信インターフェースです。OSデバッグ時のログ出力に使用します。QEMUでは、シリアルポートの出力をホストマシンのファイルやコンソールにリダイレクトできます。

### 実装概要 (src/serial.rs)

```rust
pub struct SerialPort {
    base: u16,  // I/Oポートのベースアドレス
}
```

**COM1ポートの使用:**
- ベースアドレス: `0x3f8`
- 標準的なPCで最も一般的なシリアルポート

### 初期化処理

```rust
pub fn init(&mut self) {
    // 1. すべての割り込みを無効化
    write_io_port_u8(self.base + 1, 0x00);

    // 2. DLAB有効化（ボーレート設定モード）
    write_io_port_u8(self.base + 3, 0x80);

    // 3. ボーレート設定 (115200 baud)
    const BAUD_DIVISOR: u16 = 0x0001;
    write_io_port_u8(self.base, (BAUD_DIVISOR & 0xff) as u8);
    write_io_port_u8(self.base + 1, (BAUD_DIVISOR >> 8) as u8);

    // 4. データフォーマット設定 (8bit, パリティなし, 1ストップビット)
    write_io_port_u8(self.base + 3, 0x03);

    // 5. FIFO有効化、14バイト閾値
    write_io_port_u8(self.base + 2, 0xC7);

    // 6. 割り込み有効化、RTS/DSR設定
    write_io_port_u8(self.base + 4, 0x0B);
}
```

### I/Oポートマッピング

| オフセット | レジスタ | 説明 |
|-----------|---------|------|
| +0 | Data | データ送受信 (DLABオフ時) |
| +1 | IER | 割り込み有効化レジスタ |
| +2 | IIR/FCR | 割り込み識別/FIFOコントロール |
| +3 | LCR | ラインコントロール (データフォーマット、DLAB) |
| +4 | MCR | モデムコントロール |
| +5 | LSR | ラインステータス (送信準備完了フラグ等) |

### ループバックテスト

```rust
pub fn loopback_test(&self) -> Result<()> {
    // ループバックモードに設定
    write_io_port_u8(self.base + 4, 0x1e);

    // 'T'を送信
    self.send_char('T');

    // 受信データが'T'であることを確認
    if self.try_read().ok_or("loopback_test failed: No response")? != b'T' {
        return Err("loopback_test failed: wrong data received");
    }

    // 通常モードに戻す
    write_io_port_u8(self.base + 4, 0x0f);
    Ok(())
}
```

**ループバックテストの目的:**
- ハードウェアが正常に動作しているか確認
- 送信した文字が正しく受信できるかテスト

### 文字送信処理

```rust
pub fn send_char(&self, c: char) {
    // LSR (Line Status Register) のビット5をチェック
    // ビット5: Transmitter Holding Register Empty (THRE)
    while (read_io_port_u8(self.base + 5) & 0x20) == 0 {
        busy_loop_hint();  // CPUに一時停止のヒントを送る
    }
    write_io_port_u8(self.base, c as u8);
}
```

### fmt::Writeトレイトの実装

```rust
impl fmt::Write for SerialPort {
    fn write_str(&mut self, s: &str) -> fmt::Result {
        let serial = Self::default();
        serial.send_str(s);
        Ok(())
    }
}
```

これにより、`write!` や `writeln!` マクロが使用可能になります。

---

## PCI (Peripheral Component Interconnect)

### コミット: b866dec, b4d5b26

### PCIとは

**PCI (Peripheral Component Interconnect)** は、コンピュータに接続された周辺デバイスを管理するための標準バスシステムです。

**PCIアドレス空間:**
- **Bus**: 0-255 (8ビット)
- **Device**: 0-31 (5ビット)
- **Function**: 0-7 (3ビット)

合計で最大 256 × 32 × 8 = 65,536 個のデバイスを識別可能

### BusDeviceFunction構造体

```rust
#[derive(Copy, Clone, PartialEq, Eq, PartialOrd, Ord)]
pub struct BusDeviceFunction {
    id: u16,  // Bus(8bit) | Device(5bit) | Function(3bit)
}
```

**ビットレイアウト:**
```
[15..8]: Bus番号
[7..3]:  Device番号
[2..0]:  Function番号
```

**実装例:**
```rust
impl BusDeviceFunction {
    pub fn new(bus: usize, device: usize, function: usize) -> Result<Self> {
        if !(0..256).contains(&bus)
            || !(0..32).contains(&device)
            || !(0..8).contains(&function) {
            Err("PCI bus device function out of range")
        } else {
            Ok(Self {
                id: ((bus << 8) | (device << 3) | function) as u16,
            })
        }
    }
}
```

### VendorDeviceId構造体

各PCIデバイスは、**Vendor ID** と **Device ID** で識別されます。

```rust
#[derive(Copy, Clone, PartialEq, Eq)]
pub struct VendorDeviceId {
    pub vendor: u16,  // ベンダー識別子 (例: 0x8086 = Intel)
    pub device: u16,  // デバイス識別子
}
```

### ACPI MCFGとの連携

PCI Configuration Spaceへのアクセスは、**ACPI MCFG (Memory Mapped Configuration)** テーブルで提供されるメモリマップドアクセスを使用します。

```rust
pub struct Pci {
    ecm_range: Range<usize>,  // Enhanced Configuration Mechanism のアドレス範囲
}

impl Pci {
    pub fn new(mcfg: &AcpiMcfgDescriptor) -> Self {
        // MCFGの最初のエントリからPCI Configuration Spaceのベースアドレスを取得
        assert!(mcfg.num_of_entries() == 1);
        let pci_config_space_base = mcfg.entry(0)
            .expect("Out of range")
            .base_address() as usize;
        let pci_config_space_end = pci_config_space_base + (1 << 24); // 16MB
        Self {
            ecm_range: pci_config_space_base..pci_config_space_end,
        }
    }
}
```

### PCIコンフィグレーションスペース

各デバイスは256バイトのコンフィグレーションスペースを持ちます。

**共通ヘッダー (最初の64バイト):**
```
オフセット    サイズ    説明
0x00         2         Vendor ID
0x02         2         Device ID
0x04         2         Command
0x06         2         Status
0x10-0x24    20        Base Address Registers (BAR0-BAR5)
```

### BARレジスタ (Base Address Register)

BARレジスタは、デバイスのメモリ/IOリソースのベースアドレスを格納します。

```rust
pub fn try_bar0_mem64(&self, bdf: BusDeviceFunction) -> Result<BarMem64> {
    // BAR0を読み取る
    let bar0 = self.read_register_u64(bdf, 0x10)?;

    // ビット[2:0] = 0b100 -> 64bit Memory, Non-prefetchable
    if bar0 & 0b0111 == 0b0100 {
        let addr = (bar0 & !0b1111) as *mut u8;

        // サイズを取得するために全ビットを1にして書き込む
        self.write_regiter_u64(bdf, 0x10, !0u64)?;
        let size = 1 + !(self.read_register_u64(bdf, 0x10)? & !0b1111);

        // 元の値に戻す
        self.write_regiter_u64(bdf, 0x10, bar0)?;

        Ok(BarMem64 { addr, size })
    } else {
        Err("Unexpected BAR0 Type")
    }
}
```

**BARレジスタからのサイズ取得手順:**
1. 現在の値を保存
2. 全ビットを1に設定
3. 読み戻す → サイズビットマスクが得られる
4. ビット反転してサイズを計算
5. 元の値を復元

### デバイス列挙

```rust
pub fn prove_devices(&self) {
    for bdf in BusDeviceFunction::iter() {
        if let Some(vd) = self.read_vendor_id_and_device_id(bdf) {
            info!("{vd}");
            // 対応するドライバがあればアタッチ
            if PciXhciDriver::supports(vd) {
                if let Err(e) = PciXhciDriver::attach(self, bdf) {
                    error!("PCI: driver attach() failed: {e:?}")
                }
            }
        }
    }
}
```

**処理フロー:**
1. すべてのBusDeviceFunction組み合わせを走査
2. Vendor ID / Device IDを読み取り
3. 0xFFFFの場合はスキップ（未接続）
4. 対応ドライバがあれば初期化

### Bus Masterとキャッシュ制御

```rust
pub fn enable_bus_master(&self, bdf: BusDeviceFunction) -> Result<()> {
    self.set_command_and_status_flags(bdf, 1 << 2 /* Bus Master Enable */)
}

impl BarMem64 {
    pub fn disable_cache(&self) {
        let vstart = self.addr() as u64;
        let vend = self.addr() as u64 + self.size();
        unsafe {
            with_current_page_table(|pt| {
                pt.create_mapping(vstart, vend, vstart, PageAttr::ReadWriteIo)
                    .expect("Failed to create mapping")
            })
        }
    }
}
```

**Bus Masterの有効化:**
- デバイスがDMAを実行できるようにする
- Commandレジスタのビット2を設定

**キャッシュ無効化:**
- デバイスのMMIOレジスタ領域をページテーブルで`ReadWriteIo`属性に設定
- CPUキャッシュをバイパスして直接メモリアクセス

---

## xHCIドライバ (USB 3.0ホストコントローラ)

### コミット: f62e7c9, f39c576, b34ca37, 69efab7, 9dab9fd, 7a358fb

### xHCIとは

**xHCI (eXtensible Host Controller Interface)** は、USB 3.0/3.1デバイスを制御するためのホストコントローラ規格です。

**主な特徴:**
- USB 1.x/2.0/3.0デバイスの統一的な管理
- Ring Buffer構造によるコマンド/イベント処理
- 高速データ転送とDMAサポート

### xHCIアーキテクチャ

```
┌──────────────────────────────────────────┐
│         xHCI Host Controller             │
├──────────────────────────────────────────┤
│  Capability Registers (読み取り専用)      │
│  - MaxSlots, MaxPorts, Offset情報        │
├──────────────────────────────────────────┤
│  Operational Registers (読み書き可能)     │
│  - USBCMD, USBSTS, DCBAAP, CRCR          │
├──────────────────────────────────────────┤
│  Runtime Registers                       │
│  - MFINDEX, Interrupter Register Set     │
├──────────────────────────────────────────┤
│  Doorbell Registers                      │
│  - コマンド/転送通知用                    │
└──────────────────────────────────────────┘
```

### レジスタ構造の定義

#### Capability Registers

```rust
#[repr(C)]
struct CapabilityRegisters {
    caplength: Volatile<u8>,        // Operational Registersまでのオフセット
    reserved: Volatile<u8>,
    version: Volatile<u16>,         // xHCIバージョン
    hcsparams1: Volatile<u32>,      // 構造パラメータ1 (MaxSlots, MaxPorts)
    hcsparams2: Volatile<u32>,      // 構造パラメータ2 (Scratchpad Buffers)
    hcsparams3: Volatile<u32>,
    hccparams1: Volatile<u32>,      // 機能パラメータ
    dboff: Volatile<u32>,           // Doorbell Registersオフセット
    rtsoff: Volatile<u32>,          // Runtime Registersオフセット
    hccparams2: Volatile<u32>,
}
```

**パラメータの抽出例:**
```rust
fn num_of_device_slots(&self) -> usize {
    extract_bits(self.hcsparams1.read(), 0, 8) as usize
}

fn num_of_ports(&self) -> usize {
    extract_bits(self.hcsparams1.read(), 24, 8) as usize
}
```

#### Operational Registers

```rust
#[repr(C)]
struct OperationalRegisters {
    usbcmd: Volatile<u32>,      // コマンドレジスタ (Run/Stop, Reset)
    usbsts: Volatile<u32>,      // ステータスレジスタ (HCHalted)
    pagesize: Volatile<u32>,    // ページサイズ
    rsvdz1: [u32; 2],
    dnctrl: Volatile<u32>,
    crcr: Volatile<u64>,        // Command Ring Control Register
    rsvdz2: [u64; 2],
    dcbaap: Volatile<*const RawDeviceContextBaseAddressArray>,  // DCBAA Pointer
    config: Volatile<u64>,      // デバイススロット数設定
}
```

### xHCIの初期化手順

```rust
fn new(mut regs: XhcRegisters) -> Result<Self> {
    // 1. xHCをリセット
    unsafe { regs.op_regs.get_unchecked_mut().reset_xhc(); }

    // 2. Scratchpad Buffersの割り当て
    let scratchpad_buffers = ScratchpadBuffers::alloc(
        regs.cap_regs.as_ref(),
        regs.op_regs.as_ref()
    )?;

    // 3. Device Context Base Address Array (DCBAA) の初期化
    let device_context_base_array = DeviceContextBaseAddressArray::new(scratchpad_buffers);

    // 4. Event Ringの初期化
    let primary_event_ring = Mutex::new(EventRing::new()?);

    // 5. Command Ringの初期化
    let command_ring = Mutex::new(CommandRing::default());

    let mut xhc = Self {
        regs,
        device_context_base_array: Mutex::new(device_context_base_array),
        primary_event_ring,
        command_ring,
    };

    // 6. Primary Event Ringを登録
    xhc.init_primary_event_ring()?;

    // 7. スロット数とDCBAAPを設定
    xhc.init_slots_and_contexts()?;

    // 8. Command Ringを登録
    xhc.init_command_ring();

    // 9. xHCを起動
    info!("Starting xHC...");
    unsafe { xhc.regs.op_regs.get_unchecked_mut() }.start_xhc();
    info!("xHC started running!");

    Ok(xhc)
}
```

### xHCのリセット処理

```rust
fn reset_xhc(&mut self) {
    // 1. Run/Stopビットをクリア (xHCを停止)
    self.clear_command_bits(Self::CMD_RUN_STOP);

    // 2. HCHalted ステータスを待つ
    while self.usbsts.read() & Self::STATUS_HC_HALTED == 0 {
        busy_loop_hint();
    }

    // 3. HC Resetビットをセット
    self.set_command_bits(Self::CMD_HC_RESET);

    // 4. HC Resetビットが自動的にクリアされるまで待つ
    while self.usbcmd.read() & Self::CMD_HC_RESET != 0 {
        busy_loop_hint();
    }
}
```

### Ring Buffer構造

xHCIでは、**Command Ring** と **Event Ring** という2種類のRing Bufferを使用します。

```
Command Ring (Host → xHC):
┌───┬───┬───┬───┬───┬───┐
│TRB│TRB│TRB│TRB│...│LNK│  <- Link TRBで先頭に戻る
└───┴───┴───┴───┴───┴───┘
 ↑                       ↑
producer              consumer
(ソフトウェア)          (xHC)

Event Ring (xHC → Host):
┌───┬───┬───┬───┬───┬───┐
│TRB│TRB│TRB│TRB│TRB│TRB│
└───┴───┴───┴───┴───┴───┘
 ↑                       ↑
producer              consumer
(xHC)                (ソフトウェア)
```

**TRB (Transfer Request Block):**
- 16バイトのデータ構造
- Cycle Bitでプロデューサー/コンシューマーを同期

### TRB構造

```rust
#[repr(C, align(16))]
struct GenericTrbEntry {
    data: Volatile<u64>,      // データフィールド
    option: Volatile<u32>,    // オプションフィールド
    control: Volatile<u32>,   // コントロールフィールド (TRB Type, Cycle Bit)
}
```

**Controlフィールドのビット配置:**
```
[31..24]: Slot ID
[23..16]: Reserved
[15..10]: TRB Type
[9..1]:   TRB固有フラグ
[0]:      Cycle Bit
```

### Cycle Bitによる同期

**Cycle Bit** は、TRBが有効かどうかを示すフラグです。

```rust
fn advance_index(&mut self, new_cycle: bool) -> Result<()> {
    if self.current().cycle_state() == new_cycle {
        return Err("cycle state does not change");
    }
    // Cycle Bitを更新
    self.trb[self.current_index].set_cycle_state(new_cycle);
    // インデックスを進める
    self.current_index = (self.current_index + 1) % self.trb.len();
    Ok(())
}
```

**仕組み:**
1. 最初はすべてのTRBのCycle Bit = 0
2. プロデューサーは書き込み時にCycle Bit = 1に設定
3. コンシューマーはCycle Bit = 1のTRBだけを処理
4. Ringを一周したら、Cycle Bitを反転 (1 → 0, 0 → 1)

### Command Ringの実装

```rust
struct CommandRing {
    ring: IoBox<TrbRing>,
    cycle_state_ours: bool,  // 現在の書き込み側Cycle State
}

impl CommandRing {
    pub fn push(&mut self, mut src: GenericTrbEntry) -> Result<u64> {
        let ring = unsafe { self.ring.get_unchecked_mut() };

        // Ringが満杯でないか確認
        if ring.current().cycle_state() != self.cycle_state_ours {
            return Err("Command Ring is Full");
        }

        // Cycle Bitを設定
        src.set_cycle_state(self.cycle_state_ours);
        let dst_ptr = ring.current_ptr();

        // TRBを書き込み
        ring.write_current(src);
        ring.advance_index(!self.cycle_state_ours)?;

        // Link TRBに到達したらスキップしてCycle反転
        if ring.current().trb_type() == TrbType::Link as u32 {
            ring.advance_index(!self.cycle_state_ours)?;
            self.cycle_state_ours = !self.cycle_state_ours;
        }

        Ok(dst_ptr as u64)
    }
}
```

**Link TRB:**
- Command Ringの最後のスロットに配置
- Ringの先頭アドレスを格納
- xHCがこれを読むと自動的に先頭に戻る

### Event Ringの実装

```rust
struct EventRing {
    ring: IoBox<TrbRing>,
    erst: IoBox<EventRingSegmentTableEntry>,  // Event Ring Segment Table
    cycle_state_ours: bool,     // 現在の読み取り側Cycle State
    erdp: Option<*mut u64>,     // Event Ring Dequeue Pointer
    wait_list: VecDeque<Weak<EventWaitInfo>>,  // イベント待機リスト
}
```

**Event Ring Segment Table (ERST):**
```rust
#[repr(C, align(4096))]
struct EventRingSegmentTableEntry {
    ring_segment_base_address: u64,  // Event Ringの物理アドレス
    ring_segment_size: u16,          // TRB数
    _rsvdz: [u16; 3],
}
```

**イベントのポーリング:**
```rust
async fn poll(&mut self) -> Result<()> {
    if let Some(e) = self.pop()? {
        let mut consumed = false;

        // wait_listを走査して待機中のタスクに通知
        for w in &self.wait_list {
            if let Some(w) = w.upgrade() {
                let w: &EventWaitInfo = w.as_ref();
                if w.matches(&e) {
                    w.resolve(&e)?;
                    consumed = true;
                }
            }
        }

        if !consumed {
            info!("unhandled event: {e:?}");
        }

        // 無効な待機者を削除
        self.cleanup_stale_waiters();
    }
    Ok(())
}
```

### Device Context Base Address Array (DCBAA)

**DCBAA** は、各デバイススロットの Output Context へのポインタ配列です。

```rust
#[repr(C, align(64))]
struct RawDeviceContextBaseAddressArray {
    scratchpad_table_ptr: *const *const u8,  // スロット0: Scratchpad用
    context: [u64; 255],                     // スロット1-255: デバイス用
    _pinned: PhantomPinned,
}
```

**初期化:**
```rust
impl DeviceContextBaseAddressArray {
    fn new(scratchpad_buffers: ScratchpadBuffers) -> Self {
        let mut inner = RawDeviceContextBaseAddressArray::new();
        inner.scratchpad_table_ptr = scratchpad_buffers.table.as_ref().as_ptr();
        let inner = Box::pin(inner);
        Self {
            inner,
            context: unsafe { MaybeUninit::zeroed().assume_init() },
            _scratchpad_buffers: scratchpad_buffers,
        }
    }
}
```

### スロット割り当てとアドレス設定

#### 1. Enable Slot Command

```rust
async fn init_port(xhc: &Rc<Controller>, port: usize) -> Result<u8> {
    let portsc = xhc.regs.portsc.get(port).ok_or("invalid portsc")?;

    // ポートをリセット
    info!("xhci: resetting port {port}");
    portsc.reset_port().await;
    info!("xhci: port {port} has been reset");

    // ポートが有効になったか確認
    portsc.is_enabled()
        .then_some(())
        .ok_or("port is not enabled")?;
    info!("xhci: port {port} is enabled");

    // スロットを割り当て
    let slot = xhc
        .send_command(GenericTrbEntry::cmd_enable_slot())
        .await?
        .slot_id();

    Ok(slot)
}
```

**処理フロー:**
1. ポートのリセット（非同期で待機）
2. ポートの有効化確認
3. Enable Slot Commandを送信
4. Command Completion Eventを待機
5. 割り当てられたSlot IDを取得

#### 2. Address Device Command

```rust
async fn address_device(xhc: &Rc<Controller>, port: usize, slot: u8) -> Result<()> {
    // Output Contextを割り当て
    let output_context = Box::pin(OutputContext::default());
    xhc.set_output_context_for_slot(slot, output_context);

    // Input Control Contextを初期化
    let mut input_ctrl_ctx = InputControlContext::default();
    input_ctrl_ctx.add_context(0)?;  // Slot Context
    input_ctrl_ctx.add_context(1)?;  // Endpoint 0 Context

    // Input Contextを初期化
    let mut input_context = Box::pin(InputContext::default());
    input_context.as_mut().set_input_ctrl_ctx(input_ctrl_ctx)?;

    // Slot Contextを設定
    input_context.as_mut().set_root_hub_port_number(port)?;
    input_context.as_mut().set_last_valid_dci(1)?;  // DCI 1 = EP0

    // Endpoint 0 Contextを設定
    let portsc = xhc.regs.portsc.get(port).ok_or("PORTSC was invalid")?;
    input_context.as_mut().set_port_speed(portsc.port_speed())?;

    let ctrl_ep_ring = CommandRing::default();
    input_context.as_mut().set_ep_ctx(
        1,
        EndpointContext::new_control_endpoint(
            portsc.max_packet_size()?,
            ctrl_ep_ring.ring_phys_addr()
        )?
    )?;

    // Address Device Commandを送信
    let cmd = GenericTrbEntry::cmd_address_device(input_context.as_ref(), slot);
    xhc.send_command(cmd).await?.cmd_result_ok()?;

    Ok(())
}
```

**Input Context構造:**
```
┌──────────────────────────────┐
│  Input Control Context       │  <- どのContextを更新するか指定
├──────────────────────────────┤
│  Slot Context                │  <- デバイス全体の情報
├──────────────────────────────┤
│  Endpoint 0 Context          │  <- デフォルトコントロールエンドポイント
├──────────────────────────────┤
│  Endpoint 1 Context          │
│  ...                         │
│  Endpoint 30 Context         │
└──────────────────────────────┘
```

### Endpoint Context

```rust
#[repr(C, align(32))]
struct EndpointContext {
    data: [u32; 2],
    tr_dequeue_ptr: Volatile<u64>,  // Transfer Ring Dequeue Pointer
    average_trb_length: u16,
    _reserved: [u32; 3],
}
```

**Endpoint Contextのフィールド:**
- **EP Type**: Control, Bulk, Interrupt, Isochronous
- **Max Packet Size**: 転送可能な最大パケットサイズ
- **Error Count**: エラー許容回数
- **TR Dequeue Pointer**: Transfer Ringの開始アドレス

```rust
fn new_control_endpoint(max_packet_size: u16, tr_dequeue_ptr: u64) -> Result<Self> {
    let mut ep = Self::new();
    ep.set_ep_type(EndpointType::Control)?;
    ep.set_dequeue_cycle_state(true)?;
    ep.set_error_count(3)?;
    ep.set_max_packet_size(max_packet_size);
    ep.set_ring_dequeue_pointer(tr_dequeue_ptr)?;
    ep.average_trb_length = 8;  // xHCI仕様 6.2.3
    Ok(ep)
}
```

### ポートの状態管理

```rust
struct PortScEntry {
    ptr: Mutex<*mut u32>,
}

impl PortScEntry {
    fn ccs(&self) -> bool {
        // CCS - Current Connect Status - ROS
        self.bit(0)
    }

    fn ped(&self) -> bool {
        // PED - Port Enabled/Disabled - RW1CS
        self.bit(1)
    }

    fn pr(&self) -> bool {
        // PR - Port Reset - RW1S
        self.bit(4)
    }

    pub async fn reset_port(&self) {
        // Port Powerを有効化
        self.assert_pp();
        while !self.pp() {
            yield_execution().await
        }

        // Port Resetを実行
        self.assert_pr();
        while self.pr() {
            yield_execution().await
        }
    }

    pub fn is_enabled(&self) -> bool {
        self.pp() && self.ccs() && self.ped() && !self.pr()
    }
}
```

**PORTSC (Port Status and Control Register) の主要ビット:**
- **ビット0 (CCS)**: Current Connect Status - デバイス接続状態
- **ビット1 (PED)**: Port Enabled/Disabled - ポート有効/無効
- **ビット4 (PR)**: Port Reset - ポートリセット
- **ビット9 (PP)**: Port Power - ポート電源
- **ビット13-10**: Port Speed - ポート速度 (FullSpeed, HighSpeed, SuperSpeed)

### USB速度の判定

```rust
pub enum UsbMode {
    Unknown(u32),
    FullSpeed,    // USB 1.0 (12 Mbps)
    LowSpeed,     // USB 1.0 (1.5 Mbps)
    HighSpeed,    // USB 2.0 (480 Mbps)
    SuperSpeed,   // USB 3.0 (5 Gbps)
}

impl PortScEntry {
    pub fn port_speed(&self) -> UsbMode {
        // Port Speed - ROS (Read Only, Set by hardware)
        // Protocol Speed ID (PSI) - Table 7-13: Default USB Speed ID Mapping
        match extract_bits(self.value(), 10, 4) {
            1 => UsbMode::FullSpeed,
            2 => UsbMode::LowSpeed,
            3 => UsbMode::HighSpeed,
            4 => UsbMode::SuperSpeed,
            v => UsbMode::Unknown(v),
        }
    }

    pub fn max_packet_size(&self) -> Result<u16> {
        match self.port_speed() {
            UsbMode::FullSpeed | UsbMode::LowSpeed => Ok(8),
            UsbMode::HighSpeed => Ok(64),
            UsbMode::SuperSpeed => Ok(512),
            _ => Err("Unknown Protocol Speed ID"),
        }
    }
}
```

### Doorbell Registerによる通知

**Doorbell Register** は、xHCに新しいコマンドや転送要求を通知するためのレジスタです。

```rust
pub struct Doorbell {
    ptr: Mutex<*mut u32>,
}

impl Doorbell {
    pub fn notify(&self, target: u8, task: u16) {
        let value = (target as u32) | ((task as u32) << 16);
        unsafe {
            write_volatile(*self.ptr.lock(), value);
        }
    }
}
```

**Doorbell配列:**
- **index 0**: Host Controller用（Command Ring通知）
- **index 1-255**: Device Slot用（Transfer Ring通知）

```rust
impl Controller {
    fn notify_xhc(&self) {
        // Doorbell[0]を叩いてCommand Ringの更新を通知
        self.regs.doorbell_regs[0].notify(0, 0);
    }
}
```

### 非同期コマンド送信

```rust
async fn send_command(&self, cmd: GenericTrbEntry) -> Result<GenericTrbEntry> {
    // Command RingにTRBをプッシュ
    let cmd_ptr = self.command_ring.lock().push(cmd)?;

    // Doorbellを叩いてxHCに通知
    self.notify_xhc();

    // Command Completion Eventを待機
    EventFuture::new_for_trb(&self.primary_event_ring, cmd_ptr).await
}
```

**EventFuture:**
```rust
struct EventFuture {
    wait_on: Rc<EventWaitInfo>,
    _pinned: PhantomPinned,
}

impl Future for EventFuture {
    type Output = Result<GenericTrbEntry>;

    fn poll(self: Pin<&mut Self>, _: &mut Context) -> Poll<Result<GenericTrbEntry>> {
        let mut_self = unsafe { self.get_unchecked_mut() };
        if let Some(trb) = mut_self.wait_on.trbs.lock().pop_front() {
            Poll::Ready(Ok(trb))
        } else {
            Poll::Pending
        }
    }
}
```

**仕組み:**
1. Command RingにTRBを書き込み
2. Doorbellで通知
3. EventFutureでEvent待機登録
4. xHCがコマンドを実行してEvent Ringにイベントを書き込む
5. ポーリングループがイベントを検出
6. EventFutureがReady状態に遷移
7. 結果を取得

---

## グローバルExecutorの導入

### コミット: 87ab581

### 変更内容

Chapter 4では各モジュールがローカルにExecutorを持つ設計でしたが、Chapter 5では**グローバルExecutor**を導入し、システム全体で1つのExecutorを共有する設計に変更されました。

### グローバルExecutorの実装

```rust
static GLOBAL_EXECUTOR: Mutex<Option<Executor>> = Mutex::new(None);

#[track_caller]
pub fn spawn_global(future: impl Future<Output = Result<()>> + 'static) {
    let task = Task::new(future);
    GLOBAL_EXECUTOR.lock().get_or_insert_default().enqueue(task);
}

pub fn start_global_executor() -> ! {
    info!("Starting global executor loop");
    Executor::run(&GLOBAL_EXECUTOR);
}
```

**利点:**
- タスクの一元管理
- コンテキストスイッチのオーバーヘッド削減
- グローバルな協調動作

### 使用例

```rust
pub fn attach(pci: &Pci, bdf: BusDeviceFunction) -> Result<()> {
    info!("Xhci found at: {bdf:?}");
    pci.disable_interrupt(bdf)?;
    pci.enable_bus_master(bdf)?;
    let bar0 = pci.try_bar0_mem64(bdf)?;
    bar0.disable_cache();
    let regs = Self::setup_xhc_registers(&bar0)?;
    let xhc = Controller::new(regs)?;

    // グローバルExecutorにタスクを登録
    spawn_global(Self::run(xhc));
    Ok(())
}
```

---

## ページテーブルの最適化

### コミット: 19df3f4, fd9ae50

### 変更内容

ページテーブル操作の際の**SHIFTマクロを削除**し、より直接的なビット操作に変更することで、コードの可読性と性能を向上させました。

### 変更前

```rust
const SHIFT_PML4: usize = 39;
const SHIFT_PDPT: usize = 30;
const SHIFT_PD: usize = 21;
const SHIFT_PT: usize = 12;

fn virtual_to_pml4_index(vaddr: u64) -> usize {
    ((vaddr >> SHIFT_PML4) & 0x1FF) as usize
}
```

### 変更後

```rust
fn virtual_to_pml4_index(vaddr: u64) -> usize {
    ((vaddr >> 39) & 0x1FF) as usize
}

fn virtual_to_pdpt_index(vaddr: u64) -> usize {
    ((vaddr >> 30) & 0x1FF) as usize
}

fn virtual_to_pd_index(vaddr: u64) -> usize {
    ((vaddr >> 21) & 0x1FF) as usize
}

fn virtual_to_pt_index(vaddr: u64) -> usize {
    ((vaddr >> 12) & 0x1FF) as usize
}
```

**改善点:**
- 定数を直接埋め込むことでインライン化が容易
- マジックナンバーではなく、ビットシフトの意味が明確
- コンパイラの最適化がより効果的に働く

---

## 補助機能の実装

### コミット: f62e7c9

Chapter 5では、xHCIドライバの実装に必要な補助モジュールが追加されました。

### src/bits.rs - ビット操作ユーティリティ

```rust
/// ビット範囲を抽出する
/// value: 対象値
/// start: 開始ビット位置
/// width: ビット幅
pub fn extract_bits(value: u32, start: usize, width: usize) -> u32 {
    let mask = (1u32 << width) - 1;
    (value >> start) & mask
}
```

**使用例:**
```rust
// hcsparams1の[7:0]ビットから MaxSlots を取得
fn num_of_device_slots(&self) -> usize {
    extract_bits(self.hcsparams1.read(), 0, 8) as usize
}

// hcsparams1の[31:24]ビットから MaxPorts を取得
fn num_of_ports(&self) -> usize {
    extract_bits(self.hcsparams1.read(), 24, 8) as usize
}
```

### src/volatile.rs - Volatileアクセス

```rust
#[repr(transparent)]
pub struct Volatile<T: Copy> {
    value: T,
}

impl<T: Copy> Volatile<T> {
    pub fn read(&self) -> T {
        unsafe { read_volatile(&self.value) }
    }

    pub fn write(&mut self, value: T) {
        unsafe { write_volatile(&mut self.value, value) }
    }

    pub fn read_bits(&self, start: usize, width: usize) -> u32
    where
        T: Into<u32>,
    {
        let value: u32 = self.read().into();
        extract_bits(value, start, width)
    }

    pub fn write_bits(&mut self, start: usize, width: usize, bits: u32) -> Result<()>
    where
        T: Into<u32> + From<u32>,
    {
        let value: u32 = self.read().into();
        let mask = ((1u32 << width) - 1) << start;
        let new_value = (value & !mask) | ((bits << start) & mask);
        self.write(new_value.into());
        Ok(())
    }
}
```

**目的:**
- コンパイラによる最適化を防ぐ
- デバイスレジスタへのアクセスに必須
- 読み書きが確実にメモリに反映されることを保証

### src/mmio.rs - Memory-Mapped I/O

```rust
pub struct Mmio<T> {
    ptr: *mut T,
}

impl<T> Mmio<T> {
    pub unsafe fn from_raw(ptr: *mut T) -> Self {
        Self { ptr }
    }

    pub fn as_ref(&self) -> &T {
        unsafe { &*self.ptr }
    }

    pub unsafe fn get_unchecked_mut(&self) -> &mut T {
        &mut *self.ptr
    }
}
```

**IoBox型:**
```rust
pub struct IoBox<T: Default> {
    ptr: *mut T,
}

impl<T: Default> IoBox<T> {
    pub fn new() -> Self {
        let layout = Layout::new::<T>();
        let ptr = ALLOCATOR.alloc_with_options(layout) as *mut T;
        unsafe {
            core::ptr::write(ptr, T::default());
        }
        Self { ptr }
    }
}
```

**用途:**
- デバイスレジスタへの型安全なアクセス
- DMA可能な物理メモリの割り当て
- Pin保証による安全なメモリ管理

---

## まとめ

Chapter 5では、以下の主要な機能が実装されました。

### 実装された機能

1. **Serial Port通信** - デバッグログ出力のための基盤
2. **PCIデバイス管理** - デバイスの列挙と初期化
3. **xHCIドライバ** - USB 3.0ホストコントローラの完全な実装
4. **グローバルExecutor** - 非同期タスクの一元管理
5. **ページテーブル最適化** - メモリ管理の高速化

### アーキテクチャの進化

```
Chapter 4:
OS起動 → ACPI → HPET → ローカルExecutor

Chapter 5:
OS起動 → ACPI → PCI → xHCI → グローバルExecutor
           ↓           ↓
         Serial    USBデバイス
```

### 今後の展開

- **USBデバイスの列挙**: Mass Storage、HID等
- **ファイルシステム**: FAT32対応
- **キーボード/マウス入力**: HIDドライバ実装
- **ネットワーク**: Ethernetドライバ

---

## 参考資料

- [xHCI Specification 1.2](https://www.intel.com/content/www/us/en/products/docs/io/universal-serial-bus/extensible-host-controler-interface-usb-xhci.html)
- [OSDev.org - Serial Ports](https://wiki.osdev.org/Serial_Ports)
- [OSDev.org - PCI](https://wiki.osdev.org/PCI)
- [OSDev.org - USB](https://wiki.osdev.org/USB)
- [PCI Express Base Specification](https://pcisig.com/specifications)
