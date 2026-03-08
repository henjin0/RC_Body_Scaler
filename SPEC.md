# RC Body Scaler — 仕様書 (SPEC.md)

## 概要

| 項目 | 値 |
|------|----|
| ファイル形式 | 単一HTMLファイル（サーバー不要） |
| 入力 | STL（バイナリ・ASCII両対応） |
| 出力 | STL（バイナリ形式） |
| 3Dライブラリ | Three.js r128（CDN） |
| フォント | Share Tech Mono, Barlow Condensed（Google Fonts） |

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

### 04 / Target Dimensions
- ホイールベース（X）・車高（Y）・トレッド幅（Z）を mm で入力
- 空欄 = 倍率 1.0（変更なし）
- スケールバー（赤=X・緑=Y・青=Z）でリアルタイムプレビュー
- 「▶ スケール適用」ボタン

### 05 / Export
- ゴースト表示チェック: スケール適用後に元モデルを opacity 0.18 の半透明で表示/非表示
- 「⬇ STLをダウンロード」でバイナリ STL（`rc_body_scaled.stl`）を保存

### 06 / Interior Box
- チェックで黄色ワイヤーフレーム + 半透明面の直方体を表示
- 配置: 前後輪 X 中点、タイヤ底面 Y、Z=0
- X / Y / Z 寸法を個別入力（リアルタイム更新）
- 体積（cm³）をリアルタイム表示
- デフォルト値: X = ホイールベース × 100%、Y = 車高 × 60%、Z = 車幅 × 70%

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
- `buildMarker('front')` / `buildMarker('rear')` 再描画
- `buildInteriorBox()` 再描画

---

## 主要関数

| 関数 | 役割 |
|------|------|
| `loadSTL(buffer)` | バイナリ/ASCII を判定してパース、シーンに追加 |
| `rotateModel(axis, deg)` | モデルを指定軸で回転、ホイールリセット |
| `startWheelPick(which)` | 指定モード開始、ORTHO + 側面ビューへ遷移 |
| `buildMarker(which)` | 軸位置マーカー（球・Torus・十字線）を再描画 |
| `removeMarker(which)` | マーカーを dispose して削除 |
| `confirmPick()` | 確定処理、Y ロック平均化、PERSP に戻る |
| `applyScale()` | scaledMesh を生成・表示 |
| `buildInteriorBox()` | Interior Box ジオメトリを再生成 |
| `exportSTL()` | scaledMesh をバイナリ STL としてダウンロード |
| `setOrtho(on)` | カメラ切替 |
| `updateOrthoSize()` | 正投影サイズをウィンドウに合わせて再計算 |
| `applyTheme3D(isLight)` | 3D シーンのテーマ色を更新 |

---

## グローバル変数（主要）

```javascript
let scene, renderer, mainMesh, scaledMesh;
let perspCamera, orthoCamera, camera;
let useOrtho = false;
let pickMode  = null;   // 'front' | 'rear' | null
let wheels    = { front: null, rear: null };  // { x, y } mm
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
- スケール適用済み `scaledMesh` の `BufferGeometry` から `position` 属性を読み取り
- 単位: mm（入力値と同一）

---

## 既知の制約

- テクスチャ・マテリアル情報は STL の仕様上保持されない
- 複数ファイルの同時読み込み不可（単一モデルのみ）
- 非常に大きな STL（数百万ポリゴン以上）はブラウザのメモリ上限により動作が遅くなる場合がある
