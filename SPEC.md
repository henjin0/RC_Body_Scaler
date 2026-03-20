# RC Body Scaler — 仕様書 (SPEC.md)

## 概要

| 項目 | 値 |
|------|----|
| ファイル形式 | 単一HTMLファイル（サーバー不要） |
| 入力 | STL（バイナリ・ASCII両対応） |
| 出力 | STL（バイナリ形式） |
| 3Dライブラリ | Three.js r128（CDN） |
| フォント | Share Tech Mono, Barlow Condensed（Google Fonts） |
| 制作者 | henjin01_Fab |
| GitHub | https://github.com/henjin0/RC_Body_Scaler |
| Facebook | https://www.facebook.com/minoru.inoue.90 |

---

## 軸定義

| 軸 | 方向 | 諸元名称 | 色 |
|----|------|----------|----|
| X | 進行方向（前後） | 全長・ホイールベース | 赤 |
| Y | 上下 | 全高 | 緑 |
| Z | 左右 | 全幅・トレッド | 青 |

Three.js の Y軸上向き座標系に準拠。モデル読み込み後は XZ 中央・Y 底面=0 にセンタリング。

---

## カメラ

```
perspCamera  : PerspectiveCamera（通常視点）
orthoCamera  : OrthographicCamera（正投影）
camera       : アクティブなカメラ（上記どちらか）
useOrtho     : boolean
```

- `setOrtho(on)` で切り替え。ウィンドウリサイズ時に `updateOrthoSize()` を呼んで正投影サイズを再計算。
- ホイール指定モード開始時に自動で ORTHO + 側面ビューへ遷移（easeInOut アニメーション）。確定後は PERSP に戻る。
- ヘッダーの `PERSP / ORTHO` ボタンで手動切替可能。
- ヘッダーの `ABOUT` ボタンでモーダルを表示（制作者・GitHub・Facebook リンク）。

---

## UI パネル構成（右サイドバー）

### 01 / Load STL
- ドラッグ&ドロップ または クリックでファイル選択
- 読み込み後に XYZ 寸法（mm）・三角形数を表示
  - `X (全長)` / `Y (全高)` / `Z (全幅)` と表記（自動車諸元の正式名称に準拠）

### 02 / Orientation
- X / Y / Z 軸それぞれ +90° / −90° の回転ボタン（計 6 個）
- 3D シーン内に ArrowHelper（赤=X・緑=Y・青=Z）を常時表示
- 回転後はホイール指定をリセット
- **STL 読み込み時に Meshy（生成AI）の座標系を自動補正**: X+90°×3 + Y+90°×2 を初期 `rotMatrix` として適用済みの状態でモデルを表示する（ズレがある場合は手動ボタンで追加調整可能）

### 03 / Mirror Symmetry
Z 軸（車幅方向）でモデルを左右対称化する機能。向き調整後・ホイール指定前に実行する。

**ボタン**

| ボタン | ID | 動作 |
|--------|----|------|
| Z+ 側でミラー | `mirrorZpBtn` | Z+ 半分を保持し Z− 側にミラーコピー |
| Z− 側でミラー | `mirrorZmBtn` | Z− 半分を保持し Z+ 側にミラーコピー |
| ↩ ミラー前に戻す | `mirrorUndoBtn` | ミラー操作を取り消して元形状に復元（ミラー後のみ表示） |

**アルゴリズム（平面クリッピング）**

重心判定ではなく各三角形を `Z = centerZ`（bbox 中心）で正確にクリップする。

| 保持頂点数 | 処理 |
|-----------|------|
| 3 | そのままコピー |
| 1 | 2辺との交点を補間 → 三角形 1 枚 |
| 2 | 2辺との交点を補間 → 三角形 2 枚（四角形分割） |

- 交点は `Z = centerZ` に正確に配置 → 両半分の境界頂点が完全一致
- `applyShell()` 内の `weldGeo()` が境界をシームレスに接合
- ミラーコピーは巻き方向を逆（v0,v2,v1）・法線 Z を符号反転して生成

