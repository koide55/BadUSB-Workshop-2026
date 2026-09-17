# RP2040 SECCON バッジで作る BadUSB ハンズオン Windows対応版
### HID キーボード + USB マスストレージの複合デバイスを CircuitPython で安全に体験する

- **対象**: 中級(Python が読める / Windows の基本操作に抵抗がない)
- **所要時間**: 半日(約 3.5〜4 時間、休憩・トラブル対応込み)
- **標的 OS**: Windows 10 / Windows 11(母艦・被害役ともに Windows を想定)
- **題材ボード**: RP2040 搭載 SECCON バッジ基板
- **スタック**: CircuitPython 9.x + `adafruit_hid`

---

## 0. はじめに(講師が最初に読む)

### 0.1 このハンズオンで作るもの
挿すと **「ただのUSBメモリ」に見えるのに、裏でキーボード入力を送り込める** デバイス。
いわゆる **BadUSB / Rubber Ducky** の仕組みを、RP2040 バッジと CircuitPython で理解する。

USB の「コンポジットデバイス(複合デバイス)」という仕組みを使うと、1本のケーブルで
**マスストレージ(MSC)** と **HID キーボード** を同時にホストへ見せられる。
CircuitPython は起動しただけで `CIRCUITPY` ドライブ(MSC)、USB シリアル(CDC)、HID を露出できるため、
「USBメモリに見えるものが、同時にキーボードでもある」ことを観察するのが前半のゴール。

Windows 版の実演は、**メモ帳(Notepad)を開き、無害な固定メッセージを入力するだけ** に限定する。
PowerShell、コマンドプロンプト、任意コマンドの実行、ダウンロード、削除、設定変更、認証情報取得などは扱わない。

### 0.2 学習目標
受講後、参加者は次を説明・実演できる:
1. USB コンポジットデバイスとは何か、なぜ「USBメモリのつもりがキーボード」が成立するのか
2. Windows 10/11 上で CircuitPython を導入し、`CIRCUITPY`、REPL、`adafruit_hid` を確認する
3. CircuitPython で HID キーストロークを送る(修飾キー、レイアウト、待ち時間)
4. Windows 標的で Notepad を開き、固定メッセージを入力する無害デモを実行する
5. マウント直後に確実に発火させるための信頼性対策と `autoreload` の扱いを理解する
6. GPIO アームスイッチで、安全装置付きのデモデバイスにする
7. US/JIS 配列と Windows IME が HID 入力に与える影響を説明する
8. Windows 環境で USB/HID を確認し、守り側の検知・防御を議論する

### 0.3 倫理と前提(**最初に全員へ明示する**)
> 本教材は **自分が所有・管理する機材、または明示的な許可を得た検証環境** に対してのみ使用すること。
> 他人の PC に無断で挿す行為は、たとえ無害なペイロードでも不正アクセス、業務妨害、社内規程違反等に該当し得る。
> ペイロードは **すべて非破壊・可逆** に留める。
> 本稿で扱う Windows 側の実演は **Notepad を開き、固定メッセージを入力するだけ** とする。
> 実行前に必ず「これは自分の検証用 Windows か?」を声に出して確認する運用にする。

安全装置として、本教材では **アーム用スイッチ(GPIO ジャンパ)** を導入する。
ジャンパを差した時だけペイロードが発火し、普段は単なる `CIRCUITPY` ドライブとして振る舞う設計にする(§6)。

### 0.4 タイムテーブル(目安)
| 時間 | セクション | 内容 |
|---|---|---|
| 0:00-0:30 | §1 | Windows 環境準備・CircuitPython 書き込み・ライブラリ配置 |
| 0:30-0:50 | §2 | コンポジットデバイスの理解 / `boot.py` / Windows での USB 確認 |
| 0:50-1:20 | §3 | Step1 最初のキー入力("Hello") |
| 1:20-2:00 | §4 | Step2 Windows 標的ペイロード(Notepad の固定メッセージデモ) |
| 2:00-2:10 | 休憩 | |
| 2:10-2:45 | §5 | Step3 マウント時自動発火と信頼性 / `autoreload` |
| 2:45-3:10 | §6 | Step4 アーム用スイッチ / ストレージの見せ方 |
| 3:10-3:30 | §7 | レイアウト地雷(US/JIS)と Windows IME |
| 3:30-3:50 | §8 | 防御と検知(守り側)・ディスカッション |
| 予備 | §9 | 応用 / CTF 的発展課題 |

