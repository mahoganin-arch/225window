# 225window
225window
# 日経225 CFD 窓埋め・反転ナンピン戦略 - 完全引き継ぎ文書（最新版）

**最終更新**: 2025年10月11日 18:00  
**プロジェクト**: 日経225 CFD停滞時間窓埋め・反転自動売買戦略  
**ステータス**: 強制決済版分析実施中、TradingView実装準備完了

-----

## 📋 目次

1. [プロジェクト概要](#1-プロジェクト概要)
1. [戦略の背景と狙い](#2-戦略の背景と狙い)
1. [戦略の詳細](#3-戦略の詳細)
1. [分析の完全な変遷](#4-分析の完全な変遷)
1. [最適パラメータ（強制決済版）](#5-最適パラメータ強制決済版)
1. [TradingView実装仕様（確定版）](#6-tradingview実装仕様確定版)
1. [技術的詳細（実装ガイド）](#7-技術的詳細実装ガイド)
1. [次のステップ](#8-次のステップ)

-----

## 1. プロジェクト概要

### 1.1 目的

日経225 CFDの**停滞時間**（先物取引停止時間）に発生する価格乖離（窓）を利用した、**窓埋めナンピン戦略**と**反転ナンピン戦略**の組み合わせによる自動売買システムの構築。

### 1.2 対象市場

- **商品**: 日経225 CFD (`FOREXCOM:JP225`)
- **取引時間**: ほぼ24時間（朝6:00-7:00のみ停止）
- **分析期間**: 2025年5月28日〜10月7日（約4.5ヶ月）
- **データ**: 5分足データで分析、実装は1分足を使用

### 1.3 停滞時間の定義（確定版）

日経225先物の取引停止時間帯をベースとする：

|時間帯   |基準価格取得    |エントリー開始|追跡終了 |強制決済   |
|------|----------|-------|-----|-------|
|**朝** |7:00の直前バー |8:30   |9:45 |9:45終値 |
|**昼** |11:30の直前バー|11:30  |13:30|13:30終値|
|**夕方**|16:00の直前バー|16:00  |18:00|18:00終値|

**重要な実装ルール**:

- 基準価格: 停滞開始時刻の**直前バーの終値**（`close[1]`）
- CFDは停滞時間中も取引継続（価格変動あり）
- 追跡終了時刻に未決済なら**強制決済**（時間損切り）

-----

## 2. 戦略の背景と狙い

### 2.1 市場の特性

**先物取引停止中の特殊性**:

- 日経225先物: 取引停止（流動性ゼロ）
- 日経225 CFD: 取引継続（先物価格から乖離）
- 他の市場: 海外市場の影響を受ける

→ **停滞時間中にCFD価格が先物基準価格から乖離** = 「窓」が発生

### 2.2 窓埋めの原理

**なぜ窓が埋まるのか？**

1. **アービトラージ圧力**: 先物再開後、CFDと先物の価格差を狙う取引
1. **流動性回復**: 先物市場の再開により価格発見機能が正常化
1. **統計的回帰**: 過度な乖離は時間とともに修正される傾向

**データによる裏付け**:

- 総窓数: 272件（分析期間）
- 窓埋め成功率: **95.2%**
- 平均窓サイズ: 約75円

### 2.3 反転の原理

**窓埋め後に反転が起こる理由**:

1. **利食い売り/買い**: 窓埋めトレーダーの決済
1. **オーバーシュート修正**: 窓埋め時の勢いによる行き過ぎの修正
1. **新たなトレンド**: 基準価格到達後の新しい需給バランス

**データによる裏付け**:

- 反転率: 約78-93%（時間帯・曜日により変動）
- 特に月曜朝: 反転率88.9%、平均利益259円

### 2.4 ナンピンと強制決済の狙い

**ナンピンのメリット**:

- 窓が拡大 = 平均取得価格の改善機会
- 最終的に目標到達で全ポジション利益

**強制決済の重要性**:

- 未決済ポジションを放置しない
- リスク管理の明確化
- 実運用との整合性向上

-----

## 3. 戦略の詳細

### 3.1 戦略A: 窓埋めナンピン

#### 基本概念

停滞時間中に発生した窓が拡大していく過程で、段階的にエントリーし、基準価格への回帰で利益を得る。追跡終了時刻（9:45等）までに基準価格到達しない場合は強制決済。

#### エントリーロジック

**上窓の場合**（基準価格40,000円、最適戦略の例）:

```
停滞中の価格変動:
7:00直前 → 基準価格40,000円を記録
7:00-8:30 → 窓を監視（エントリーなし）
8:30 → 40,085円（窓85円） → エントリー1（80円閾値で1ロット）
8:40 → 40,125円（窓125円） → エントリー2（120円閾値で1ロット）
8:50 → 40,165円（窓165円） → エントリー3（160円閾値で1ロット）

保有状況:
- 3ロット（40,080円、40,120円、40,160円で各1ロット）
- 平均取得価格: 40,120円
- 最終エントリー価格: 40,160円

パターン1: 窓埋め成功
9:15 → 40,000円到達（基準価格）
利益 = 80 + 120 + 160 = 360円

パターン2: 損切り
窓が360円まで拡大（最終160円 + 損切り200円）
損失 = 200 × 3ロット = 600円

パターン3: 強制決済（新規）
9:45時点で40,100円（未決済）
損益 = (40,080-40,100) + (40,120-40,100) + (40,160-40,100)
     = -20 + 20 + 60 = +60円（小幅利益）
```

#### パラメータ範囲（拡張版）

|パラメータ  |値                 |
|-------|------------------|
|エントリー回数|1, 2, 3回          |
|エントリー間隔|30, 40, 50円       |
|初回閾値   |50〜150円（10円刻み）    |
|損切り幅   |100〜200円（10円刻み）   |
|損切り基準  |平均価格 or 最終価格      |
|利確     |基準価格到達（固定）        |
|強制決済   |9:45/13:30/18:00終値|

### 3.2 戦略B: 反転ナンピン

#### 基本概念

窓埋め完了後、価格が反転する傾向を利用して逆ポジションを取り、逆行時にナンピンで平均価格を改善する。30分後に未決済なら強制決済。

#### エントリーロジック

**上窓埋め後のロング**（基準価格40,000円、最適戦略の例）:

```
窓埋め完了:
9:10 → 40,000円（窓埋め成功）

反転エントリー（ロング）:
9:10 → 40,000円 → エントリー1（1ロット）
9:15 → 39,970円 → エントリー2（-30円逆行で1ロット買い増し）
9:20 → 39,940円 → エントリー3（-60円逆行で1ロット買い増し）

保有状況:
- 3ロット（40,000円、39,970円、39,940円）
- 平均取得価格: 39,970円
- 最終エントリー価格: 39,940円

パターン1: 利確成功（+50円目標）
9:25 → 40,020円到達
利益 = (40,020 - 40,000) + (40,020 - 39,970) + (40,020 - 39,940)
     = 20 + 50 + 80 = 150円

パターン2: 損切り（-180円、最終価格基準）
39,760円まで下落（最終39,940円 - 180円）
損失 = 180 × 3ロット = 540円

パターン3: 強制決済（新規）
9:40（30分後）時点で39,990円（未決済）
損益 = (39,990 - 40,000) + (39,990 - 39,970) + (39,990 - 39,940)
     = -10 + 20 + 50 = +60円（小幅利益）
```

#### パラメータ範囲（拡張版）

|パラメータ  |値                |
|-------|-----------------|
|エントリー回数|1, 2, 3回         |
|エントリー間隔|30, 40, 50円（逆行時） |
|利確目標   |+10〜100円（10円刻み）  |
|損切り幅   |-100〜-200円（10円刻み）|
|損切り基準  |平均価格 or 最終価格     |
|強制決済   |30分後の終値          |

-----

## 4. 分析の完全な変遷

### 4.1 第1版：厳密版（初期実装）

**特徴**:

- 基準価格: 特定時刻の終値（5:55、11:25、15:40）
- 朝のエントリー: 8:30-8:45の15分間のみ
- 月曜朝: 土曜5:55終値を基準
- 未決済ポジション: 分析から除外

**結果**:

- 窓埋め最適（2位選択）: 純利益27,310円
- 反転最適（4位選択）: 純利益6,270円
- 合計: 33,580円

**問題点**:

- TradingView実装が複雑
- 祝日対応が困難
- 未決済ポジションが評価されない

### 4.2 第2版：柔軟版（改良実装）

**変更点**:

- 基準価格: 停滞開始バーの直前バーの終値
- 朝のエントリー: 8:30-9:45に延長（75分）
- 曜日判定不要

**結果**:

- 窓埋め: 純利益29,170円（厳密版より改善）
- エントリー機会が増加

**発見されたバグ**:

- 追跡期間の計算ミス（8:45-8:45）
- エントリー期間が8:45で切れていた

### 4.3 第3版：柔軟版（修正済）

**修正内容**:

- エントリー期間を9:45まで正しく延長
- 追跡期間のバグを修正
- 損切り判定も延長期間で実施

**結果**:

- さらなる改善を確認

### 4.4 第4版：拡張版

**パラメータ範囲を拡大**:

- 初回閾値: 50〜150円（従来は50〜100円）
- 間隔: 30, 40, 50円（従来は10, 20, 30円）
- 損切り: 100〜200円（従来は30〜120円）

**結果**（強制決済前）:

- 窓埋め最適: 純利益**37,480円**（間隔40円、初回80円）
- 反転最適: 純利益**7,080円**（間隔30円、利確50円）
- 合計: **44,560円**

**重要な発見**:

- より大きな窓からのエントリーが有効
- 広い間隔と広い損切り幅が優位

### 4.5 第5版：強制決済版（最新・実施中）

**最終的な改善**:

- 追跡終了時刻（9:45等）で未決済なら**終値で強制決済**
- 反転戦略も30分後に未決済なら**終値で強制決済**
- すべてのトレードを正確に評価

**期待される効果**:

- より実運用に近い評価
- 未決済ポジションのリスクを可視化
- 純利益の正確な算出

**現在の状況**:

- 分析実施中（推定30-40分）
- 結果次第でTradingView実装へ

-----

## 5. 最適パラメータ（強制決済版）

### 5.1 窓埋めナンピン（拡張版・強制決済前）

**パラメータ**:

|項目     |値                 |
|-------|------------------|
|エントリー回数|3回                |
|エントリー間隔|40円               |
|初回閾値   |80円               |
|2回目閾値  |120円              |
|3回目閾値  |160円              |
|損切り幅   |200円              |
|損切り基準  |最終エントリー価格         |
|利確     |基準価格到達            |
|強制決済   |9:45/13:30/18:00終値|

**成績**（強制決済前）:

- 純利益: **37,480円**
- 総利益: 未公開
- 総損失: 未公開
- 成功率: 未公開

**強制決済版の結果待ち**

### 5.2 反転ナンピン（拡張版・強制決済前）

**パラメータ**:

|項目     |値        |
|-------|---------|
|エントリー回数|3回       |
|エントリー間隔|30円（逆行時） |
|利確目標   |+50円（一括） |
|損切り幅   |-180円    |
|損切り基準  |最終エントリー価格|
|強制決済   |30分後の終値  |

**成績**（強制決済前）:

- 純利益: **7,080円**
- 総利益: 未公開
- 総損失: 未公開
- 成功率: 未公開

**強制決済版の結果待ち**

### 5.3 複合戦略の期待値（強制決済前）

```
窓埋めナンピン: 37,480円
反転ナンピン:     7,080円
─────────────────────
合計純利益:     44,560円（分析期間4.5ヶ月）

月平均:        約9,902円
トレード期待値: 約164円/窓
```

**強制決済版で数値が変わる可能性あり**

-----

## 6. TradingView実装仕様（確定版）

### 6.1 基本設計

**実装方式**: Pine Script v6  
**スクリプトタイプ**: Strategy（バックテスト + リアルタイムアラート）  
**チャート設定**: 1分足（ユーザー指定）  
**タイムゾーン**: `Asia/Tokyo`

### 6.2 バックテスト設定（確定）

```pine
strategy("日経225 窓埋め・反転ナンピン戦略", 
         overlay=true,
         initial_capital=500000,
         default_qty_type=strategy.fixed,
         default_qty_value=1,
         commission_type=strategy.commission.cash_per_order,
         commission_value=352,
         slippage=0)
```

- **初期資金**: 500,000円
- **ポジションサイズ**: 各エントリー1枚（固定）
- **手数料**: 1取引あたり352円
- **スリッページ**: 0（検証後に調整）

### 6.3 必須機能

#### 1. 時間判定（確定版）

**基準価格の取得**:

```pine
// 7:00のバーを検出
if hour == 7 and minute == 0
    base_price := close[1]  // 直前バーの終値
```

**メリット**:

- シンプルで確実
- 分析コードとの整合性100%
- データ欠損に強い
- 祝日対応不要

#### 2. 個別エントリー追跡

```pine
var float entry_price_1 = na
var float entry_price_2 = na
var float entry_price_3 = na
var int position_count = 0

// 個別エントリー（窓埋め）
if gap_size >= 80 and na(entry_price_1)
    strategy.entry("gap_fill_1", strategy.short, qty=1)
    entry_price_1 := high
    position_count := position_count + 1
    alert("窓埋めエントリー1: 80円でショート1枚")

if gap_size >= 120 and na(entry_price_2)
    strategy.entry("gap_fill_2", strategy.short, qty=1)
    entry_price_2 := high
    position_count := position_count + 1
    alert("窓埋めエントリー2: 120円でショート追加1枚")

if gap_size >= 160 and na(entry_price_3)
    strategy.entry("gap_fill_3", strategy.short, qty=1)
    entry_price_3 := high
    position_count := position_count + 1
    alert("窓埋めエントリー3: 160円でショート追加1枚")
```

#### 3. 損益管理

**利確（基準価格到達）**:

```pine
if position_count > 0 and low <= base_price
    strategy.close_all()
    alert("窓埋め利確: 基準価格到達、全決済")
    // リセット処理
```

**損切り（最終価格+200円）**:

```pine
last_price = entry_price_3
stop_loss_line = last_price + 200

if position_count > 0 and high >= stop_loss_line
    strategy.close_all()
    alert("窓埋め損切り: 損切りライン到達、全決済")
    // リセット処理
```

**強制決済（9:45時刻）**:

```pine
if position_count > 0 and hour == 9 and minute == 45
    strategy.close_all()
    alert("窓埋め強制決済: 時間終了、全決済")
    // リセット処理
```

#### 4. 反転戦略（オプション）

```pine
// パラメータで有効/無効を切り替え
enable_reversal = input.bool(true, "反転戦略を有効化")

// 窓埋め完了時にフラグ設定
var bool gap_filled = false

if position_count > 0 and low <= base_price
    gap_filled := true
    reversal_base := close

// 反転エントリー
if enable_reversal and gap_filled
    // 3回のナンピンエントリー
    // 30分後の強制決済
```

#### 5. アラートシステム

**各エントリー時**:

- “窓埋めエントリー1: FOREXCOM:JP225 上窓80円でショート1枚”
- “窓埋めエントリー2: FOREXCOM:JP225 上窓120円でショート追加1枚”
- “窓埋めエントリー3: FOREXCOM:JP225 上窓160円でショート追加1枚”

**決済時**:

- “窓埋め利確: 基準価格到達、全決済”
- “窓埋め損切り: 損切りライン到達、全決済”
- “窓埋め強制決済: 時間終了、全決済”

**反転時**:

- “反転エントリー1: FOREXCOM:JP225 窓埋め後ロング1枚”
- “反転エントリー2: FOREXCOM:JP225 ロング追加1枚（逆行）”
- “反転利確: 利確目標到達、全決済”
- “反転損切り: 損切りライン到達、全決済”
- “反転強制決済: 30分経過、全決済”

#### 6. 視覚表示

```pine
// エントリーライン
plot(base_price + 80, "エントリー1", color=color.blue, linewidth=1)
plot(base_price + 120, "エントリー2", color=color.blue, linewidth=1)
plot(base_price + 160, "エントリー3", color=color.blue, linewidth=1)

// 損切りライン
plot(last_price + 200, "損切り", color=color.red, linewidth=2)

// 利確ライン（基準価格）
plot(base_price, "利確（基準価格）", color=color.green, linewidth=2)

// 窓の範囲（背景色）
bgcolor(in_stagnation ? color.new(color.yellow, 90) : na)
```

### 6.4 パラメータ設定

```pine
// 窓埋めナンピン
init_threshold = input.int(80, "初回閾値", minval=50, maxval=150)
interval = input.int(40, "エントリー間隔", minval=30, maxval=50)
num_entries = input.int(3, "エントリー回数", minval=1, maxval=3)
stop_loss = input.int(200, "損切り幅", minval=100, maxval=200)
stop_basis = input.string("最終", "損切り基準", options=["平均", "最終"])

// 反転ナンピン
enable_reversal = input.bool(true, "反転戦略を有効化")
reversal_interval = input.int(30, "反転エントリー間隔", minval=30, maxval=50)
reversal_profit = input.int(50, "反転利確目標", minval=10, maxval=100)
reversal_stop = input.int(180, "反転損切り幅", minval=100, maxval=200)

// 表示設定
show_lines = input.bool(true, "エントリー/損切りラインを表示")
show_background = input.bool(true, "停滞時間の背景色を表示")
```

-----

## 7. 技術的詳細（実装ガイド）

### 7.1 時間判定ロジック（完全版）

#### 朝の処理フロー

```pine
// グローバル変数
var float base_price = na
var bool in_stagnation = false
var bool can_entry = false
var int stagnation_period = 0  // 1=朝, 2=昼, 3=夕方

// 7:00のバーで基準価格を記録
if hour == 7 and minute == 0
    base_price := close[1]  // 直前バーの終値
    in_stagnation := true
    can_entry := false
    stagnation_period := 1

// 8:30でエントリー開始
if hour == 8 and minute == 30
    can_entry := true

// 9:45で強制決済
if hour == 9 and minute == 45
    if position_count > 0
        strategy.close_all()
        alert("窓埋め強制決済: 時間終了")
    
    // リセット
    in_stagnation := false
    can_entry := false
    base_price := na
    // エントリー価格もリセット
```

#### 昼と夕方（同様のパターン）

```pine
// 昼: 11:30-13:30
if hour == 11 and minute == 30
    base_price := close[1]
    in_stagnation := true
    can_entry := true  // 昼は即座にエントリー可能
    stagnation_period := 2

if hour == 13 and minute == 30
    if position_count > 0
        strategy.close_all()
    // リセット

// 夕方: 16:00-18:00
if hour == 16 and minute == 0
    base_price := close[1]
    in_stagnation := true
    can_entry := true
    stagnation_period := 3

if hour == 18 and minute == 0
    if position_count > 0
        strategy.close_all()
    // リセット
```

### 7.2 窓サイズの計算

```pine
// 上窓（ショート）
gap_size_up = high - base_price

// 下窓（ロング）
gap_size_down = base_price - low

// 窓の方向を判定
is_up_window = gap_size_up > gap_size_down
gap_size = is_up_window ? gap_size_up : gap_size_down

// 閾値チェック（50円以上）
has_gap = gap_size >= 50
```

### 7.3 エントリー管理（完全版）

```pine
// 変数定義
var float entry_price_1 = na
var float entry_price_2 = na
var float entry_price_3 = na
var int position_count = 0
var bool is_up_window_active = false

// エントリー判定（上窓の場合）
if can_entry and has_gap and is_up_window

    // エントリー1（80円）
    if gap_size >= 80 and na(entry_price_1)
        strategy.entry("gap_fill_1", strategy.short, qty=1)
        entry_price_1 := base_price + 80
        position_count := position_count + 1
        is_up_window_active := true
        alert("窓埋めエントリー1: 80円でショート1枚", alert.freq_once_per_bar)
    
    // エントリー2（120円）
    if gap_size >= 120 and na(entry_price_2)
        strategy.entry("gap_fill_2", strategy.short, qty=1)
        entry_price_2 := base_price + 120
        position_count := position_count + 1
        alert("窓埋めエントリー2: 120円でショート追加1枚", alert.freq_once_per_bar)
    
    // エントリー3（160円）
    if gap_size >= 160 and na(entry_price_3)
        strategy.entry("gap_fill_3", strategy.short, qty=1)
        entry_price_3 := base_price + 160
        position_count := position_count + 1
        alert("窓埋めエントリー3: 160円でショート追加1枚", alert.freq_once_per_bar)
```

### 7.4 決済管理（完全版）

```pine
// 平均価格と最終価格の計算
calc_avg_price() =>
    count = 0
    sum = 0.0
    if not na(entry_price_1)
        sum := sum + entry_price_1
        count := count + 1
    if not na(entry_price_2)
        sum := sum + entry_price_2
        count := count + 1
    if not na(entry_price_3)
        sum := sum + entry_price_3
        count := count + 1
    count > 0 ? sum / count : na

avg_price = calc_avg_price()
last_price = not na(entry_price_3) ? entry_price_3 : 
             not na(entry_price_2) ? entry_price_2 : 
             entry_price_1

// 損切り基準の選択
stop_ref = stop_basis == "平均" ? avg_price : last_price

// 上窓（ショート）の場合
if position_count > 0 and is_up_window_active
    
    // 利確判定（基準価格到達）
    if low <= base_price
        strategy.close_all()
        alert("窓埋め利確: 基準価格到達、全決済", alert.freq_once_per_bar)
        // リセット処理
        entry_price_1 := na
        entry_price_2 := na
        entry_price_3 := na
        position_count := 0
        is_up_window_active := false
    
    // 損切り判定
    else if not na(stop_ref) and high >= stop_ref + stop_loss
        strategy.close_all()
        alert("窓埋め損切り: 損切りライン到達、全決済", alert.freq_once_per_bar)
        // リセット処理
```

### 7.5 反転戦略（完全版）

```pine
// 反転用の変数
var bool gap_filled = false
var float reversal_base = na
var float reversal_entry_1 = na
var float reversal_entry_2 = na
var float reversal_entry_3 = na
var int reversal_position = 0
var int reversal_start_time = 0

// 窓埋め完了を検出
if position_count > 0 and is_up_window_active and low <= base_price
    gap_filled := true
    reversal_base := close
    reversal_start_time := time
    // 窓埋めポジションは既に決済済み

// 反転エントリー（上窓埋め後のロング）
if enable_reversal and gap_filled and is_up_window_active
    
    // エントリー1（即座）
    if na(reversal_entry_1)
        strategy.entry("reversal_1", strategy.long, qty=1)
        reversal_entry_1 := close
        reversal_position := reversal_position + 1
        alert("反転エントリー1: 窓埋め後ロング1枚", alert.freq_once_per_bar)
    
    // エントリー2（-30円逆行）
    if low <= reversal_base - 30 and na(reversal_entry_2)
        strategy.entry("reversal_2", strategy.long, qty=1)
        reversal_entry_2 := reversal_base - 30
        reversal_position := reversal_position + 1
        alert("反転エントリー2: ロング追加1枚", alert.freq_once_per_bar)
    
    // エントリー3（-60円逆行）
    if low <= reversal_base - 60 and na(reversal_entry_3)
        strategy.entry("reversal_3", strategy.long, qty=1)
        reversal_entry_3 := reversal_base - 60
        reversal_position := reversal_position + 1
        alert("反転エントリー3: ロング追加1枚", alert.freq_once_per_bar)
    
    // 反転の平均価格計算
    rev_avg = (reversal_entry_1 + 
               (not na(reversal_entry_2) ? reversal_entry_2 : 0) + 
               (not na(reversal_entry_3) ? reversal_entry_3 : 0)) / 
              reversal_position
    
    rev_last = not na(reversal_entry_3) ? reversal_entry_3 : 
               not na(reversal_entry_2) ? reversal_entry_2 : 
               reversal_entry_1
    
    rev_stop_ref = stop_basis == "平均" ? rev_avg : rev_last
    
    // 利確判定（+50円）
    if high >= rev_avg + 50
        strategy.close_all()
        alert("反転利確: 利確目標到達、全決済", alert.freq_once_per_bar)
        // リセット
    
    // 損切り判定（-180円）
    else if low <= rev_stop_ref - 180
        strategy.close_all()
        alert("反転損切り: 損切りライン到達、全決済", alert.freq_once_per_bar)
        // リセット
    
    // 強制決済（30分経過）
    else if (time - reversal_start_time) >= 30 * 60 * 1000
        strategy.close_all()
        alert("反転強制決済: 30分経過、全決済", alert.freq_once_per_bar)
        // リセット
```

-----

## 8. 次のステップ

### 8.1 即座に実行（現在進行中）

1. ✅ **強制決済版の分析完了**（実施中）
- 未決済ポジションを正確に評価
- 最終的な最適パラメータを確定
1. 🔄 **Pine Script v6コード作成**（次のタスク）
- 強制決済版の結果を反映
- 個別エントリー追跡
- アラートシステム
- バックテスト機能
1. 🧪 **バックテストと検証**
- TradingViewでの動作確認
- 分析結果との整合性チェック
- 異なる期間でのテスト

### 8.2 中期的な改善項目

1. **パラメータの微調整**
- 強制決済のタイミング検証
- 手数料・スリッページの調整
- 実トレードとの比較
1. **リスク管理の強化**
- 最大ドローダウン制限
- 1日あたりの最大損失制限
- 連続損失後の一時停止
1. **実運用準備**
- デモ口座での検証（最低1ヶ月）
- スリッページの実測
- 約定確率の確認
- 資金管理ルールの策定

### 8.3 長期的な拡張

1. **戦略の拡張**
- 曜日・時間帯別のパラメータ調整
- ボラティリティフィルター
- 複数時間足の分析
1. **自動売買システム化**
- ブローカーAPI連携
- 完全自動実行
- モニタリングシステム

-----

## 9. リスクと注意事項

### 9.1 戦略固有のリスク

**ナンピンのリスク**:

- 資金負担が大きい（最大3ロット保有）
- 窓が想定以上に拡大すると大損失
- 証拠金不足の可能性

**強制決済のリスク**:

- タイミングが悪いと不利な価格で決済
- 直後に目標到達する可能性
- 機会損失

**対策**:

- 1トレードのリスクを資金の2%以内に
- 十分な証拠金を確保（推奨3倍以上）
- ポジションサイズを適切に設定
- 強制決済時刻の見直し（必要に応じて）

### 9.2 市場環境の変化

**想定外のイベント**:

- 大規模な市場ショック
- 規制変更（CFD取引の制限）
- 取引時間の変更
- 流動性の枯渇

**対策**:

- 重要イベント前は取引停止
- 損切りを確実に実行
- 定期的なパフォーマンス見直し
- 市場環境の変化を監視

### 9.3 技術的リスク

**システム障害**:

- 接続エラー
- 約定遅延
- データフィード停止
- TradingViewのダウンタイム

**対策**:

- 複数の通信手段
- 手動決済の準備
- 緊急時の対応手順
- バックアップシステム

-----

## 10. 参考資料

### 10.1 データソース

- GitHub: https://github.com/mahoganin-arch/225window
- 24ファイルのCSVデータ（5分足）
- 期間: 2025年5月28日〜10月7日

### 10.2 分析コード

- **厳密版**: Google Colab（初期実装）
- **柔軟版**: Google Colab（修正済）
- **拡張版**: Google Colab（パラメータ拡張）
- **強制決済版**: Google Colab（最新・実施中）

### 10.3 評価パターン数

- 窓埋めナンピン: 約2,178パターン
- 反転ナンピン: 約1,980パターン
- 合計: 約4,158パターン

### 10.4 使用ツール

- **分析**: Python 3.x, pandas, numpy
- **実装**: TradingView Pine Script v6
- **バックテスト**: TradingView Strategy機能

-----

## 11. バージョン履歴

|日付              |バージョン|変更内容                     |
|----------------|-----|-------------------------|
|2025/10/11 09:00|v1.0 |初版作成（厳密版）                |
|2025/10/11 12:00|v1.1 |柔軟版追加                    |
|2025/10/11 14:00|v1.2 |柔軟版バグ修正                  |
|2025/10/11 16:00|v1.3 |拡張版追加                    |
|2025/10/11 18:00|v2.0 |強制決済版追加、TradingView実装仕様確定|

-----

## 12. 連絡先・サポート

**プロジェクトリード**: [担当者名]  
**技術サポート**: [連絡先]  
**緊急連絡**: [緊急時の連絡先]

-----

*本文書は日経225 CFD窓埋め・反転ナンピン戦略の完全な技術仕様と実装ガイドです。*  
*強制決済ルールを含む最新の分析結果を反映しています。*  
*実運用前に必ずデモ口座での十分な検証を実施してください。*

**最終更新**: 2025年10月11日 18:00  
**ステータス**: 強制決済版分析実施中、Pine Script実装準備完了  
**次のアクション**: 分析結果確認後、Pine Script v6コード作成
