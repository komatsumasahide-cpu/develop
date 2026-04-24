# テトリス 内部設計書

## 1. システム概要

| 項目 | 内容 |
|------|------|
| ファイル | `index.html` (単一ファイル構成) |
| 動作環境 | モダンブラウザ（Web Audio API 対応） |
| 言語 | HTML / CSS / JavaScript (Vanilla) |
| 描画 | Canvas 2D API |
| 音声 | Web Audio API（外部ファイル不使用） |

---

## 2. 画面構成

```
┌──────────────────────────────────────┐
│  #app (flex レイアウト)               │
│  ┌────────────────┐  ┌────────────┐  │
│  │  #board        │  │  #side     │  │
│  │  Canvas        │  │  SCORE     │  │
│  │  300 x 600 px  │  │  LEVEL     │  │
│  │                │  │  LINES     │  │
│  │  #message      │  │  NEXT      │  │
│  │  (GAME OVER)   │  │  ボタン    │  │
│  └────────────────┘  │  操作説明  │  │
│                       └────────────┘  │
└──────────────────────────────────────┘
```

---

## 3. 定数定義

| 定数 | 値 | 説明 |
|------|----|------|
| `COLS` | 10 | フィールド横幅（列数） |
| `ROWS` | 20 | フィールド縦幅（行数） |
| `BLOCK` | 30 | 1ブロックのピクセルサイズ |
| `COLORS` | 8色配列 | インデックス 1〜7 がピース色、0 は空 |
| `PIECES` | 2次元配列×7 | 各ピースの形状定義（数値がピースの色インデックス） |

### ピース定義

| インデックス | 種類 | 形状 |
|-------------|------|------|
| 1 | I | `[[1,1,1,1]]` |
| 2 | J | `[[2,0],[2,0],[2,2]]` |
| 3 | L | `[[0,3],[0,3],[3,3]]` |
| 4 | O | `[[4,4],[4,4]]` |
| 5 | S | `[[0,5,5],[5,5,0]]` |
| 6 | Z | `[[6,6,0],[0,6,6]]` |
| 7 | T | `[[0,7,0],[7,7,7]]` |

---

## 4. 状態変数

| 変数 | 型 | 説明 |
|------|-----|------|
| `grid` | `number[][]` | フィールド全体の状態（20×10）。0=空、1〜7=ブロック色 |
| `piece` | `number[][]` | 現在落下中のピース形状 |
| `pieceX` | `number` | ピースの X 座標（列） |
| `pieceY` | `number` | ピースの Y 座標（行）。負値はフィールド上部外 |
| `nextPiece` | `number[][]` | 次に出現するピース形状 |
| `score` | `number` | 現在のスコア |
| `level` | `number` | 現在のレベル（1〜） |
| `lines` | `number` | 累計消去ライン数 |
| `gameLoop` | `number` | `setInterval` のタイマーID |
| `running` | `boolean` | ゲームプレイ中フラグ |
| `paused` | `boolean` | ポーズ中フラグ |

---

## 5. 関数一覧

### 5.1 ゲームロジック

#### `newGrid() → number[][]`
20×10 の二次元配列をすべて 0 で初期化して返す。

#### `randomPiece() → number[][]`
`PIECES[1〜7]` からランダムに1つ返す。

#### `rotate(p) → number[][]`
ピース行列を時計回り 90° 回転した新しい行列を返す。  
変換式: `result[c][rows-1-r] = p[r][c]`

#### `fits(p, x, y) → boolean`
ピース `p` を座標 `(x, y)` に配置したとき衝突しないか判定する。  
- フィールド範囲外（左右・下）→ `false`
- `grid` の既存ブロックと重複 → `false`
- Y が負値（画面外上部）のセルは `grid` チェックをスキップする

#### `place()`
現在の `piece` を `grid` に書き込む（ピース固定）。

#### `clearLines()`
`grid` を下から走査し、横一列が埋まっている行を削除して上部に空行を追加する。  
同時消去数に応じたスコア加算とレベル更新を行う。

**スコア計算:**

| 消去数 | 基礎点 |
|--------|--------|
| 1 | 100 |
| 2 | 300 |
| 3 | 500 |
| 4（テトリス） | 800 |

加算点 = 基礎点 × `level`

**レベル計算:** `level = floor(lines / 10) + 1`

