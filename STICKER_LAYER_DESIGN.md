# STICKER LAYER 設計

対象範囲: **04 // STICKER LAYER** メニュー全体（BOMB範囲、透明度、自由変形、オフセット、透明マスク、Undo/Redo）の仕様。

## 1. レイヤー構造（キャンバス内 z-index 順、下→上）

```
canvas-container (overflow:hidden, position:relative)
├ bg-image-layer (z:0)      背景画像（transform: translate + rotate + size）
├ dummy-bg-layer (z:0)      ダミーボム画像
├ sticker-layer (z:1)       ★canvas要素★ ステッカー画像 + マスク統合
│  └ transform: matrix3d(ft) * scale(sc)  （自由変形 + 拡大）
│  └ 内部描画: drawImage で全ステッカーを合成
│  └ マスクは同じcanvasに destination-out で穴あけ（同じ座標系で完全一致）
├ selected-sticker DOM (z:9990)  選択中ステッカーのハンドル付き要素
└ ※注: target-rect / target-indicator / ft-point / ft-outline は
       canvas-wrapper 直下（z:7400〜7500、モバイルパネル7999より下）
```

### 重要方針
- **sticker-layer は canvas要素**（img ではない）
- ステッカー合成とマスクを**同じcanvas内**で処理 → 座標系が完全一致、変形と同期する
- マスクはキャンバス座標系（canvasSize）で描画。transform は canvas要素に CSS で適用
- これによりマスクはsticker画像と一体で変形される

## 2. 状態モデル

### 2.1 BOMB範囲（全体 / 中心 / その他）

変数:
- `targetArea`: 'all' | 'center' | 'custom'
- `customTarget`: 1点指定時の中心 `{x,y}` | null
- `customDragRect`: ドラッグ範囲 `{x,y,w,h}` | null
- `customVisible`: 枠の表示フラグ（bool）

ルール:
- ラジオボタン（3つから1つ選択）
- その他 active の時のみ custom の枠が操作可能
- `customVisible` が枠表示/非表示を制御（データは保持）

### 2.2 自由変形

変数:
- `ftState`: 'idle' | 'picking' | 'editing'
- `ftInitial`: クリック時の4点（変形の基準）
- `ftPoints`: 現在の4点（ハンドルの位置）
- `ftMatrix`: matrix3d文字列
- `ftVisible`: 枠の表示フラグ（editing 中のみ意味を持つ）

ルール:
- idle: ボタン「始点を選択」（消灯）
- picking: ボタン「4点を選択 (N/4)」（点灯）、キャンバスで4点クリック
- editing + ftVisible=true: ハンドル表示、画像変形中
- editing + ftVisible=false: ハンドル非表示、画像は変形維持
- 変形は ftMatrix で常に適用（ftVisible と独立）

### 2.3 透明マスク

変数:
- `eraserMode`: bool（マスクツールON/OFF）
- `eraserOp`: 'mask' | 'unmask'（どちらの操作か）
- `eraserPaths`: `[{x, y, r, mode}]`（描画履歴。x,y はキャンバス座標）

ルール:
- マスクボタン: ON時 eraserMode=true, eraserOp='mask'
- アンマスクボタン: ON時 eraserMode=true, eraserOp='unmask'
- 両方とも同一ボタン再タップで OFF
- 排他（mask と unmask は同時ONにできない）

### 2.4 透明度・オフセット

- `stickerOpacity`: 0-100（sticker-layer の opacity CSS）
- `offsetX`, `offsetY`: -500〜500（sticker-layer の translate）

## 3. 座標系

| 名前 | 単位 | 範囲 | 用途 |
|---|---|---|---|
| 内部座標 (canvasSize) | 整数 | 0〜canvasSize.w/h | stickers配置、customTarget、ftPoints、eraserPaths |
| 画面座標 (clientX/Y) | CSS px | ビューポート基準 | pointerdown イベント |
| 要素相対座標 | CSS px | canvas.getBoundingClientRect基準 | 逆変換の中間 |

### 変換関数