**アンドゥ（`preMirrorState`）**

- ミラー実行前に `{ originalGeo, rotMatrix }` を `preMirrorState` に保存
- 「ミラー前に戻す」ボタンで復元、ホイール・スケール・シェルも同時リセット
- 新規ファイル読み込み時に `preMirrorState` をクリア・ボタンを非表示

**副作用**

- ミラー後は `originalGeo` と `rotMatrix` をミラー済み形状で上書き → 回転ボタンが引き続き機能する
- ホイール位置・`scaledGeo`・`shellGeo` はすべてリセットされる

### 04 / Wheel Points
- 「前輪軸をクリック指定」「後輪軸をクリック指定」ボタン
- 指定モード中は ORTHO + 側面ビューに自動切替
- クリックで軸位置を指定（レイキャスト → XZ 平面フォールバック）
- 矢印キーで微調整、`Enter` または「✓ 軸位置を確定」ボタンで確定

**キー操作（指定モード中のみ有効）**

| キー | 移動量 |
|------|--------|
| `← →` | X 方向 1mm |
| `↑ ↓` | Y 方向 1mm |
| `Shift` + 矢印 | 0.1mm |
| `Tab` | 前輪 ↔ 後輪切替 |
| `Enter` | 確定・PERSP 視点に戻る |
| `Esc` | キャンセル |

> `input` / `textarea` にフォーカス中は矢印キーのハンドラを無効化（`Esc` のみ通過）

**タイヤ半径入力（可視化用）**
- 前輪: シアン/青の球 + 十字線 + Torus
- 後輪: オレンジ/橙の球 + 十字線 + Torus

**Y 軸ロック**
- チェック時、片方の Y を動かすともう一方が追従
- 矢印キー ↑↓ 時: 相手方の Y をリアルタイム同期して `buildMarker` を再描画
- クリック新規指定時: 相手方が確定済みならそのYに合わせる
- 確定ボタン押下時: 両輪の Y を平均値に揃えてから確定
- 確定後は `pickMode` と `activeAdj` を両方 null にリセット（確定後のビューポートクリックで車輪位置が上書きされるバグを防ぐ）

### 05 / Target Dimensions
- ホイールベース（X）・全高（Y）・トレッド（Z）を mm で入力
  - 各ラベルは自動車諸元の正式名称に準拠（全高＝路面〜ルーフ高さ、全幅＝ボディ幅、トレッド＝左右輪接地面中心間距離）
- 空欄 = 倍率 1.0（変更なし）
- スケールバー（赤=X・緑=Y・青=Z）でリアルタイムプレビュー
  - バー表記: `X (WB)` / `Y (全高)` / `Z (トレッド)`
- 「▶ スケール適用」ボタン

**プリセット**

入力欄の上にプリセット選択 UI を配置。プラットフォームを選ぶとモデル一覧が展開され、選択するとホイールベース・全高・トレッドを自動入力してスケールバーを即時更新する。

> ⚠ プリセット値はあくまでも目安。実際のシャーシ寸法は必ず実測すること。

| ID | 要素 |
|----|------|
| `presetPlatform` | プラットフォーム選択 `<select>` |
| `presetModel` | モデル選択 `<select>`（プラットフォーム未選択時は disabled） |

収録プリセット（`PRESETS` 定数）:

| プラットフォーム | モデル |
|---|---|
| ミニ四駆 | FM-A / MA / MS / Super-1・VS・AR |
| ミニッツ（Kyosho） | MR-03 N/M/W・MR-04 M・AWD MA-020 M |
| タミヤ ラジコン | TT-02/TT-01・TA07・M-07/M-06・DT-03 |
| WLtoys | K989/K969(1/28)・284131(1/28)・144001(1/14)・12428(1/12) |
| Turbo Racing | C70/C71(1/76)・C72(1/76)・TB01(1/76 F1) |

