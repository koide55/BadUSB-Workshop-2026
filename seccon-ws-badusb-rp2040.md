# RP2040 SECCON バッジで作る BadUSB ハンズオン
### HID キーボード + USB マスストレージの複合デバイスを CircuitPython で体験する

- **対象**: 中級(Python が読める / ターミナル操作に抵抗がない)
- **所要時間**: 半日(約 3.5〜4 時間、休憩・トラブル対応込み)
- **標的 OS**: macOS(母艦・被害役ともに Mac を想定)
- **題材ボード**: RP2040 搭載 SECCON バッジ基板
- **スタック**: CircuitPython 9.x + `adafruit_hid`

---

## 0. はじめに(講師が最初に読む)

### 0.1 このハンズオンで作るもの
挿すと **「ただのUSBメモリ」に見えるのに、裏で勝手にキーボード入力を送り込む** デバイス。
いわゆる **BadUSB / Rubber Ducky** を、RP2040 バッジで自作する。

USB の「コンポジットデバイス(複合デバイス)」という仕組みを使い、1本のケーブルで
**マスストレージ(MSC)** と **HID キーボード** を同時にホストへ見せるのがミソ。
CircuitPython は起動しただけで `CIRCUITPY` ドライブ(MSC)と HID キーボードの両方を露出するので、
「同時エミュレーション」は特別なことをしなくても既に成立している ── まずこれを腹落ちさせるのが前半のゴール。

### 0.2 学習目標
受講後、参加者は次を説明・実演できる:
1. USB コンポジットデバイスとは何か、なぜ「USBメモリのつもりがキーボード」が成立するのか
2. CircuitPython で HID キーストロークを送る(修飾キー、レイアウト、待ち時間)
3. macOS 標的でアプリ起動・文字入力を自動化する
4. マウント直後に確実に発火させるための信頼性対策
5. なぜ危険か / どう検知・防御するか(守り側の視点)

### 0.3 倫理と前提(**最初に全員へ明示する**)
> 本教材は **自分が所有・管理する機材、または明示的な許可を得た検証環境** に対してのみ使用すること。
> 他人の PC に無断で挿す行為は、たとえ無害なペイロードでも不正指令電磁的記録・不正アクセス等に該当し得る。
> ペイロードは **原則すべて非破壊・可逆(アプリを開く / テキストを打つ程度)** に留める。
> 実行前に必ず「これは自分の Mac か?」を声に出して確認する運用にする。

安全装置として、本教材では **アーム用スイッチ(GPIO ジャンパ)** を導入する。
ジャンパを差した時だけペイロードが発火し、普段は単なるUSBメモリとして振る舞う設計にする(§6)。

### 0.4 タイムテーブル(目安)
| 時間 | セクション | 内容 |
|---|---|---|
| 0:00–0:30 | §1 | 環境準備・CircuitPython 書き込み・ライブラリ配置 |
| 0:30–0:50 | §2 | コンポジットデバイスの理解 / `boot.py` |
| 0:50–1:20 | §3 | Step1 最初のキー入力("Hello") |
| 1:20–2:00 | §4 | Step2 macOS 標的ペイロード(Spotlight でアプリ起動) |
| 2:00–2:10 | 休憩 | |
| 2:10–2:45 | §5 | Step3 マウント時自動発火と信頼性 |
| 2:45–3:10 | §6 | Step4 アーム用スイッチ / ストレージの見せ方 |
| 3:10–3:30 | §7 | レイアウト地雷(JIS/US)と Keyboard Setup Assistant |
| 3:30–3:50 | §8 | 防御と検知(守り側)・ディスカッション |
| 予備 | §9 | 応用 / CTF 的発展課題 |

### 0.5 持ち物 / 事前準備(講師)
- RP2040 SECCON バッジ(人数分)+ データ通信対応 USB ケーブル(充電専用ケーブル厳禁)
- 参加者 Mac(各自)。管理者権限があると望ましい
- CircuitPython UF2 と Adafruit ライブラリバンドルを **オフライン配布**(会場 Wi-Fi を当てにしない)
- ジャンパワイヤ or タクトスイッチ(アーム用、§6)。バッジ上の空きピン/ボタンで代替可

---

## 1. 環境準備(30分)