#### `spawnPiece()`
`nextPiece` を `piece` に昇格させ、新たな `nextPiece` を生成する。  
出現位置は横中央、`pieceY = -piece.length`（画面外上部）。  
出現直後に `fits` が `false` の場合 `gameOver()` を呼ぶ。

#### `ghostY() → number`
現在のピースをそのまま落下させた場合の最終 Y 座標を返す。  
ゴーストピース表示に使用。

#### `tick()`
ゲームループで毎インターバル呼ばれる処理。
1. ピースを 1 マス下に移動できるか判定
2. 移動可能 → `pieceY++`
3. 移動不可 → `place()` → `clearLines()` → `spawnPiece()`
4. `draw()` で再描画

**落下速度:**  
`setInterval` 間隔 = `max(100, 700 - (level - 1) × 60)` ms  
→ レベル1: 700ms、レベル10: 160ms、レベル11以降: 100ms（上限）

---

### 5.2 描画

#### `draw()`
Canvas 全体を再描画する。描画順:
1. 画面クリア
2. `grid` の固定ブロック
3. ゴーストピース（半透明白）
4. 落下中ピース
5. グリッド線（薄い線）

#### `drawBlock(context, x, y, color, size)`
1ブロックを描画する。上部に白ハイライトを重ねて立体感を表現。

#### `drawNext()`
`#nextCanvas` に次のピースをプレビュー表示する。  
120×120px のキャンバスに 24px ブロックで中央寄せ描画。

---

### 5.3 ゲーム制御

#### `startGame()`
ゲーム初期化処理。`grid`・各カウンタをリセットし、タイマー起動・BGM再生。

#### `gameOver()`
タイマー停止・BGM停止・GAME OVER メッセージ表示。

---

### 5.4 BGM（Web Audio API）

#### 構成

```
OscillatorNode (MELODY: square波)
    └─ GainNode (エンベロープ)
           └─ GainNode (masterGain: 0.15)
                  └─ destination

OscillatorNode (BASS: triangle波)
    └─ GainNode (エンベロープ)
           └─ GainNode (masterGain: 0.08)
                  └─ destination
```

#### 楽曲データ形式

```js
[ノート名, 拍数]  // 例: ['E5', 1], ['B4', 0.5]
```

BPM: 180。1拍 = 60/180 = 0.333 秒。

#### `playSequence(notes, type, gainVal, loop)`
音符配列をスケジューリングして再生する。  
`loop=true` の場合は `setInterval` で曲の長さごとに再スケジュールを繰り返す。  
各音符にエンベロープ（`exponentialRampToValueAtTime`）を適用してスタッカート感を出す。

#### ループ管理
`bgmNodes` 配列に `GainNode` と `setInterval` ID を格納し、`stopBGM()` で一括破棄する。

---

## 6. キー操作

| キー | 処理 |
|------|------|
| `←` | ピースを左に1マス移動 |
| `→` | ピースを右に1マス移動 |
| `↓` | ソフトドロップ（1マス下・+1点） |
| `↑` | ピースを時計回りに90°回転 |
| `Space` | ハードドロップ（即着地・+2点/マス） |

---

## 7. ゲームフロー

```
[START ボタン]
      │
      ▼
  startGame()
  ├─ grid 初期化
  ├─ spawnPiece()
  ├─ setInterval(tick, interval)
  └─ startBGM()
      │
      ▼ (毎インターバル)
    tick()
    ├─ fits(下)?
    │   ├─ Yes → pieceY++
    │   └─ No  → place()
    │              └─ clearLines()
    │              └─ spawnPiece()
    │                  └─ fits 不可? → gameOver()
    └─ draw()
      │
      ▼
  [GAME OVER]
  stopBGM()
  clearInterval()
```

---

## 8. ファイル構成

```
index.html
├── <style>   ... レイアウト・UIスタイル
├── <canvas id="board">    ... メインフィールド (300×600)
├── <canvas id="nextCanvas"> ... NEXTプレビュー (120×120)
└── <script>
    ├── 定数 (COLS, ROWS, BLOCK, COLORS, PIECES)
    ├── BGM モジュール (NOTE, MELODY, BASS, playSequence 等)
    ├── ゲームロジック (newGrid, rotate, fits, place 等)
    ├── 描画 (draw, drawBlock, drawNext)
    ├── ゲーム制御 (startGame, gameOver, tick)
    └── イベントハンドラ (keydown, startBtn click)
```