### 0.5 持ち物 / 事前準備(講師)
- RP2040 SECCON バッジ(人数分)+ データ通信対応 USB ケーブル(充電専用ケーブル厳禁)
- 参加者 Windows PC(Windows 10 / 11)。管理者権限があるとドライバ確認やログ確認がしやすい
- CircuitPython UF2 と Adafruit ライブラリバンドルを **オフライン配布**(会場 Wi-Fi を当てにしない)
- REPL 確認用ツール: Mu Editor、Thonny、Tera Term、PuTTY などのいずれか
- ジャンパワイヤ or タクトスイッチ(アーム用、§6)。バッジ上の空きピン/ボタンで代替可
- 講師デモ用の Windows 検証端末。個人端末や業務端末を標的にしない

---

## 1. 環境準備(30分)

### 1.1 CircuitPython を書き込む
1. バッジの **BOOTSEL(ブート)ボタンを押しながら** USB を Windows PC に接続する。
   - バッジ固有のボタン位置はバッジの配布資料で確認。RP2040 の BOOTSEL パッド/ボタン。
2. エクスプローラーに **`RPI-RP2`** という小容量ドライブが現れる。
3. 配布された **CircuitPython の `.uf2`** をそのドライブにドラッグ&ドロップする。
4. 自動的に再起動し、エクスプローラーに **`CIRCUITPY`** という新しいドライブが現れれば成功。

> **UF2 の入手について**
> `circuitpython.org/downloads` でバッジ専用ビルドがあればそれを使う。
> 専用ビルドが無い場合は **Raspberry Pi Pico (RP2040) 汎用ビルド** を候補にする。
> USB HID/MSC の演習だけなら大きな差は出にくいが、GPIO ピン番号はボード資料に合わせて確認する。
> バージョンは **9.x 系の最新安定版** を使うこと(RP2040 は広く対応)。

### 1.2 ライブラリを配置する
`adafruit_hid` を使う。配布バンドルから **`CIRCUITPY\lib\` に `adafruit_hid` フォルダごと** コピーする。

```text
CIRCUITPY\
├── code.py          ← メインで書くファイル(保存すると即実行)
├── boot.py          ← 起動時に一度だけ実行(USB構成の設定はここ)※§2で作る
└── lib\
    └── adafruit_hid\
        ├── __init__.py
        ├── keyboard.py
        ├── keyboard_layout_us.py
        ├── keyboard_layout_base.py
        └── keycode.py