### 1.1 CircuitPython を書き込む
1. バッジの **BOOTSEL(ブート)ボタンを押しながら** USB を Mac に接続する。
   - バッジ固有のボタン位置はバッジの配布資料で確認。RP2040 の BOOTSEL パッド/ボタン。
2. Finder に **`RPI-RP2`** という小容量ドライブが現れる。
3. 配布された **CircuitPython の `.uf2`** をそのドライブにドラッグ&ドロップ。
4. 自動的に再起動し、**`CIRCUITPY`** という新しいドライブが現れれば成功。

> **UF2 の入手について**
> `circuitpython.org/downloads` でバッジ専用ビルドがあればそれを使う。
> 専用ビルドが無い場合は **Raspberry Pi Pico (RP2040) 汎用ビルド** で USB HID/MSC は問題なく動く
> (GPIO ピン配置はボードで異なるが、今回使うのは USB だけなので影響は小さい)。
> バージョンは **9.x 系の最新安定版** を使うこと(RP2040 は広く対応)。

### 1.2 ライブラリを配置する
`adafruit_hid` を使う。配布バンドルから **`CIRCUITPY/lib/` に `adafruit_hid` フォルダごと** コピーする。

```
CIRCUITPY/
├── code.py          ← メインで書くファイル(保存すると即実行)
├── boot.py          ← 起動時に一度だけ実行(USB構成の設定はここ)※§2で作る
└── lib/
    └── adafruit_hid/
        ├── __init__.py
        ├── keyboard.py
        ├── keyboard_layout_us.py
        ├── keyboard_layout_base.py
        └── keycode.py
```

### 1.3 動作確認(REPL)
シリアルに繋いで CircuitPython の REPL が出ることを確認する。

```bash
# デバイス名は各自で確認(usbmodem 番号は環境依存)
ls /dev/tty.usbmodem*
screen /dev/tty.usbmodem<番号> 115200
```

- REPL に入るには `Ctrl-C`(実行中スクリプトを止める)→ プロンプト `>>>`。
- 抜けるには `Ctrl-A` → `K` → `y`(screen の場合)。
- `tio` や `minicom`、Mu エディタでも可。

> **チェックポイント①**: `CIRCUITPY` が見えて、REPL が出れば準備完了。ここで詰まる人を全員拾ってから次へ。

---

## 2. USB コンポジットデバイスを理解する(20分)

### 2.1 なぜ「USBメモリのつもりがキーボード」になるのか
USB には、1つの物理デバイスが複数の **インターフェース(機能)** を束ねる
**コンポジットデバイス** の仕組みがある。ホストは列挙(enumeration)時に
「このデバイスはストレージ(MSC)も持つし、キーボード(HID)も持つ」と認識する。

- **MSC(Mass Storage Class)**: USBメモリ。ユーザーが警戒しない“顔”。
- **HID(Human Interface Device)キーボード**: OS はドライバ不要で無条件に受け入れ、
  接続直後から任意のキー入力を送れる。**ここが攻撃面**。

CircuitPython は起動時、既定で以下を **同時に** 露出している:
- `CIRCUITPY` ドライブ = **MSC**
- キーボード/マウス/コンシューマ = **HID**
- シリアル = **CDC**

つまり **「同時エミュレーション」は追加実装なしで成立済み**。本教材ではこの既定を使い、
必要に応じて `boot.py` で構成を調整する。

### 2.2 `boot.py` の役割
- `code.py` は起動後・保存時に何度も走る「本体」。
- `boot.py` は **電源投入/ハードリセット時に一度だけ**、しかも **USB がホストに見える前** に走る。
  → **USB の構成(どの機能を出す/隠す)は `boot.py` でしか変えられない**。

まずは「今どんな HID デバイスが出ているか」を確認するだけの `boot.py`:

```python
# boot.py — 現状確認用
import usb_hid
print("HID devices:", [d.usage for d in usb_hid.devices])
```

保存 → バッジの物理リセット(または再挿入)→ REPL / `boot_out.txt` で出力を確認。

> **理解の確認**: 「今この瞬間、Mac から見てこのバッジは “USBメモリ” でもあり “キーボード” でもある」
> を Finder(ドライブ)+ この出力(HID)で二重に確認させる。ここが前半の山場。

---

## 3. Step1 — 最初のキー入力(30分)