```
screenToInternal(clientX, clientY) → {x, y} (内部座標)
  1. (px, py) = clientX/Y - canvasRect.left/top
  2. layer に transform があれば DOMMatrix.inverse().transformPoint で逆変換
  3. ★ perspective対応: 結果の同次座標 (x/w, y/w) を取り出す ★
  4. 結果を canvasSize にスケール: x / canvasPxW * canvasSize.w
```

**重要**: perspective 変換時、`transformPoint` の結果は同次座標 `(x, y, z, w)` で返るため、`w != 1` の可能性がある。`x/w`, `y/w` で正規化しないと斜めに進むなどの座標ズレが発生する。

### matrix3d 計算

- computeMatrix3d(srcInternal, dstInternal) → matrix3d文字列
- **内部座標 → CSS px に変換してから計算**（要素サイズと一致させる）
- ftMatrix と scale3d は **DOMMatrix で単一の matrix3d に合成**（`.toString()`）

## 4. ボタン動作

### 4.1 BOMB範囲ボタン

| 現状態 | 操作 | 新状態 | 表示 |
|---|---|---|---|
| all/center | 別ボタン押す | その target へ | custom選択時、customData あれば枠復元 |
| custom (visible) | custom再押下 | custom (hidden) | customVisible=false、枠非表示、データ保持 |
| custom (hidden) | custom再押下 | custom (visible) | customVisible=true、枠復元 |
| * | 全体/中心押す | 切替 | custom枠は非表示にするがデータ保持 |

### 4.2 自由変形ボタン

| 現状態 | 操作 | 新状態 |
|---|---|---|
| idle | ボタン押す | picking |
| picking | ボタン押す | idle（キャンセル） |
| picking | 4点クリック完了 | editing + ftVisible=true |
| editing (visible) | ボタン押す | editing (visible=false)、枠非表示 |
| editing (hidden) | ボタン押す | editing (visible=true)、枠復元 |
| editing (*) | リセットボタン | idle、データクリア、変形リセット |

### 4.3 マスクボタン

| 現状態 | 操作 | 新状態 |
|---|---|---|
| off | マスク押す | on, op='mask' |
| off | アンマスク押す | on, op='unmask' |
| on (mask) | マスク再押下 | off |
| on (unmask) | アンマスク再押下 | off |
| on (mask) | アンマスク押す | on, op='unmask' |
| on (unmask) | マスク押す | on, op='mask' |

## 5. キャンバス上 pointerdown の優先順位

上から順にチェック、マッチしたら処理して return。

1. **マスクモード ON** → マスク描画、`stopPropagation`、return
2. **picking 状態** → 4点クリックを1点ずつ記録、`stopPropagation`、return
3. **editing + ftVisible=true** の時、ft-point/ハンドル以外をタップ → `ftVisible=false` のみ、return（他のモードに移行しない）
4. **editing + ftVisible=false** の時、キャンバスタップ → **picking に遷移**（既存データ破棄）、そのタップを1点目として続行
5. **custom + customVisible=true** の時、ステッカー以外をタップ → 新規customDragStart、return
6. **ステッカー選択中ハンドル** → resize/rotate/delete
7. **ステッカーヒットテスト** → 選択

### 5.1 枠外タップの挙動

- **4点 editing (visible) で枠外タップ** → `ftVisible=false`（枠のみ消える、データ・変形維持）、**後続処理は実行しない**
- **custom (visible) で枠外タップ** → 新規customDragStart（新しい範囲を作り始める）

### 5.2 相互作用

- **custom + 4点 editing** 両方 active な場合、**表示されている方が優先**
  - customVisible=true → キャンバスタップで新規custom
  - customVisible=false && ftVisible=true → キャンバスタップで `ftVisible=false`
  - どちらも非表示 → キャンバスタップで新規 picking（既存データ破棄）

## 6. イベントハンドラ（canvas要素）

キャプチャフェーズ（`addEventListener(..., true)`）:
- **マスク** → eraserMode時のみ発火、stopPropagation

非キャプチャ（通常）の単一 pointerdown ハンドラで、§5 の順に分岐：