```

Windows のエクスプローラーでは拡張子が非表示になっていることがある。
`code.py.txt` になっていると実行されないため、表示メニューから **ファイル名拡張子** を表示して確認する。

### 1.3 動作確認(REPL)
シリアル REPL に接続し、CircuitPython が応答することを確認する。

Windows では次のいずれかが扱いやすい:
- **Mu Editor**: Mode を CircuitPython にし、Serial を開く
- **Thonny**: インタプリタを CircuitPython / MicroPython 系にし、対応する COM ポートを選ぶ
- **Tera Term / PuTTY**: デバイスマネージャーで確認した `USB Serial Device (COMx)` に 115200 bps で接続する

REPL に入る基本操作:
- 実行中スクリプトを止める: `Ctrl-C`
- プロンプト: `>>>`
- ソフトリセットして `code.py` を再実行: `Ctrl-D`
- 簡単な確認:

```python
import os
os.listdir("/")
```

> **チェックポイント①**: `CIRCUITPY` が見えて、REPL が出れば準備完了。
> ここで詰まる人を全員拾ってから次へ進む。

---

## 2. USB コンポジットデバイスを理解する(20分)

### 2.1 なぜ「USBメモリのつもりがキーボード」になるのか
USB には、1つの物理デバイスが複数の **インターフェース(機能)** を束ねる
**コンポジットデバイス** の仕組みがある。ホストは列挙(enumeration)時に
「このデバイスはストレージ(MSC)も持つし、キーボード(HID)も持つ」と認識する。

- **MSC(Mass Storage Class)**: USBメモリ。Windows では `CIRCUITPY` ドライブとして見える。
- **HID(Human Interface Device)キーボード**: Windows は標準ドライバで受け入れ、接続直後からキー入力を扱える。
- **CDC(USB シリアル)**: REPL 用の COM ポートとして見える。

CircuitPython は起動時、既定で以下を **同時に** 露出できる:
- `CIRCUITPY` ドライブ = **MSC**
- キーボード/マウス/コンシューマ = **HID**
- `USB Serial Device (COMx)` = **CDC**

つまり **「同時エミュレーション」は追加実装なしで成立済み**。
本教材ではこの既定を使い、必要に応じて `boot.py` で構成を調整する。

### 2.2 Windows で USB/HID を確認する
バッジを挿した状態で、Windows 側から次を確認する。

1. **エクスプローラー**
   - `CIRCUITPY` ドライブが見える
   - `code.py`、`boot.py`、`lib` を編集できる
2. **デバイス マネージャー**
   - **ディスク ドライブ** または **ポータブル デバイス** に `CIRCUITPY` 相当のストレージが見える
   - **キーボード** に `HID Keyboard Device` が増える
   - **ポート (COM と LPT)** に `USB Serial Device (COMx)` が見える
   - **ユニバーサル シリアル バス デバイス** に `USB Composite Device` が見えることがある
3. **設定アプリ**
   - Windows 11: 設定 > Bluetooth とデバイス > デバイス
   - Windows 10: 設定 > デバイス

> **理解の確認**:
> 「エクスプローラーでは USB メモリに見える」
> 「デバイス マネージャーではキーボードとしても見える」
> この2つを同時に確認する。

### 2.3 `boot.py` の役割
- `code.py` は起動後・保存時に何度も走る「本体」。
- `boot.py` は **電源投入/ハードリセット時に一度だけ**、しかも **USB がホストに見える前** に走る。
  → **USB の構成(どの機能を出す/隠す)は `boot.py` でしか変えられない**。

まずは「今どんな HID デバイスが出ているか」を確認するだけの `boot.py`:

```python
# boot.py — 現状確認用
import usb_hid
print("HID devices:", [d.usage for d in usb_hid.devices])
```

保存 → バッジの物理リセット(または再挿入)→ `boot_out.txt` または REPL で出力を確認する。

HID をキーボードだけに絞りたい場合は、講師デモとして次のような `boot.py` も扱える。
ただし、参加者が詰まったときの切り戻しを容易にするため、最初は既定構成のままで進める。

```python
# boot.py — 任意: HID をキーボードだけにする例
import usb_hid
usb_hid.enable((usb_hid.Device.KEYBOARD,))
```

---

## 3. Step1 — 最初のキー入力(30分)

### 3.1 最小コード
`code.py` に以下を書いて保存する。
**Notepad を手動で開き、本文入力欄にカーソルを置いてから** 保存すること。

```python
# code.py — Step1: Hello を1回だけ打つ
import time
import usb_hid
from adafruit_hid.keyboard import Keyboard
from adafruit_hid.keyboard_layout_us import KeyboardLayoutUS
from adafruit_hid.keycode import Keycode

kbd = Keyboard(usb_hid.devices)
layout = KeyboardLayoutUS(kbd)

time.sleep(2)               # 保存直後の暴発防止 & ホスト準備待ち
layout.write("Hello from RP2040 SECCON badge!\n")
```

- `layout.write(...)` は文字列を「US配列前提で」キーストロークに変換して送る。
- `\n` は Enter。
- Windows 側の IME が日本語入力モードだと期待通りに入らないことがある。最初は入力モードを **A / 半角英数** にしておく。

### 3.2 修飾キーと単発キー
```python
kbd.send(Keycode.CONTROL, Keycode.A)       # 全選択(Ctrl+A)
kbd.send(Keycode.DELETE)                   # 削除
kbd.press(Keycode.SHIFT)
kbd.press(Keycode.A)
kbd.release_all()                          # "A"
```

- `send()` = 押して離す。`press()`/`release_all()` = 明示制御。
- Windows キーは HID では GUI キーとして扱われる。`Keycode.GUI` または環境によって `Keycode.WINDOWS` を使う。
- 押しっぱなしを `release_all()` で必ず解除する(離し忘れは事故の元)。

### 3.3 演習 3-A
「Notepad に自分のハンドルネームを3行打ち込む」コードを書く。
`layout.write` と `\n`、`time.sleep` を組み合わせる。

> **チェックポイント②**: 全員が「保存 → 自動で文字が打たれる」を1回体験。
> ここで **保存時に毎回発火してうっとうしい** ことに気づかせる(→ §5 の伏線)。

---

## 4. Step2 — Windows 標的ペイロード(40分)

### 4.1 この章で扱う範囲
Windows の BadUSB デモでは、PowerShell やコマンドプロンプトに文字列を流し込む例がよく使われる。
本ワークショップでは、それらは扱わない。

この章の実演は次だけに限定する:
1. Notepad を開く
2. 無害な固定メッセージを入力する
3. それ以上は何もしない

固定メッセージは講師が決め、参加者が任意のコマンド文字列に置き換えない運用にする。

### 4.2 Notepad を開く定石
Windows でアプリを開く方法はいくつかある。
この教材では、わかりやすさと再現性のため **Win+R から `notepad` だけを実行する**。
これは Notepad を開くための無害な操作であり、任意コマンド入力の練習にはしない。

```python
def open_notepad(wait=0.8):
    kbd.send(Keycode.GUI, Keycode.R)   # Win+R: ファイル名を指定して実行
    time.sleep(wait)
    layout.write("notepad")
    time.sleep(0.2)
    kbd.send(Keycode.ENTER)
    time.sleep(1.5)
