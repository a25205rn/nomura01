# ダイナミックプライシング対応UI 実装手順

`index.html` に、時間だけでなく**在庫残**にも連動して価格が変わるダイナミックプライシングと、その値動きを店主・客の双方が理解できるUIを追加する。

## 1. 現状と変更方針

### 現状（固定スケジュール方式）

| 箇所 | 内容 |
|---|---|
| `index.html:823` `currentPct()` | 時刻のみで 0% / `eveningPct` / `closingPct` の3段階を返す |
| `index.html:828` `currentPrice()` | `sellPrice × (1 - pct/100)` を四捨五入 |
| `index.html:832` `nextDropInfo()` | 次の段階変更時刻を文字列で返す |
| `index.html:369-383` | 値下げスケジュール入力（18時以降 / 19時以降の割引率） |
| `listing.discount` | `{ eveningPct, closingPct }` のみ保持 |

3段階の階段状にしか動かず、売れ行き（在庫の減り方）が価格に反映されない。

### 変更方針

`listing` に **価格モード**を持たせ、既存の `schedule`（従来動作）と新規の `dynamic` を切り替え可能にする。既存の公開済みデータやデモ挙動を壊さないよう、**デフォルトは `schedule` のまま**とする。

ダイナミック時の価格式:

```
price = sellPrice × timeFactor(t) × stockFactor(r)
r = remainingQty / qty   （在庫消化率の残り）

timeFactor(t)  : 15:00→1.00 から 20:00→(1 - maxDiscountPct/100) へ連続的に低下
stockFactor(r) : 残りが少ないほど 1.0 に近づき（値下げを抑える）、
                 売れ残るほど下げ幅を強める。感応度は stockSensitivity で調整
floorPrice     : costPrice × (1 + minMarginPct/100) を下回らないようクランプ
最終値         : 10円単位に丸める
```

在庫が捌けていれば値下げしない／売れ残っていれば早めに下げる、という店主の判断をそのままロジック化する。

## 2. 実装手順

### Step 1 — 価格ロジックの差し替え（JS）

対象: `index.html` の `currentPct()` 〜 `nextDropInfo()` 周辺（823〜843行付近）

1. 定数を追加する。
   ```js
   const PRICE_STEP = 10;          // 丸め単位
   const OPEN_TIME  = 900;         // 15:00（既存の初期値を定数化）
   ```
2. `stockFactor(listing)` を新設。`listing.remainingQty / listing.qty` と `listing.pricing.stockSensitivity` から係数を返す。在庫0のときは0除算を避けて 1.0 を返す。
3. `timeFactor(listing)` を新設。`OPEN_TIME`〜`CLOSE_TIME` を 0〜1 に正規化し、`maxDiscountPct` まで線形に低下させる。
4. `floorPrice(listing)` を新設。`costPrice × (1 + minMarginPct/100)` を返す（原価割れ防止）。
5. `currentPrice(listing)` を、`listing.pricing.mode` で分岐するよう書き換える。
   - `"schedule"` → 既存の `currentPct()` 経由の計算をそのまま呼ぶ
   - `"dynamic"` → 上記3係数を掛け合わせ、`floorPrice` でクランプし `PRICE_STEP` で丸める
6. `currentPct(listing)` は「表示用の割引率」に役割を変更し、ダイナミック時は `currentPrice` から逆算した実効割引率を返す。既存の呼び出し側（バッジ表示）を変更せずに済ませる。
7. `nextDropInfo(listing)` をダイナミック対応にする。段階変更がないため、「30分後の想定価格」を返す形に拡張し、文言を「30分後は◯◯円の見込みです」に切り替える。

> 注意: `currentPrice()` は予約確定時の価格確保（`confirmReservation()`、993行付近）でも使われる。**予約成立時点の価格を `reservation` に固定して保存する既存挙動は必ず維持する**こと。後から価格が動いても予約済みの金額が変わってはいけない。

### Step 2 — listing / draft のデータ構造拡張

対象: `publishBtn` ハンドラ（780行付近）と `generateListingContent()`（667行付近）

1. `draft` および `listing` に `pricing` を追加する。
   ```js
   pricing: {
     mode: "schedule",        // "schedule" | "dynamic"
     maxDiscountPct: 40,      // 閉店時点の最大値引き率
     stockSensitivity: 50,    // 0〜100。在庫残への反応の強さ
     minMarginPct: 10         // 原価に対する最低利益率（フロア）
   }
   ```
2. 既存の `discount: { eveningPct, closingPct }` は `schedule` モード用として**そのまま残す**。
3. `resetRegisterForm()`（806行付近）で `pricing` も初期値に戻す。

### Step 3 — 店主モードUI（HTML + CSS）

対象: `index.html:367-383` の「値下げスケジュール」フィールド

1. フィールド見出しを「値付けの方法」に変更し、上部にモード切替のセグメントボタンを置く。
   ```html
   <div class="pricing-mode-switch">
     <button type="button" class="pmode active" data-pmode="schedule">時間で値下げ</button>
     <button type="button" class="pmode" data-pmode="dynamic">売れ行きに合わせる</button>
   </div>
   ```