### 3.1 最小コード
`code.py` に以下を書いて保存する。**カーソルがテキスト入力できる場所(メモ / TextEdit の新規書類など)に置いてから** 保存すること。

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

### 3.2 修飾キーと単発キー
```python
kbd.send(Keycode.COMMAND, Keycode.A)      # 全選択(⌘A)
kbd.send(Keycode.DELETE)                   # 削除
kbd.press(Keycode.SHIFT); kbd.press(Keycode.A); kbd.release_all()  # "A"
```

- `send()` = 押して離す。`press()`/`release_all()` = 明示制御。
- macOS の ⌘(Command)は **`Keycode.COMMAND`**(= GUI キー)。
- 押しっぱなしを `release_all()` で必ず解除する(離し忘れは事故の元)。

### 3.3 演習 3-A
「メモ.app に自分のハンドルネームを3行打ち込む」コードを書く。
`layout.write` と `\n`、`time.sleep` を組み合わせる。

> **チェックポイント②**: 全員が「保存 → 自動で文字が打たれる」を1回体験。
> ここで **保存時に毎回発火してうっとうしい** ことに気づかせる(→ §5 の伏線)。

---

## 4. Step2 — macOS 標的ペイロード(40分)

### 4.1 Spotlight でアプリを起動する定石
macOS の自動化の基本は **Spotlight(⌘+Space)からアプリ名を打って Enter**。

```python
# 例: 電卓を開く(無害)
kbd.send(Keycode.COMMAND, Keycode.SPACEBAR)   # Spotlight
time.sleep(0.6)
layout.write("Calculator")
time.sleep(0.6)
kbd.send(Keycode.ENTER)
```

- `SPACEBAR` はスペース。`⌘+Space` が Spotlight。
- 各アクションの間に **必ず待ち**(`time.sleep`)を入れる。ウィンドウが出る前に打つと取りこぼす。

### 4.2 ペイロードは関数に分ける
```python
def spotlight(app, wait=0.6):
    kbd.send(Keycode.COMMAND, Keycode.SPACEBAR)
    time.sleep(wait)
    layout.write(app)
    time.sleep(wait)
    kbd.send(Keycode.ENTER)
    time.sleep(1.0)

def type_line(s, wait=0.1):
    layout.write(s)
    kbd.send(Keycode.ENTER)
    time.sleep(wait)
```

### 4.3 演習 4-A(無害デモ)
「TextEdit を開いて『この端末は演習で操作されました』と3行書く」ペイロードを作る。

```python
time.sleep(2)
spotlight("TextEdit")
# 新規書類が開くまで待つ。環境により「開く」ダイアログが出る場合の分岐は §7 で扱う
time.sleep(1.0)
type_line("=== SECCON WS BadUSB demo ===")
type_line("この Mac は演習用デバイスによって自動操作されました。")
type_line("実運用では、ここに任意のコマンドが入り得ます。")
```

### 4.4 “本物の攻撃”は何が違うのか(座学・手は動かさない)
実際の BadUSB 攻撃では、Spotlight から `Terminal` を起動し、1行のシェルコマンド
(バックドア設置・情報送信など)を流し込む。**やることは今日のデモと構造的に同じ**で、
打ち込む文字列が「無害な文章」か「悪意あるコマンド」かの違いしかない。

> **ここが最大の教育ポイント**:
> 「HID は無条件に信頼される」「入力速度は人間の限界を超えられる」という2点だけで、
> GUI 操作可能な全てが自動実行の射程に入る。だからこそ **物理ポートと画面ロックが防御線** になる(→ §8)。
>
> 本教材では概念説明に留め、**具体的な攻撃コマンドは配布コードに含めない**。

---

## 5. Step3 — マウント時に確実に発火させる(35分)

§3 で気づいた「保存のたびに暴発」問題を解決し、
**“挿した瞬間に1回だけ、確実に”** 動くようにする。

### 5.1 ホストの準備を待つ
接続直後は macOS 側の列挙が終わっておらず、早すぎるキー入力は捨てられる。

```python
import supervisor
# USB がホストと接続されるまで待つ
while not supervisor.runtime.usb_connected:
    time.sleep(0.1)
time.sleep(3)   # 列挙・キーボード認識・(初回の)Keyboard Setup Assistant を吸収するマージン
```

- `time.sleep(3)` は経験則。会場の Mac でばらつくので **3〜5秒で各自調整**。