```

環境によって Win+R が無効化されている場合は、講師判断で Start メニュー検索から Notepad を開くデモに切り替える。
ただし、その場合も起動するアプリは Notepad に限定する。

```python
def open_notepad_from_start(wait=0.8):
    kbd.send(Keycode.GUI)              # Start
    time.sleep(wait)
    layout.write("notepad")
    time.sleep(wait)
    kbd.send(Keycode.ENTER)
    time.sleep(1.5)
```

### 4.3 ペイロードは関数に分ける
```python
SAFE_MESSAGE = (
    "SECCON WS BADUSB SAFE DEMO\n"
    "THIS DEVICE OPENED NOTEPAD AND TYPED ONLY THIS FIXED MESSAGE.\n"
    "NO SHELL OR POWERSHELL WAS USED.\n"
)

def type_safe_message():
    layout.write(SAFE_MESSAGE)

def run_payload():
    open_notepad()
    type_safe_message()
```

メッセージを英字・数字・スペース・ピリオド中心にしているのは、US/JIS 配列差や IME の影響を減らすため。
日本語メッセージは Windows IME の状態に依存するため、ハンズオン本体では扱わない。

### 4.4 演習 4-A(無害デモ)
「Notepad を開いて固定メッセージを3行書く」ペイロードを作る。

```python
import time
import usb_hid
from adafruit_hid.keyboard import Keyboard
from adafruit_hid.keyboard_layout_us import KeyboardLayoutUS
from adafruit_hid.keycode import Keycode

kbd = Keyboard(usb_hid.devices)
layout = KeyboardLayoutUS(kbd)

SAFE_MESSAGE = (
    "SECCON WS BADUSB SAFE DEMO\n"
    "THIS DEVICE OPENED NOTEPAD AND TYPED ONLY THIS FIXED MESSAGE.\n"
    "NO SHELL OR POWERSHELL WAS USED.\n"
)

def open_notepad(wait=0.8):
    kbd.send(Keycode.GUI, Keycode.R)
    time.sleep(wait)
    layout.write("notepad")
    time.sleep(0.2)
    kbd.send(Keycode.ENTER)
    time.sleep(1.5)

def run_payload():
    open_notepad()
    layout.write(SAFE_MESSAGE)

time.sleep(3)
run_payload()
```

> **講師メモ**
> `Win+R` で実行する文字列は `notepad` に固定する。
> 参加者に「ここへ PowerShell コマンドを入れる」ような説明はしない。
> 教育ポイントは「HID 入力は OS に正規キーボード入力として扱われる」ことの理解であり、攻撃コマンドの作成ではない。

### 4.5 “本物の攻撃”は何が違うのか(座学・手は動かさない)
実際の BadUSB 攻撃では、GUI 操作でシェルや管理ツールを開き、短い入力列で危険な処理を実行させることがある。
**構造的には今日の Notepad デモと同じ**で、打ち込む文字列と対象アプリが変わるだけ。

> **ここが最大の教育ポイント**:
> 「HID は無条件に近い形で信頼される」
> 「入力速度は人間の限界を超えられる」
> この2点だけで、画面がアンロックされた端末では多くの GUI 操作が自動化の射程に入る。
> だからこそ **物理ポート管理、画面ロック、未知 HID の制御** が防御線になる(→ §8)。
>
> 本教材では概念説明に留め、**具体的な攻撃コマンドは配布コードにも説明にも含めない**。

---

## 5. Step3 — マウント時に確実に発火させる(35分)

§3 で気づいた「保存のたびに暴発」問題を解決し、
**“挿した瞬間に1回だけ、確実に”** 動くようにする。

### 5.1 ホストの準備を待つ
接続直後は Windows 側の列挙が終わっておらず、早すぎるキー入力は捨てられる。
初回接続時は標準ドライバのセットアップが走るため、余裕を持って長めに待つ。

```python
import time
import supervisor

