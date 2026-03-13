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

| 軸 | 方向 | 色 |
|----|------|----|
| X | 進行方向（前後） | 赤 |
| Y | 車高（上下） | 緑 |
| Z | 車幅（左右） | 青 |

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

### 02 / Orientation
- X / Y / Z 軸それぞれ +90° / −90° の回転ボタン（計 6 個）
- 3D シーン内に ArrowHelper（赤=X・緑=Y・青=Z）を常時表示
- 回転後はホイール指定をリセット

### 03 / Wheel Points
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

### 04 / Target Dimensions
- ホイールベース（X）・車高（Y）・トレッド幅（Z）を mm で入力
- 空欄 = 倍率 1.0（変更なし）
- スケールバー（赤=X・緑=Y・青=Z）でリアルタイムプレビュー
- 「▶ スケール適用」ボタン

### 05 / Export
- ゴースト表示チェック: スケール適用後に元モデルを opacity 0.18 の半透明で表示/非表示
- 「⬇ STLをダウンロード」でバイナリ STL（`rc_body_scaled.stl`）を保存
- エクスポート優先順位: `shellGeo` → `scaledGeo` → `currentGeo`

### 06 / Interior Box
- チェックで黄色ワイヤーフレーム + 半透明面の直方体を表示
- 配置: 前後輪 X 中点、タイヤ底面 Y、Z=0
- X / Y / Z 寸法を個別入力（リアルタイム更新）
- 体積（cm³）をリアルタイム表示
- デフォルト値: X = ホイールベース × 100%、Y = 車高 × 60%、Z = 車幅 × 70%

### 07 / Shell Thickness
内側方向に肉厚を持ったシェルを生成する機能。開口部（底面など）は元の形状のまま保持する。

**パラメータ**

| UI要素 | ID | 説明 |
|--------|----|------|
| 手法 | `shellMethod` | `hollow`（クローズドホロー）/ `offset`（法線オフセット）/ `scale`（スケールシェル）/ `solidcap`（ソリッドキャップ） |
| 肉厚 (mm) | `shellThk` | 内側に押し込む厚み（正の値） |
| 面フィルタ | `shellFilter` | `none` / `outward` / `raycast` |
| スムージング回数 | `shellSmooth` | 0 = 無効、1〜200 = ラプラシアン反復回数 |
| 部位カラー表示 | `shellColorize` | チェック時、外面=シアン / 内面=グリーン / 底面リング=オレンジ で頂点カラーを付与 |
| タイヤ穴くり抜き | `shellCutTire` | チェック時、前輪・後輪それぞれ Z 方向全貫通シリンダー 2 本でシェルから除去（左右対称のため 2 本で 4 タイヤをまとめて処理） |
| クリアランス | `tireClearance` | タイヤ半径に加算する余裕代（デフォルト 2mm） |

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

**手法: ソリッドキャップ（`solidcap`）**
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
weldGeo()          → 頂点溶接・面インデックス化
laplacianSmooth()  → （smoothIter > 0 の場合）抽象化頂点生成
面フィルタ          → （filterMode に応じて）対象面を絞り込み
buildShellGeo()    → 外面 + 内面（逆巻き） + 境界壁ストリップ
 or
buildShellGeoScale()
```

**境界エッジ検出**
- 有向エッジ `[a,b]` に対して逆向き `[b,a]` が存在しない場合を境界エッジとする
- 境界エッジ1本につき2三角形の壁ストリップを生成

**シェルメッシュ**
- `scaledMesh` と同じテーマ色・マテリアルを使用（`MeshPhongMaterial`、`opacity: 0.92`、`DoubleSide`）
- 表示中は `mainMesh` / `scaledMesh` を非表示にする
- `applyTheme3D` でテーマ変更時に色を追従

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
    scaled: { color: 0x0a2a18, specular: 0x39d353, shininess: 80 },
    ghost:  { color: 0x1a2a3a, specular: 0x00e5ff, shininess: 30 },
    front:  0x00e5ff,  rear: 0xff6b35,  box: 0xf6c90e,
    light1: { color: 0x00e5ff, intensity: 0.8 },
    light2: { color: 0xff6b35, intensity: 0.4 },
  },
  light: {
    model:  { color: 0x8ab0cc, specular: 0x336688, shininess: 50 },
    scaled: { color: 0x5a9e72, specular: 0x2d7a44, shininess: 70 },
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
| `rotateModel(axis, deg)` | モデルを指定軸で回転、ホイールリセット、shellMesh/shellGeo もクリア |
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
| `cutTireFaces(geo, tireDefs)` | Z方向全貫通シリンダーのXY距離判定で面を除去してタイヤ穴を作成。`tireDefs`: `[{wx,wy,r2}]` |
| `buildShellGeoHollow(uniqueVerts, faces, thickness, normVerts, withColors)` | クローズドホロー方式。閉じたソリッドを中空化。壁ストリップなし。`withColors=true` で部位別頂点カラーを付与。`g._boundaryCount` で境界エッジ数を返す |
| `buildShellGeoSolidCap(uniqueVerts, faces, thickness, normVerts, withColors)` | ソリッドキャップ方式。境界YクランプによりスライサーのB層誤検知を解消。`withColors=true` で部位別頂点カラーを付与 |
| `applyShell()` | シェル生成パイプライン全体を async で実行（進捗表示付き） |
| `resetShell()` | shellMesh/shellGeo を破棄し元メッシュを再表示 |

---

## グローバル変数（主要）

```javascript
let scene, renderer, mainMesh, scaledMesh, shellMesh;
let perspCamera, orthoCamera, camera;
let useOrtho = false;
let currentGeo = null;   // 読み込み済み元ジオメトリ
let scaledGeo  = null;   // スケール適用済みジオメトリ
let shellGeo   = null;   // シェル生成済みジオメトリ
let pickMode  = null;    // 'front' | 'rear' | null  ※確定時に必ず null にリセット
let activeAdj = null;    // 'front' | 'rear' | null
const wheels  = { front: null, rear: null };  // THREE.Vector3 (mm)
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

---

## 既知の制約

- テクスチャ・マテリアル情報は STL の仕様上保持されない
- 複数ファイルの同時読み込み不可（単一モデルのみ）
- 非常に大きな STL（数百万ポリゴン以上）はブラウザのメモリ上限により動作が遅くなる場合がある
- スケールシェルの肉厚は場所によって物理的厚みが異なる（重心からの距離に依存）