**ホイールアーチ局所変形（スケール適用後に表示、オプション機能）**

スケール適用後に `archDeformBlock` が展開される。チェックボックス `archDeformEnabled` を ON にすると操作UIが表示され、同時にグリーン・黄色の可視化円も描画される。OFF の場合は入力欄・可視化ともに非表示。

▶ アーチ変形 ボタンは累積適用される。クリックのたびに各頂点が目標半径へ段階的に近づく（1クリックで完全に到達するわけではない）。

| UI 要素 | ID | 説明 |
|--------|----|------|
| ホイールアーチ局所変形（オプション） | `archDeformEnabled` | チェック時のみ操作UIと可視化円（グリーン・黄色）を表示 |
| 影響半径 | `archInfluence` | ホイール中心から何 mm 以内の頂点を変形するか（デフォルト 20mm）。**グリーン円**で可視化 |
| 目標アーチ半径 | `archTargetRadius` | 変形後にアーチがホイール中心から何 mm の位置になるかの目標値（デフォルト 15mm）。**黄色円**で可視化 |
| 縮小方向にも変形する | `archShrinkEnabled` | OFF（デフォルト）: push-only（拡大のみ）。ON: pull-only（縮小のみ、溝部分を引き込む） |
| ▶ アーチ変形 | `archApplyBtn` | 変形を累積適用。クリックごとに目標半径へ段階的に近づく |
| ↩ 戻す | `archUndoBtn` | 最初の適用前の `scaledGeo` に戻す（1段階 undo） |
| ポリゴン細分化 | `subdivEnabled` | チェック時のみ細分化オプションを表示（デフォルト：非表示、オプション機能） |
| 最大辺長 | `subdivMaxEdge` | この長さ (mm) を超える辺を持つ三角形を 4 分割（デフォルト 3mm）。`subdivBtn` で実行 |

**アルゴリズム**

各頂点について、前輪・後輪それぞれのホイール中心からの XY 平面距離 `d` を計算する。
より大きな weight を持つホイールを選択し、半径方向（Z 不変）に移動する。

```
weight  = max(0, (1 - d / influence))²                       ← 二乗 falloff（境界が滑らか）

[shrinkEnabled OFF — 拡大モード]
delta   = max(0, targetRadius - d)   ← push-only: d < targetRadius の頂点のみ外へ押し出し

[shrinkEnabled ON — 縮小モード]
delta   = min(0, targetRadius - d)   ← pull-only: d > targetRadius の頂点のみ内へ引き込み

new_d   = d + delta × weight
vertex.x = wx + (dx/d) × new_d
vertex.y = wy + (dy/d) × new_d
vertex.z 変化なし
```

- **拡大モード（デフォルト）**: `d < targetRadius` の頂点のみ外側へ押し出し。外側は不変
- **縮小モード（オプション）**: `d > targetRadius` の頂点のみ内側へ引き込み（溝・アーチ外縁を縮小）。内側は不変
- `d = targetRadius` 付近は weight × delta が 0 に近づくため滑らかに変化
- `d ≥ influence` の頂点は変化なし（影響範囲外）
- スケール適用済みの場合、ホイール座標を `wheels.x × sx`、`wheels.y × sy` で変換してから使用
- 変形結果は `scaledGeo` を差し替えて保存（`shellGeo` は無効化）
- `preArchDeformGeo` に変形前の `scaledGeo` をバックアップ（最初の適用時のみ保存）

**可視化（`buildArchDeformViz`）**

スケール適用後に `archDeformBlock` が表示されている間、各ホイール中心に２種類の円筒を描画する。

| 色 | 意味 | 半径 |
|----|------|------|
| グリーン | 影響範囲（ここまでの頂点が変形対象） | `archInfluence` |
| 黄色 | 目標アーチ半径（変形後の到達目標） | `archTargetRadius` |

- 入力値変更でリアルタイム更新
- `removeArchDeformViz()` でシーンからクリア（`rebuildMesh` 時など）