while not supervisor.runtime.usb_connected:
    time.sleep(0.1)

time.sleep(5)   # Windows の列挙・初回ドライバ準備を吸収するマージン
```

- `time.sleep(5)` は経験則。会場の Windows でばらつくので **5〜8秒で各自調整**。
- 初回だけ失敗し、2回目から成功する場合は、Windows のデバイスセットアップ待ちが原因であることが多い。

### 5.2 開発中の「保存のたび暴発」を止める
`code.py` は保存すると自動リロードされ、そのたびに発火する。開発中はこれを切る。

```python
import supervisor
supervisor.runtime.autoreload = False   # 保存では走らない。走らせたい時は物理リセット
```

- こうすると、**物理リセット/再挿入のときだけ** `code.py` が走る = 実演の「挿した瞬間」に近い挙動になる。
- 「今すぐ試したい」ときは REPL に入って `Ctrl-D`(ソフトリセット)で再実行。
- 参加者が混乱しやすいので、`autoreload=False` を入れた後は「保存しても動かないのが正しい」と説明する。

### 5.3 「1回だけ撃つ」フラグ(任意・仕組みの理解用)
「1度の給電サイクル内で複数回撃たない」ようにしたいときはフラグファイルを使う。

```python
try:
    with open("/fired.flag", "r") as f:
        already = True
except OSError:
    already = False

if not already:
    run_payload()
    try:
        with open("/fired.flag", "w") as f:
            f.write("1")
    except OSError:
        pass
```

> **重要な注意(CircuitPython の書き込み制約)**
> CircuitPython は既定で **ファイルシステムをホスト(Windows)に書き込み許可し、自分(code.py)は読み取り専用** にする。
> そのため `code.py` から `/fired.flag` を書くには、`boot.py` で
> `storage.remount("/", readonly=False)` を実行し **CircuitPython 側を書き込み可・Windows 側を読み取り専用** に切り替える必要がある。
> すると `CIRCUITPY` は Windows から書き込みにくくなるため、演習では必須にしない。
>
> 本ワークショップでは、フラグ方式よりも §6 の GPIO アームスイッチを主な安全装置にする。

> **チェックポイント③**: `autoreload=False` にして、
> 「保存では動かない / 再挿入またはリセットで1回動く」を全員で確認。

---

## 6. Step4 — アーム用スイッチとストレージの見せ方(25分)

### 6.1 なぜアーム(安全装置)が要るか
開発中の自分の Windows で毎回ペイロードが発火すると危険で不便。
**GPIO ピンの状態でペイロードの発火を切り替える** ことで、
「普段はただのUSBメモリ、ジャンパを差した時だけデモモード」にする。

この安全装置は、ワークショップの運用上も重要。
参加者が誤って別の端末に挿した場合でも、未アームなら何も入力しない。

### 6.2 GPIO で発火を制御する
バッジ上の空きピン(またはボタン)を1つ使う。ここでは仮に `GP15` とする。
**実際のピンはバッジの資料で空きピンを確認して置き換える**。

```python
import board
import digitalio

arm = digitalio.DigitalInOut(board.GP15)   # ← バッジの空きピンに合わせて変更
arm.direction = digitalio.Direction.INPUT
arm.pull = digitalio.Pull.UP               # 通常 True。GND に落とす(ジャンパ)と False = 発火

def is_armed():
    return not arm.value                   # GND 接続で armed
```

メイン処理:

```python
while not supervisor.runtime.usb_connected:
    time.sleep(0.1)

time.sleep(5)

if is_armed():
    run_payload()
else:
    pass    # 何もしない = ただの CIRCUITPY ドライブとして振る舞う