2. 既存の割引率グリッドを `<div id="scheduleSettings">` で包む（`schedule` 選択時のみ表示）。
3. `<div id="dynamicSettings" hidden>` を新設し、以下3つのスライダーを置く。数値を隣にライブ表示する。
   - 閉店時の最大値引き（0〜70%）
   - 売れ行きへの反応（弱い〜強い）
   - 最低利益率（0〜40%）
4. CSS はファイル冒頭の `<style>`（7〜283行）末尾に追記する。既存の `.discount-grid` / `.subtab` の配色（藍色基調）とトーンを合わせ、**絵文字は使わない**（README のデザイン方針）。
5. スマホ幅375pxでスライダーがはみ出さないよう、`.pricing-grid` は `grid-template-columns: 1fr` を基本にし、PC幅のみ多カラムにする。

### Step 4 — 価格シミュレーショングラフ（店主モード）

設定した値が「今日一日でどう効くか」を公開前に確認できるようにする。

1. `renderPriceSimulation()` を新設し、プレビューパネル内の `<div id="priceSim">` に**インラインSVG**で描画する（外部CDN禁止のため）。
2. 15:00〜20:00 を5分刻みでサンプリングし、価格推移を折れ線で描く。
3. 在庫が想定どおり捌けた場合／売れ残った場合の2本を重ねて描き、ダイナミック方式の効果を可視化する。
4. 原価ラインとフロア価格を破線で引き、値引きが原価を割らないことを示す。
5. スライダーの `input` イベントで即座に再描画する。
6. `mode === "schedule"` のときは従来の階段状の線を1本だけ描く。

### Step 5 — 客モードUI

対象: `renderListings()`（868行付近）のカード生成部

1. 現在価格の隣に**値動きの理由バッジ**を表示する。
   - 残りが少ない → 「残りわずかのため据え置き中」
   - 売れ残り傾向 → 「売れ行きに合わせて値下げ中」
   - `schedule` モード → 既存の「◯%引き」バッジのまま
2. 価格の下に**ミニスパークライン**（幅100px程度のインラインSVG）を置き、その日の値動きを1本の線で示す。現在時刻の位置に点を打つ。
3. `nextDropInfo()` のカウントダウン表示を、ダイナミック時は「30分後の見込み価格」に差し替える。
4. 取り置きモーダル（`openReserveModal()`、967行付近）の確認文に「**この価格で確保されます。以降値下がりしても、値上がりしても変わりません**」を明記する。価格が連続的に動く以上、確保の意味を従来以上にはっきり書く。

### Step 6 — 時刻スライダーとの連動確認

`timeSlider` の `input` ハンドラ（541行付近）は `renderListings()` を呼ぶだけなので、価格計算が `state.currentTime` を参照している限り追加変更は不要。ただし店主モード側のシミュレーショングラフに**現在時刻の縦線**を引く場合は、ここから `renderPriceSimulation()` も呼ぶ。

### Step 7 — 動作確認（手動チェックリスト）

ビルド不要のため、`index.html` をブラウザで開いて確認する。

- [ ] `schedule` モードで公開 → 従来どおり18時/19時の階段状に価格が変わる（**既存挙動のデグレなし**）
- [ ] `dynamic` モードで公開 → 時刻スライダーを動かすと価格が連続的に下がる
- [ ] 在庫を予約で減らす → 残りが少ないほど値下げが緩やかになる
- [ ] 最低利益率を上げる → フロア価格で下げ止まり、原価を割らない
- [ ] 予約確定後に時刻を進めても、予約一覧の金額が変わらない
- [ ] 数量0・仕入れ値0・希望売価0 でクラッシュしない（0除算の確認）
- [ ] 375px幅でスライダー・グラフ・カードが崩れない
- [ ] PC幅で崩れない
- [ ] 外部通信が発生しない（デモモードでオフライン動作）

### Step 8 — README 更新

`README.md` の「店主モード」節に値付け方法の切替を追記し、「客モード」節に値動きバッジとスパークラインを追記する。「ダイナミックプライシング」の項を新設し、価格式の考え方を数式ではなく言葉で説明する。

## 3. コミットとPRの手順

`origin`（`a26126th-hase/uotarou-demo`）には push 権限がないため、フォーク `fork`（`a25205rn/nomura01`）経由でPRを出す。

```bash
# 現在のブランチ: feature/nomura_01
git add docs/dynamic-pricing-ui.md
git commit -m "Add dynamic pricing UI implementation plan"

# 実装後
git add index.html README.md
git commit -m "Add dynamic pricing UI"

git push -u fork feature/nomura_01
gh pr create --repo a26126th-hase/uotarou-demo \
  --base main --head a25205rn:feature/nomura_01 \
  --title "ダイナミックプライシング対応UIの追加" \
  --body "..."
```

## 4. 技術的な制約（README 準拠、逸脱しないこと）

- 単一HTMLファイルに CSS・JS を内包する。**外部CDN・npmパッケージは追加しない**
- グラフ・スパークラインは**インラインSVGを手書き**する（チャートライブラリ不可）
- データは `localStorage` に保存せず JS 変数に保持する。リロードで初期化される
- 絵文字は使用しない
- 不正入力でクラッシュせず、インラインでエラー表示する