### 06 / Export
- ゴースト表示チェック: スケール適用後に元モデルを opacity 0.18 の半透明で表示/非表示
- 「⬇ STLをダウンロード」でバイナリ STL（`rc_body_scaled.stl`）を保存
- エクスポート優先順位: `shellGeo` → `scaledGeo` → `currentGeo`

### 07 / Interior Box
- チェックで黄色ワイヤーフレーム + 半透明面の直方体を表示
- 配置: 前後輪 X 中点、タイヤ底面 Y、Z=0
- X / Y / Z 寸法を個別入力（リアルタイム更新）
- 体積（cm³）をリアルタイム表示
- デフォルト値: X = ホイールベース × 100%、Y = 車高 × 60%、Z = 車幅 × 70%

### 08 / Shell Thickness
内側方向に肉厚を持ったシェルを生成する機能。開口部（底面など）は元の形状のまま保持する。

**パラメータ**

| UI要素 | ID | 説明 |
|--------|----|------|
| 手法 | `shellMethod` | `solidcap`（ソリッドキャップ・**デフォルト/おすすめ**）/ `hollow`（クローズドホロー）/ `offset`（法線オフセット）/ `scale`（スケールシェル）/ `sdf`（SDF補正）/ `planb`（シュリンクラップ） |
| 肉厚 (mm) | `shellThk` | 内側に押し込む厚み（正の値） |
| 面フィルタ | `shellFilter` | `none` / `outward` / `raycast` |
| スムージング回数 | `shellSmooth` | 0 = 無効、1〜200 = ラプラシアン反復回数 |
| 部位カラー表示 | `shellColorize` | チェック時、外面=シアン / 内面=グリーン / 底面リング=オレンジ で頂点カラーを付与 |
| タイヤ穴くり抜き | `shellCutTire` | チェック時、前輪・後輪それぞれ Z 方向全貫通でシェルから除去（左右対称のため 2 本で 4 タイヤをまとめて処理） |
| 穴の形状 | `cutTireShape` | `circle`（円形）/ `kamaboko`（かまぼこ型 D形） |
| タイヤ半径（円形） | `cutTireR` | 円形モード時のカット半径（スケール後実寸）、デフォルト 30mm |
| 横半径 RX（かまぼこ） | `cutTireRx` | ホイール中心からの水平距離、デフォルト 30mm |
| 縦半径 RY（かまぼこ） | `cutTireRy` | 矩形部の高さ ＝ 半楕円の垂直半軸、デフォルト 30mm |
| クリアランス | `tireClearance` | タイヤ半径に加算する余裕代（デフォルト 2mm） |
| 肉厚を加算 | `cutAddThk` | チェック時、カット半径にシェル肉厚を加算する（デフォルト OFF）。OFF 時: カット半径 = 指定値 + クリアランス。ON 時: カット半径 = 指定値 + クリアランス + 肉厚 |
| カット範囲制限 | `shellClipRange` | チェック時、6方向それぞれの最大到達距離を % で制限（100=制限なし） |

**タイヤ穴くり抜きの座標系**

スケール適用済み（`scaledGeo` が存在する）場合は **スケール後のジオメトリに直接カット**する。

- ホイール座標をスケール後実寸に変換: `wx = wheels.x × sx`、`wy = wheels.y × sy`
- カット半径はそのまま実寸で使用（割り戻し不要）
- これにより XZ スケール比が 1:1 でない場合でも穴が真円/真D形を保つ
- スケール未適用の場合は `currentGeo` に対してそのまま適用

**穴の形状: 円形（`circle`）**

- 判定: `(cx-wx)² + (cy-wy)² < (cutTireR + clearance [+ thk])²`（Z 方向は全貫通、`cutAddThk` チェック時のみ `+thk`）
- 可視化: `THREE.CylinderGeometry`（半透明オレンジ円筒）

**穴の形状: かまぼこ型（`kamaboko`）**