```

- タクトスイッチなら「押しながら挿す/リセット」で発火、という運用にできる。
- バッジに LED があれば `armed` 時に点灯させると事故が減る(演習 6-A)。
- 参加者には、発火前に「アーム済み」「標的は検証用 Windows」「Notepad デモのみ」を声に出して確認させる。

### 6.3 ストレージの「顔」を作る(任意)
「USBメモリらしさ」を上げる小ネタ:
- `CIRCUITPY` に **無害なファイル**(`README.txt`、配布資料のコピーなど)を置く。
- ドライブ名は通常 `CIRCUITPY` として見える。名前が不自然であることも守り側の観察ポイントになる。
- `boot.py` で `storage.disable_usb_drive()` を呼べば **ドライブ自体を隠す** ことも可能。
  ただし今回は「USBメモリに見える複合デバイス」を理解する方針なので **隠さない**。

> **演習 6-A**: アーム時に LED 点灯 + 未アーム時は完全に無反応、を実装する。
> 「安全に配れるデモデバイス」を各自完成させるのがこのセクションのゴール。

---

## 7. レイアウト地雷と Windows IME(20分)

### 7.1 US 配列前提という罠
`KeyboardLayoutUS` は **ホストが US 配列である前提** で文字→キーコード変換する。
ホストの Windows が **日本語キーボード(JIS)** や **日本語 IME** の状態だと、
記号(`@ : _ " ( )` など)や一部の入力が **化ける**。
BadUSB が現場で失敗する最頻原因のひとつ。

本教材の固定メッセージを英字・数字・スペース・ピリオド中心にしているのはこのため。

### 7.2 Windows IME の状態
日本語 Windows では、キーボード右下の入力インジケーターが `A` / `あ` のように切り替わる。

- `A` / 半角英数: デモに向く
- `あ` / 日本語入力: ローマ字がかな入力として解釈され、Notepad の結果が変わることがある

演習前に、参加者が手動で入力モードを **A / 半角英数** にしてから進める。
ペイロード側から IME 状態を強制する方法は環境差が大きいため、本教材では扱わない。

### 7.3 US/JIS 差の観察
演習として、次のような記号を含む文字列を Notepad に打ち、崩れ方を観察する。

```python
layout.write('user@example.com "test" (1)\n')
```

観察ポイント:
- 英字と数字は比較的安定する
- 記号は物理配列と入力言語の影響を受ける
- IME が `あ` の状態だと、英字列そのものが期待通りに入らないことがある

> **チェックポイント④**: レイアウト差や IME 状態で入力結果が変わる様子を全員が観測。
> 「BadUSB は万能ではない、前提条件に脆い」ことを理解する。

---

## 8. 守り側 ── 検知と防御(20分・ディスカッション中心)

### 8.1 なぜ止めにくいのか
- HID キーボードは **標準ドライバで動き、ユーザー入力として扱われる**。
- 入力は正規のキーイベントと区別がつきにくい。
- USB メモリに見えるデバイスが、同時にキーボードでもあり得る。
- 画面がアンロックされていると、ユーザー権限で可能な GUI 操作の多くが自動化される。

### 8.2 Windows での検知の観点
- **列挙イベント**: ロック中/離席中に新しい `HID Keyboard Device` や `USB Composite Device` が増える
- **デバイス識別子**: 見慣れない VID/PID、製造元、製品名、シリアル番号
- **入力速度**: 人間離れした高速・無誤打の連続入力
- **相関**: 「USB 挿入 → 数秒後にアプリ起動 → 大量のキー入力」という時系列
- **ユーザー視点の兆候**: 勝手に Notepad が開く、カーソルが高速に動く、入力欄に文字列が流れ込む

### 8.3 Windows で確認できる場所
- **デバイス マネージャー**
  - キーボード: `HID Keyboard Device`
  - ヒューマン インターフェイス デバイス: HID 関連デバイス
  - ユニバーサル シリアル バス デバイス: `USB Composite Device`
  - ポート: `USB Serial Device (COMx)`
- **イベント ビューアー**
  - `Microsoft-Windows-Kernel-PnP`
  - `DeviceSetupManager`
  - `DriverFrameworks-UserMode`
- **EDR / MDM / 資産管理**
  - 新規 USB デバイスの接続履歴
  - 許可されていない HID デバイスの検知
  - ユーザー操作として不自然な高速入力の相関

### 8.4 防御策
- **物理**: 離席時は必ず画面ロック。受付・展示・共有端末では USB ポートを物理的に管理する。
- **運用**: 出所不明の USB を挿さない。拾得 USB を検証端末以外に接続しない。
- **Windows ポリシー**:
  - 既知のキーボードだけを許可するデバイス制御
  - グループポリシーの「デバイスのインストール制限」
  - Microsoft Defender for Endpoint / Intune などによる Device Control