### 5.2 開発中の「保存のたび暴発」を止める
`code.py` は保存すると自動リロードされ、そのたびに発火する。開発中はこれを切る。

```python
import supervisor
supervisor.runtime.autoreload = False   # 保存では走らない。走らせたい時は物理リセット
```

- こうすると、**物理リセット/再挿入のときだけ** `code.py` が走る = 実戦の「挿した瞬間」に近い挙動になる。
- 「今すぐ試したい」ときは REPL に入って `Ctrl-D`(ソフトリセット)で再実行。

### 5.3 「1回だけ撃つ」フラグ(任意・仕組みの理解用)
「1度の給電サイクル内で複数回撃たない」ようにしたいときはフラグファイルを使う。

```python
import storage
try:
    with open("/fired.flag", "r") as f:
        already = True
except OSError:
    already = False

if not already:
    # ... ここでペイロード ...
    try:
        with open("/fired.flag", "w") as f:
            f.write("1")
    except OSError:
        pass  # 書き込めない場合は握りつぶす(下記の注意参照)
```

> **重要な注意(CircuitPython の書き込み制約)**
> CircuitPython は既定で **ファイルシステムをホスト(Mac)に書き込み許可し、自分(code.py)は読み取り専用**。
> そのため `code.py` から `/fired.flag` を書くには、`boot.py` で
> `storage.remount("/", readonly=False)` を実行し **CP 側を書き込み可・ホスト側を読み取り専用** に切り替える必要がある。
> すると「USBメモリ」は Mac からは読み取り専用に見える(多くの実物 USB でも普通のこと)。
> ── この綱引き自体が良い教材なので、§6 のアーム用スイッチと合わせて扱うとよい。
> **フラグ方式は必須ではない**。実戦では「挿す=給電=1回起動」なので、次節のスイッチ制御の方が実用的。

> **チェックポイント③**: `autoreload=False` にして、
> 「保存では動かない / 再挿入(=攻撃シミュレーション)で確実に1回動く」を全員で確認。

---

## 6. Step4 — アーム用スイッチとストレージの見せ方(25分)

### 6.1 なぜアーム(安全装置)が要るか
開発中の自分の Mac で毎回ペイロードが暴れると危険で不便。
**GPIO ピンの状態でペイロードの発火を切り替える** = 実銃の安全装置に相当。
「普段はただのUSBメモリ、ジャンパを差した時だけ攻撃モード」にする。

### 6.2 GPIO で発火を制御する
バッジ上の空きピン(またはボタン)を1つ使う。ここでは仮に `GP15` とする(**バッジの資料で空きピンを確認して置き換える**)。

```python
import board
import digitalio

arm = digitalio.DigitalInOut(board.GP15)   # ← バッジの空きピンに合わせて変更
arm.direction = digitalio.Direction.INPUT
arm.pull = digitalio.Pull.UP               # 通常 True。GND に落とす(ジャンパ)と False = 発火

def is_armed():
    return not arm.value                   # GND 接続で armed

# --- メイン ---
time.sleep(2)
if is_armed():
    run_payload()      # §4 で作ったペイロード
else:
    pass               # 何もしない = ただのUSBメモリとして振る舞う
```

- タクトスイッチなら「押しながら挿す/リセット」で発火、という運用にできる。
- バッジに LED があれば `armed` 時に点灯させると事故が減る(演習 6-A)。

### 6.3 ストレージの「顔」を作る(任意)
「USBメモリらしさ」を上げる小ネタ:
- `CIRCUITPY` に **もっともらしいファイル**(`資料.pdf`、`README.txt` など、無害な中身)を置くだけで、
  受け手の警戒を下げられる ── これが BadUSB の心理的な肝。
- ドライブ名(ボリュームラベル)は CircuitPython では基本 `CIRCUITPY` 固定(変更はビルド依存で本教材の範囲外)。
  「名前で見破れる」ことも守り側の知識として扱う(§8)。
- `boot.py` で `storage.disable_usb_drive()` を呼べば **ドライブ自体を隠す** ことも可能(HID だけのステルス構成)。
  ただし今回は「メモリの顔で油断させる」方針なので **隠さない**。この選択の意味を議論させる。

> **演習 6-A**: アーム時に LED 点灯 + 未アーム時は完全に無反応、を実装する。
> 「安全に配れる BadUSB」を各自完成させるのがこのセクションのゴール。