断面 = 矩形（下部） + 半楕円（上部）。ホイール中心 Y（`wy`）が矩形と半楕円の接線になる。

```
        (0, wy+ry)          ← 楕円頂点
       /           \
(-rx, wy) -------- (rx, wy) ← 接線（ホイール中心 Y）
      |               |
(-rx, wy-ry) ---- (rx, wy-ry) ← 底辺
```

| 部位 | Y 範囲 | 判定 |
|------|--------|------|
| 矩形 | `wy-ry ≤ cy < wy` | `\|cx-wx\| < rx` |
| 半楕円 | `cy ≥ wy` | `(cx-wx)²/rx² + (cy-wy)²/ry² < 1` |

- `rx`, `ry` の実効値: `cutTireRx/Ry + clearance [+ thk]`（`cutAddThk` チェック時のみ `+thk`）
- 可視化: `THREE.Shape`（矩形+半楕円断面）+ `ExtrudeGeometry` でZ方向押し出し

**カット範囲制限（`shellClipRange`）**

| ID | 方向 | 動作 |
|----|------|------|
| `clipXp` | X+ | `bbox.max.x × (値/100)` を超える三角形を除去 |
| `clipXm` | X− | `bbox.min.x × (値/100)` を下回る三角形を除去 |
| `clipYp` | Y+ | `bbox.max.y × (値/100)` を超える三角形を除去 |
| `clipYm` | Y− | `bbox.min.y × (値/100)` を下回る三角形を除去 |
| `clipZp` | Z+ | `bbox.max.z × (値/100)` を超える三角形を除去 |
| `clipZm` | Z− | `bbox.min.z × (値/100)` を下回る三角形を除去 |

- 判定は三角形の重心座標で行う（頂点単位ではない）
- シェル生成完了後に `clipBodyByRange()` で適用（タイヤ穴くり抜きと独立して動作）
- 頂点カラー属性がある場合はそのまま引き継ぐ

**手法: クローズドホロー（`hollow`）** ← AI生成モデル推奨
- **対象**: 閉じたウォータータイトなソリッドメッシュ（境界エッジがゼロのもの）
- 外面 = 元の面をそのまま使用
- 内面 = 法線逆方向にオフセットした面（逆巻き）
- **壁ストリップを一切生成しない** → スライサーが壁2枚だけ認識するクリーンな出力
- 境界エッジが検出された場合はステータスバーに件数を警告表示（処理は継続）
- `hollow` 選択時は部位カラー表示を自動 ON
- `g._boundaryCount` で境界エッジ数を返す

**手法: 法線オフセット（`offset`）**
- 各頂点の面積加重法線を計算し、法線の逆方向に `thickness` mm オフセットして内面頂点を生成
- スムージング > 0 の場合、ラプラシアンスムージング済み頂点の法線を使ってオフセット方向を算出（外面位置は元のまま保持）

**手法: ソリッドキャップ（`solidcap`）← デフォルト・おすすめ**
- 境界頂点の内側 Y 座標を外側境界と同一にクランプし、底面リングを水平化する
- 現行手法の「斜め壁ストリップ」をスライサーが複数の壁として誤認する問題（壁4枚問題）を解消
- 内側 XZ は法線オフセットで計算、Y のみ外側にそろえる
- 部位カラー表示（`shellColorize` チェック）を有効にすると 外面=シアン / 内面=グリーン / 底面リング=オレンジ で色分け表示
- `solidcap` 選択時は部位カラー表示を自動 ON

**手法: スケールシェル（`scale`）**
- 全頂点の重心を算出
- 重心からの平均距離 `meanDist` を計算し `scale = 1 - thickness / meanDist` を導出
- 内面頂点 = `centroid + (基準頂点 - centroid) × scale`
- スムージング > 0 の場合、ラプラシアン済み頂点を内面形状のベースに使用（外面は元のまま）
- トポロジーが外面と完全一致するため境界ステッチが安定し、法線計算由来の毛羽立ちが発生しない