```pseudo
function onCanvasPointerDown(e) {
    if (eraserMode) return;  // キャプチャで処理済
    const p = screenToInternal(e.clientX, e.clientY);

    // 1. picking
    if (ftState === 'picking' && 4点以内) {
        addPickedPoint(p); return;
    }

    // 2. editing + visible の枠外タップ
    if (ftState === 'editing' && ftVisible
        && !e.target.classList.contains('ft-point')) {
        setFtVisible(false);
        return;  // 後続処理はしない
    }

    // 3. editing + hidden でキャンバスタップ → 新規 picking
    if (ftState === 'editing' && !ftVisible
        && !customVisible
        && !e.target.closest('.sticker')) {
        clearFtData();
        enterPicking();
        addPickedPoint(p);
        return;
    }

    // 4. custom + visible
    if (targetArea === 'custom' && customVisible
        && !e.target.closest('.sticker')) {
        startCustomDrag(p); return;
    }

    // 5-6. ステッカー操作
    ...
}
```

## 7. マスクの実装（重要）

### 7.1 レイヤー配置

```html
<div class="canvas-container">
  <img class="sticker-layer" />
  <canvas class="eraser-mask-canvas"></canvas>  <!-- sticker-layer の直上 -->
</div>
```

CSS:
```css
.eraser-mask-canvas {
    position: absolute;
    inset: 0;
    width: 100%;
    height: 100%;
    z-index: 2;
    pointer-events: none;
    transform-origin: 0 0;
    /* transform は JS で sticker-layer と完全に同期 */
}
```

### 7.2 描画

- `eraser-mask-canvas` の内部サイズは `canvasSize.w × canvasSize.h`
- 描画は**キャンバス座標系（canvasSize）**で行う
- `eraserPaths` の各点について:
  - `mode='mask'` → `destination-out` で黒い穴（透明化）
  - `mode='unmask'` → `source-over` で白で塗る（復元）

sticker-layer の画像は**変更しない**。マスクキャンバスが上に重なって「穴が空いたように見える」。

### 7.3 transform 同期

`apply3d()` で sticker-layer に transform を設定するたびに、**同じ transform を eraser-mask-canvas にも設定**。こうすれば画像とマスクが完全に一致して変形される。

### 7.4 座標変換

ユーザーが画面上でドラッグ → `screenToInternal()` で canvasSize 単位の (x, y) を取得 → `eraserPaths` に push。

`eraser-mask-canvas` は canvasSize 単位で描画され、CSS で `width:100%` により sticker-layer と同じ画面サイズに描画される。sticker-layer と同じ transform がかかるので、見た目の位置が一致する。

### 7.5 リセットタイミング

以下のタイミングで `eraserPaths = []`:
- BOMB! 実行時
- 削除BOMB（clearAll）実行時
- 自由変形リセット時（変形と一緒にマスクもリセットすべきか検討。とりあえず連動させない）

## 8. Undo / Redo

スナップショットに含める:
- `stickers`（src は除外、srcId だけ）
- `eraserPaths`
- `ftInitial`, `ftPoints`, `ftMatrix` (オプション、複雑なので一旦 stickers と eraser のみ)

JSON化時は軽量化必須（src は大きすぎる）。

## 9. 既存仕様との整合性チェックポイント

- [ ] モバイル(04メニュー)の rebindPanelEvents で新ボタン(maskToggle, unmaskToggle)も bind する
- [ ] renderCanvas 時にマスクレイヤーも再生成/維持
- [ ] 書き出し(PNG/JPG) 時、マスクも合成する
- [ ] プロジェクト保存/読み込みで eraserPaths も対象にする
- [ ] メニュー閉じた時の枠非表示動作

---

この設計を基に実装を進める。実装時に必ずこのドキュメントを参照し、堂々巡りを防ぐ。

---

## 試行錯誤ログ

デバッグ・設計変更の履歴。試したがダメだった方法・その理由・最終結論を時系列で残す。
次セッションでも参照するため、`/debug-assist` skill がここに追記する。

**書式**：
```
### YYYY-MM-DD — （症状の一言タイトル）

**症状**: 期待 vs 実際

**試した仮説**:
- 仮説1: XX → 偽（理由：...）
- 仮説2: YY → 真
- 仮説3: ZZ → 未検証

**根本原因**: ...

**修正**: ファイル:行番号 / 変更内容の要約

**学び**: 次回似た症状が出たら真っ先に〇〇を疑う
```

（ここから下に時系列で積む。最新が上）