- **監視**:
  - 新規 HID の接続をログ化する
  - ロック解除中に未知 HID が増えた場合に通知する
  - USB 挿入とアプリ起動・高速入力を相関させる

注意点:
- リムーバブルストレージの禁止だけでは HID キーボードを止められない。
- キーボード全体をブロックすると業務影響が大きい。
- 完全な技術的防御だけでなく、物理管理とユーザー教育が重要。

### 8.5 ディスカッション課題
- 「MSC の顔(無害なファイル入り)」は攻撃成功率をどれだけ上げるか?
- 画面ロックはどの程度有効か?
- 企業環境で未知 HID をブロックする場合、利便性とのトレードオフをどこに置くか?
- 展示会・カンファレンス会場での安全な USB 運用はどう設計すべきか?

---

## 9. 応用 / CTF 的発展課題(時間が余ったら / 上級者向け)

1. **クロスプラットフォーム化の設計だけを考える**:
   Windows とそれ以外の OS でアプリ起動キーが異なることを整理する。具体的な攻撃コマンドは書かない。
2. **ステルス構成の議論**:
   `boot.py` で `storage.disable_usb_drive()` を使うと HID だけの構成にできる。今回は実装しないが、守り側がどう気づくかを議論する。
3. **タイミング最適化**:
   Windows の初回デバイスセットアップ、2回目以降、PC の負荷による `time.sleep` の必要値を測る。
4. **守り側チャレンジ(青チーム)**:
   デバイス マネージャーやイベント ビューアーから「このバッジが挿さった」痕跡を探す。
5. **可視化**:
   バッジの LED を使い、待機/アーム/発火の状態を光で表現し、演習の安全性を上げる。

> いずれも **具体的な悪性コマンドは書かせない**。
> 「構造、信頼性、安全装置、検知、防御」の探求に留める。

---

## 付録 A. トラブルシューティング

| 症状 | 原因 | 対処 |
|---|---|---|
| `CIRCUITPY` が出ない | 充電専用ケーブル / UF2 書き込み失敗 / BOOTSEL に入れていない | データ線ありのケーブルに交換 / BOOTSEL 押しながら再接続 / UF2 を再書き込み |
| `RPI-RP2` が出ない | BOOTSEL 操作失敗 / USB ハブ相性 | ボタンを押したまま直挿し / 別ポートを使う |
| REPL の COM ポートが見えない | CDC ドライバ認識待ち / ケーブル不良 | デバイス マネージャーを更新 / 再接続 / ケーブル交換 |
| `ImportError: adafruit_hid` | ライブラリ未配置 / フォルダ階層ミス | `CIRCUITPY\lib\adafruit_hid` を確認 |
| キーが全く打たれない | ホスト準備前に発火 / フォーカスが入力欄に無い / アームされていない | 待ち時間を増やす / Notepad の入力欄へカーソル / GPIO ジャンパ確認 |
| Notepad が開かない | Win+R が無効 / Windows Search や UAC などが前面にある / IME 状態 | 手動で Notepad を開いて Step1 に戻る / Start メニュー方式に切替 / 入力モードを A にする |
| 文字が化ける | US/JIS 配列差 / IME が日本語入力 | 固定メッセージを英数字中心にする / 入力モードを A / 半角英数にする |
| 保存のたびに暴発 | auto-reload | `supervisor.runtime.autoreload = False`(§5.2) |
| 保存しても動かない | `autoreload=False` が効いている | 物理リセット、再挿入、または REPL で `Ctrl-D` |
| `code.py` を書き換えられない | CircuitPython 側を書き込み可に remount している | `boot.py` の `storage.remount` 設定を戻し、物理リセット |
| Windows が警告を出す | 初回デバイス認識 / 組織のデバイス制御 | 講師に確認。管理対象端末では無理に続行しない |

## 付録 B. よく使う Keycode(Windows)