**面フィルタ**

| 値 | 動作 |
|----|------|
| `none` | 全面を処理 |
| `outward` | ドット積で外向き面のみを対象（高速） |
| `raycast` | 重心へのレイキャストで遮蔽なしの面のみを対象（低速・高精度） |

レイキャストは 200 面チャンクごとに `await setTimeout(0)` で UI に制御を返し、進捗を `stepHint` に表示する。

**シェル構築フロー（共通）**

```
タイヤ穴くり抜き（scaledGeo に直接 or currentGeo）
    ↓
weldGeo()          → 頂点溶接・面インデックス化
laplacianSmooth()  → （smoothIter > 0 の場合）抽象化頂点生成
面フィルタ          → （filterMode に応じて）対象面を絞り込み
buildShellGeo*()   → 手法に応じてシェル生成
clipBodyByRange()  → （shellClipRange チェック時）範囲外三角形を除去
```

**境界エッジ検出**
- 有向エッジ `[a,b]` に対して逆向き `[b,a]` が存在しない場合を境界エッジとする
- 境界エッジ1本につき2三角形の壁ストリップを生成

**シェルメッシュ**
- `scaledMesh` と同じテーマ色・マテリアルを使用（`MeshPhongMaterial`、`opacity: 0.92`、`DoubleSide`）
- 表示中は `mainMesh` / `scaledMesh` を非表示にする
- `applyTheme3D` でテーマ変更時に色を追従

**後スムージング（`applyShellPostSmooth`）**
- 生成済み `shellGeo` を `weldGeo` → `laplacianSmooth` → バッファ再構築の順で平滑化
- 新ジオメトリは頂点カラー属性を持たないため、スムージング完了後にマテリアルの `vertexColors` を `false` に戻し、テーマ色（`THEME.scaled`）を再適用する
  - これを行わないと `vertexColors: true` のまま色データが欠落し、モデルが真っ黒になる

---

## テーマ

### CSS 変数

| 変数 | ダーク | ライト |
|------|--------|--------|
| `--bg` | `#0a0c0f` | `#f0ede8` |
| `--panel` | `#111418` | `#e4e0d8` |
| `--border` | `#1e2530` | `#c8c2b8` |
| `--accent` | `#00e5ff` | `#0077aa` |
| `--accent2` | `#ff6b35` | `#cc5500` |
| `--text` | `#c8d4e0` | `#2a2520` |
| `--dim` | `#4a5568` | `#7a7268` |
| `--scene-bg` | `#070a0d` | `#dedad4` |

切替時に `transition: background-color .25s, border-color .25s, color .25s` を全要素に適用。

### 3D テーマ色（`THEME` 定数）

```javascript
const THEME = {
  dark: {
    model:  { color: 0x1a2a3a, specular: 0x00e5ff, shininess: 60 },
    scaled: { color: 0xc8ffe8, specular: 0xffffff, shininess: 80 },  // ミントホワイト（シェル・スケール済みメッシュ）
    ghost:  { color: 0x1a2a3a, specular: 0x00e5ff, shininess: 30 },
    front:  0x00e5ff,  rear: 0xff6b35,  box: 0xf6c90e,
    light1: { color: 0x00e5ff, intensity: 0.8 },
    light2: { color: 0xff6b35, intensity: 0.4 },
  },
  light: {
    model:  { color: 0x8ab0cc, specular: 0x336688, shininess: 50 },
    scaled: { color: 0x2e7d5a, specular: 0x1a5c3a, shininess: 75 },
    ghost:  { color: 0x8ab0cc, specular: 0x336688, shininess: 20 },
    front:  0x0077aa,  rear: 0xcc5500,  box: 0x996600,
    light1: { color: 0x6699bb, intensity: 0.6 },
    light2: { color: 0xbb7733, intensity: 0.3 },
  }
};
```