---

## 7. レイアウト地雷と Keyboard Setup Assistant(20分)

### 7.1 US 配列前提という罠
`KeyboardLayoutUS` は **ホストが US 配列である前提** で文字→キーコード変換する。
ホストの Mac が **日本語(JIS)入力/かな入力** になっていると、
記号(`@ : _ " ( )` など)や一部の文字が **化ける**。BadUSB が現場で失敗する最頻原因のひとつ。

- 実演: `layout.write('user@example.com "test" (1)')` を、
  Mac の入力ソースを「ABC(US)」と「日本語(ローマ字/かな)」で切り替えて打ち比べる。
- 記号が崩れるのを見せ、**「攻撃側はホストのレイアウトを知らないと詰む」** を体感させる。

### 7.2 対策の考え方(座学)
- **英字と数字・限られた記号だけで組む**(記号を避けたペイロード設計)。
- 事前に **入力ソースを US に強制**するキー操作を打つ(ただし確実性は環境依存)。
- コミュニティ製の **JIS 用レイアウト** を使う手もあるが、標準バンドル外。
- 「かな入力」だと英字すら通らないので、**IME/入力モードの状態管理** まで考える必要がある。

### 7.3 macOS 特有: Keyboard Setup Assistant
未知の USB キーボードを挿すと、macOS が
**「キーボード設定アシスタント」**(Shift の右隣のキーを押してと促す ANSI/ISO/JIS 判定ダイアログ)
を出すことがある。これがフォーカスを奪い、初回の発火を妨げる。

- 対策: §5.1 の待ち時間を長めに取る / 2回目以降は出ないことを利用する / デモ機で事前に一度挿しておく。
- 守り側視点では「見慣れないキーボード認識ダイアログ = 不審な HID」の **気づきのサイン** になる(§8)。

> **チェックポイント④**: レイアウト差で記号が化ける様子を全員が観測。
> 「BadUSB は万能ではない、前提条件に脆い」ことを理解する。

---

## 8. 守り側 ── 検知と防御(20分・ディスカッション中心)

### 8.1 なぜ止めにくいのか
- HID キーボードは **ドライバ不要・無条件信頼**。ソフトで「キーボードを拒否」するのは実務上難しい。
- 入力は正規のキーイベントと区別がつかない。

### 8.2 検知の観点
- **入力速度**: 人間離れした高速・無誤打の連続入力。EDR/監視で「機械的タイピング」を検知する研究がある。
- **列挙イベント**: ロック中/離席中に新しいキーボードが増える、見慣れない VID/PID の HID が現れる。
- **相関**: 「USB挿入 → 数秒後に Terminal/PowerShell 起動 → 1行コマンド」という時系列パターン。
- macOS: 予期しない **Keyboard Setup Assistant** の出現、`system_profiler SPUSBDataType` に不審な複合デバイス。

### 8.3 防御策
- **物理**: 離席時は必ず画面ロック(施錠された画面では大半のペイロードが無力)。ポート管理・USBポートの物理封止。
- **ポリシー**: 新規 HID 接続の承認制。Windows なら「新しいキーボードのブロック」系ポリシー、Linux なら **USBGuard**。
  macOS は Apple Silicon の「アクセサリの接続を許可」設定や MDM で USB を制御。
- **運用**: 出所不明の USB を挿さない文化。「拾った USB を挿すな」を体で理解させるのが本ハンズオンの裏の狙い。

### 8.4 ディスカッション課題
- 「MSC の顔(無害なファイル入り)」は攻撃成功率をどれだけ上げるか?
- 完全な防御は可能か? 「利便性 vs 安全」のトレードオフをどこで引くか?

---

## 9. 応用 / CTF 的発展課題(時間が余ったら / 上級者向け)

1. **クロスプラットフォーム化**: 起動キーの叩き方で OS(mac/win/linux)を推定し、分岐して発火する“賢い”ペイロード。
2. **ステルス構成**: `boot.py` で `storage.disable_usb_drive()`、HID だけの最小デバイスにして列挙痕跡を減らす。逆に検知側はどう気づくか。
3. **タイミング最適化**: `time.sleep` を詰めて、Keyboard Setup Assistant を避けつつ最速で撃つ限界を計測。
4. **守り側チャレンジ(青チーム)**: `system_profiler SPUSBDataType` やログから「このバッジが挿さった」痕跡を特定するスクリプトを書く。
5. **可視化**: バッジの LED を使い、待機/アーム/発火の状態を光で表現し、演習の安全性を上げる。