| 操作 | コード |
|---|---|
| Windows キー | `Keycode.GUI` または `Keycode.WINDOWS` |
| Ctrl | `Keycode.CONTROL` |
| Alt | `Keycode.ALT` |
| Shift | `Keycode.SHIFT` |
| Space | `Keycode.SPACEBAR` |
| Enter | `Keycode.ENTER` |
| Esc | `Keycode.ESCAPE` |
| Tab | `Keycode.TAB` |
| Win+R(Notepad 起動に限定) | `send(GUI, R)` |
| 全選択 | `send(CONTROL, A)` |
| コピー | `send(CONTROL, C)` |
| 貼り付け | `send(CONTROL, V)` |

> この表はキー操作の理解用。
> 本ワークショップの配布コードで自動実行するのは Notepad デモだけにする。

## 付録 C. 配布用スターター `code.py`(安全装置つき・Windows / Notepad固定デモ)

```python
# code.py — SECCON WS BadUSB スターター(Windows / Notepad固定デモ)
# 安全方針:
# - GPIO がアームされている時だけ動く
# - 開くアプリは Notepad のみ
# - 入力する文字列は固定メッセージのみ
# - PowerShell、cmd、ダウンロード、削除、設定変更は行わない

import time
import board
import digitalio
import supervisor
import usb_hid
from adafruit_hid.keyboard import Keyboard
from adafruit_hid.keyboard_layout_us import KeyboardLayoutUS
from adafruit_hid.keycode import Keycode

supervisor.runtime.autoreload = False       # 保存では暴発しない

kbd = Keyboard(usb_hid.devices)
layout = KeyboardLayoutUS(kbd)

SAFE_MESSAGE = (
    "SECCON WS BADUSB SAFE DEMO\n"
    "THIS DEVICE OPENED NOTEPAD AND TYPED ONLY THIS FIXED MESSAGE.\n"
    "NO SHELL OR POWERSHELL WAS USED.\n"
)

# --- アーム用スイッチ(GND に落ちていたら発火) ---
arm = digitalio.DigitalInOut(board.GP15)     # TODO: バッジの空きピンに変更
arm.direction = digitalio.Direction.INPUT
arm.pull = digitalio.Pull.UP

def is_armed():
    return not arm.value

def open_notepad(wait=0.8):
    kbd.send(Keycode.GUI, Keycode.R)         # Win+R は Notepad 起動だけに使う
    time.sleep(wait)
    layout.write("notepad")
    time.sleep(0.2)
    kbd.send(Keycode.ENTER)
    time.sleep(1.5)

def run_payload():
    open_notepad()
    layout.write(SAFE_MESSAGE)

# --- メイン ---
while not supervisor.runtime.usb_connected:
    time.sleep(0.1)

time.sleep(5)                                # Windows のデバイス認識待ち

if is_armed():
    run_payload()
# 未アームなら何もしない = ただの CIRCUITPY ドライブ
```

## 付録 D. 任意の `boot.py` 例

### D.1 状態確認用
```python
import usb_hid
print("HID devices:", [d.usage for d in usb_hid.devices])
```

### D.2 HID をキーボードだけにする例
```python
import usb_hid
usb_hid.enable((usb_hid.Device.KEYBOARD,))
```

### D.3 フラグファイル方式を試す場合の remount 例
```python
import storage
storage.remount("/", readonly=False)
```

> D.3 を有効にすると、Windows 側から `CIRCUITPY` に書き込みにくくなる。
> 初心者演習では混乱しやすいため、講師デモまたは上級者向けに留める。

## 付録 E. 講師用チェックリスト

- [ ] UF2 とライブラリバンドルをオフライン配布した
- [ ] Windows 10 / 11 の検証端末を用意した
- [ ] データ通信ケーブルを人数分用意した(充電専用を混ぜない)
- [ ] REPL 用ツール(Mu / Thonny / Tera Term / PuTTY 等)を案内できる
- [ ] 倫理・許可・非破壊/可逆の説明を最初に全員へ行った(§0.3)
- [ ] PowerShell、cmd、任意コマンド、ダウンロード、削除、設定変更を扱わないと明示した
- [ ] デモで開くアプリは Notepad のみに限定した
- [ ] 固定メッセージは英数字中心で、参加者が任意コマンドに置き換えない運用にした
- [ ] 各チェックポイント①〜④で落伍者を拾った
- [ ] GPIO アームスイッチが未アーム時に無反応であることを確認した
- [ ] Windows のデバイス マネージャーで MSC / HID / CDC / USB Composite Device を確認した
- [ ] Windows の防御・検知観点をディスカッションした
- [ ] 最後に「出所不明の USB を挿さない」「離席時は画面ロック」を全員で確認した