### `applyTheme3D(isLight)` が更新する対象
- `window._light1` / `window._light2` の色・強度
- mainMesh / scaledMesh / ghost のマテリアル
- `shellMesh` のマテリアル（`scaled` テーマ色を使用）
- `buildMarker('front')` / `buildMarker('rear')` 再描画
- `buildInteriorBox()` 再描画

---

## 主要関数

| 関数 | 役割 |
|------|------|
| `loadSTL(buffer)` | バイナリ/ASCII を判定してパース、シーンに追加 |
| `rebuildMesh(geo)` | `currentGeo` を更新・センタリング・mainMesh 再生成・カメラフィット |
| `rotateModel(axis, deg)` | モデルを指定軸で回転、ホイールリセット、shellMesh/shellGeo もクリア |
| `mirrorHalf(keepSide)` | Z 軸平面クリッピングによりミラー対称化。`keepSide`: `'zp'`\|`'zm'`。ミラー前状態を `preMirrorState` に保存 |
| `undoMirror()` | `preMirrorState` から元形状を復元 |
| `startPick(which)` | 指定モード開始、ORTHO + 側面ビューへ遷移 |
| `buildMarker(which)` | 軸位置マーカー（球・Torus・十字線）を再描画 |
| `removeMarker(which)` | マーカーを dispose して削除 |
| `confirmWheelAdj()` | 確定処理、Y ロック平均化、`pickMode`/`activeAdj` リセット、PERSP に戻る |
| `applyLockY()` | 両輪の Y を平均値に揃える |
| `applyScale()` | scaledMesh を生成・表示 |
| `buildInteriorBox()` | Interior Box ジオメトリを再生成 |
| `exportSTL()` | `shellGeo \|\| scaledGeo \|\| currentGeo` をバイナリ STL としてダウンロード |
| `setOrtho(on)` | カメラ切替 |
| `updateOrthoSize()` | 正投影サイズをウィンドウに合わせて再計算 |
| `applyTheme3D(isLight)` | 3D シーンのテーマ色を更新 |
| `yld(msg)` | メッセージ表示 + `setTimeout(0)` で UI に制御を返す async ヘルパー |
| `weldGeo(srcGeo)` | BufferGeometry から重複頂点を統合し `{ uniqueVerts, faces }` を返す |
| `laplacianSmooth(uniqueVerts, faces, iterations)` | 隣接平均によるラプラシアンスムージング、抽象化頂点配列を返す |
| `buildShellGeo(uniqueVerts, faces, thickness, normVerts)` | 法線オフセット方式でシェルジオメトリを生成 |
| `buildShellGeoScale(uniqueVerts, faces, thickness, normVerts)` | スケールシェル方式でシェルジオメトリを生成 |
| `cutTireFaces(geo, tireDefs)` | Z方向全貫通でタイヤ穴を作成。円形: `tireDefs[i] = {wx,wy,r2}`、かまぼこ型: `{wx,wy,rx,ry,rx2,ry2}` |
| `clipBodyByRange(geo, ranges)` | 各軸6方向のクリップ平面で三角形を除去。`ranges`: `{xp,xm,yp,ym,zp,zm}`（各値 0〜100%）。バウンディングボックス最大値×割合をクリップ平面として使用 |
| `buildShellGeoHollow(uniqueVerts, faces, thickness, normVerts, withColors)` | クローズドホロー方式。閉じたソリッドを中空化。壁ストリップなし。`withColors=true` で部位別頂点カラーを付与。`g._boundaryCount` で境界エッジ数を返す |
| `buildShellGeoSolidCap(uniqueVerts, faces, thickness, normVerts, withColors)` | ソリッドキャップ方式。境界YクランプによりスライサーのB層誤検知を解消。`withColors=true` で部位別頂点カラーを付与 |
| `applyShell()` | シェル生成パイプライン全体を async で実行（進捗表示付き） |
| `applyShellPostSmooth()` | 生成済みシェルを後スムージング。完了後 vertexColors をリセットしてテーマ色を再適用 |
| `resetShell()` | shellMesh/shellGeo を破棄し元メッシュを再表示 |
| `applyArchDeform()` | ホイールアーチ周辺を局所的に XY 平面上で放射状押し出し変形（push-only）。`scaledGeo` を更新 |
| `resetArchDeform()` | `preArchDeformGeo` から変形前の `scaledGeo` を復元（1段階 undo） |
| `buildArchDeformViz()` | 影響範囲（グリーン円筒）と目標アーチ半径（黄色円筒）をシーンに描画 |
| `removeArchDeformViz()` | アーチ変形可視化メッシュをシーンから削除・dispose |
| `subdivideGeoByEdge(geo, maxEdge, maxIter)` | maxEdge を超える辺を持つ三角形を 4 分割。非インデックス BufferGeometry を返す |
| `applySubdivide()` | `subdivBtn` から呼び出し。`scaledGeo` を細分化して `scaledMesh` を再構築 |