> いずれも **具体的な悪性コマンドは書かせない**。「構造と信頼性・検知回避/検知」の探求に留める。

---

## 付録 A. トラブルシューティング

| 症状 | 原因 | 対処 |
|---|---|---|
| `CIRCUITPY` が出ない | 充電専用ケーブル / UF2 書き込み失敗 | データ線ありのケーブルに / BOOTSEL 押しながら再書き込み |
| キーが全く打たれない | ホスト準備前に発火 / フォーカスが入力欄に無い | `time.sleep` を増やす / 入力欄にカーソル |
| 記号だけ化ける | ホストが JIS/かな入力 | 入力ソースを US(ABC)に / §7 |
| 保存のたびに暴発 | auto-reload | `supervisor.runtime.autoreload = False`(§5.2) |
| `ImportError: adafruit_hid` | ライブラリ未配置 | `lib/adafruit_hid` を確認(§1.2) |
| `code.py` を書き換えられない | CP 側が書き込みロック中 | `boot.py` の `remount` 設定を戻す / 物理リセット |
| 最初だけ変なダイアログ | Keyboard Setup Assistant | 待ち時間増 / 事前に一度挿す(§7.3) |

## 付録 B. よく使う Keycode(macOS)

| 操作 | コード |
|---|---|
| ⌘ Command | `Keycode.COMMAND` |
| ⌥ Option | `Keycode.OPTION`(= ALT) |
| ⌃ Control | `Keycode.CONTROL` |
| ⇧ Shift | `Keycode.SHIFT` |
| Space | `Keycode.SPACEBAR` |
| Enter | `Keycode.ENTER`(= RETURN) |
| Esc | `Keycode.ESCAPE` |
| Tab | `Keycode.TAB` |
| Spotlight | `send(COMMAND, SPACEBAR)` |
| 全選択 | `send(COMMAND, A)` |

## 付録 C. 配布用スターター `code.py`(安全装置つき・穴埋め式)

```python
# code.py — SECCON WS BadUSB スターター(macOS / 無害デモ)
# TODO のところを参加者が埋める
import time, board, digitalio, supervisor
import usb_hid
from adafruit_hid.keyboard import Keyboard
from adafruit_hid.keyboard_layout_us import KeyboardLayoutUS
from adafruit_hid.keycode import Keycode

supervisor.runtime.autoreload = False       # 保存では暴発しない

kbd = Keyboard(usb_hid.devices)
layout = KeyboardLayoutUS(kbd)

# --- アーム用スイッチ(GND に落ちていたら発火) ---
arm = digitalio.DigitalInOut(board.GP15)     # TODO: バッジの空きピンに変更
arm.direction = digitalio.Direction.INPUT
arm.pull = digitalio.Pull.UP

def is_armed():
    return not arm.value

def spotlight(app, wait=0.6):
    kbd.send(Keycode.COMMAND, Keycode.SPACEBAR)
    time.sleep(wait)
    layout.write(app)
    time.sleep(wait)
    kbd.send(Keycode.ENTER)
    time.sleep(1.0)

def run_payload():
    # TODO: ここに無害なデモを書く(例: TextEdit を開いてメッセージを打つ)
    spotlight("TextEdit")
    time.sleep(1.0)
    layout.write("SECCON WS BadUSB demo: this Mac was auto-operated.\n")

# --- メイン ---
while not supervisor.runtime.usb_connected:
    time.sleep(0.1)
time.sleep(3)                                # ホスト準備待ち(各自調整)

if is_armed():
    run_payload()
# 未アームなら何もしない = ただのUSBメモリ
```

## 付録 D. 講師用チェックリスト
- [ ] UF2 とライブラリバンドルをオフライン配布した
- [ ] データ通信ケーブルを人数分用意した(充電専用を混ぜない)
- [ ] 倫理・許可の説明を最初に全員へ行った(§0.3)
- [ ] 各チェックポイント①〜④で落伍者を拾った
- [ ] ペイロードは全員 非破壊・可逆に留まっている
- [ ] 最後に「拾った USB を挿すな」を全員で確認した