---

## グローバル変数（主要）

```javascript
let scene, renderer, mainMesh, scaledMesh, shellMesh;
let perspCamera, orthoCamera, camera;
let useOrtho = false;
let originalGeo    = null;   // 読み込み直後の元ジオメトリ（回転・ミラーの基点）
let currentGeo     = null;   // 作業中ジオメトリ（センタリング済み）
let scaledGeo      = null;   // スケール適用済みジオメトリ
let shellGeo       = null;   // シェル生成済みジオメトリ
let rotMatrix      = new THREE.Matrix4();  // 累積回転行列
let preMirrorState   = null;   // ミラー前の { originalGeo, rotMatrix }（アンドゥ用）
let preArchDeformGeo = null;   // アーチ局所変形前の scaledGeo バックアップ（アンドゥ用）
let appliedScales  = null;   // applyScale() 実行時の { sx, sy, sz }
let pickMode  = null;        // 'front' | 'rear' | null  ※確定時に必ず null にリセット
let activeAdj = null;        // 'front' | 'rear' | null
const wheels  = { front: null, rear: null };  // THREE.Vector3 (mm, プレスケール座標)
const markers = { front: null, rear: null };  // { dot, torus, cross }
// window._light1, window._light2: DirectionalLight
```

---

## STL パーサー

- 先頭 80 バイトのヘッダー確認 → `solid` で始まる場合は ASCII、それ以外はバイナリと判定
- バイナリ: `DataView` で三角形数・法線・頂点を読み取り
- ASCII: 正規表現で `vertex` 行をパース

---

## STL エクスポート

- バイナリ形式固定（80 バイトヘッダー + 4 バイト三角形数 + 三角形ごとに 50 バイト）
- エクスポート優先順位: `shellGeo` → `scaledGeo` → `currentGeo`
- 単位: mm（入力値と同一）

---

## 頂点溶接（weldGeo）

STL は三角形ごとに独立した頂点を持つため、シェル生成前に重複頂点を統合する。

```
キー: "${Math.round(x*1000)},${Math.round(y*1000)},${Math.round(z*1000)}"
精度: 0.001mm グリッドで丸め
```

戻り値: `{ uniqueVerts: THREE.Vector3[], faces: [a,b,c][] }`

ミラー対称化後は境界頂点が Z=centerZ に正確に配置されるため、`weldGeo` によりシームが自動接合される。

---

## 既知の制約

- テクスチャ・マテリアル情報は STL の仕様上保持されない
- 複数ファイルの同時読み込み不可（単一モデルのみ）
- 非常に大きな STL（数百万ポリゴン以上）はブラウザのメモリ上限により動作が遅くなる場合がある
- スケールシェルの肉厚は場所によって物理的厚みが異なる（重心からの距離に依存）
- ミラー対称化のクリッピング後の三角形は巻き方向が不均一になる場合があるが、DoubleSide マテリアルおよび `weldGeo` 後の法線再計算で実用上問題ない
