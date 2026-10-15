# 射的サバイバル — 宇宙転生インベーダー

**射的の名人が宇宙に転生し、宇宙人と戦うシューティングゲーム。**

既存のインベーダーゲームを、姿勢入力・生成AIの敵・適応的な敵編成・宇宙的な演出でリッチにするプロジェクト。まずはローカルPCで動く小さな縦切りPoCを作り、生成系AIが止まってもゲーム本体は遊べる設計にする。

## 1. 推奨スタック

| 層 | 技術 | 責務 |
|---|---|---|
| Frontend / Renderer | **p5.js + TypeScript（段階導入）** | ゲーム画面、照準、弾、敵スプライト、パーティクル、HUD、入力の表示 |
| Local Backend / API | **Python + FastAPI + WebSocket** | ゲーム状態、敵スポーン指示、イベント配信、モデル/APIとの境界 |
| Computer Vision | **MediaPipe Tasks / Python SDK** | カメラから身体・手のランドマークを取得。射的の照準や回避ジェスチャーに変換 |
| CV / VLM | **OpenCV + 必要時のみVLM** | カメラ入力の前処理。VLMはラウンド間の状況要約・敵テーマ選択など低頻度用途に限定 |
| Image Generation | **Imagen等の画像生成API** | 敵・ボス・背景の素材生成。ゲーム中は非同期で生成し、完成済み素材をキャッシュ |
| Enemy Director | **Pythonのルールエンジン → MCP Server** | 難易度、敵タイプ、スポーン位置、波状攻撃、ボス出現を管理。MCPは明示的なツール境界として提供 |
| Offline ML | **Kaggle（GPU Notebook）** | 敵の特徴・行動パラメータ・難易度制御モデルの学習と評価。学習済み成果物をローカルへ持ち帰る |
| Game Logic Engine | **初期はPythonの状態機械。Godotは第2段階で評価** | まずゲームルールを単純な状態機械で実装。物理、衝突、複雑なシーン、3D演出が必要になった時点でGodotへ移行を判断 |
| Asset / Data | **JSON + PNG/WebP + JSONLログ** | 敵定義、生成条件、素材メタデータ、ラウンド履歴、評価記録 |
| Protocol | **WebSocket + JSON schema** | p5.jsとPythonの疎結合な通信契約 |

### Godotを最初から入れるべきか

**初期版では入れない方針を推奨。** p5.jsをフロントにするなら、Godotをゲームロジック専用に追加すると、p5.js ↔ Godot ↔ Pythonの同期・デバッグが増えます。最初はPythonの状態機械を権威あるゲーム状態として、p5.jsは描画と入力表示に集中させます。

次の条件が出たらGodotを再評価します。

- 3D空間、複雑な当たり判定、物理演算、シーン遷移が主要機能になった
- 敵や弾の数が増えてp5.js側の管理が難しくなった
- エディタ、アニメーションツリー、エクスポート先の利点が追加の接続コストを上回る

Godotを採用する場合は、**ゲームロジックの所有者をGodotに一本化**し、PythonはCV・生成・敵編成提案、p5.jsは独立した演出ビューに限定する。PythonとGodotの両方でHP・衝突・スポーン状態を二重管理しない。

## 2. アーキテクチャ

```text
Camera
  ↓
Python / MediaPipe ── pose features ───────────┐
                                               ↓
p5.js UI  ←── WebSocket / JSON ──→ Python Game State
  ↑                                    │
  │ render only                        ├── Rule-based Enemy Director
  │                                    ├── MCP tools: spawn / set wave / inspect state
  │                                    ├── VLM: round-boundary scene understanding (optional)
  │                                    └── Imagen API: async enemy/background generation
  │                                                  ↓
  └──────────── cached PNG/WebP assets ←── Asset Registry
                                                     ↑
                                  Kaggle offline training/evaluation
```

**リアルタイム経路と生成経路を分離する。**

- リアルタイム経路：入力取得 → ゲーム状態更新 → WebSocket → 描画。毎フレーム外部AIを待たない。
- 生成経路：敵の設計 → 画像生成 → 検証・縮小・透過処理 → キャッシュ登録。生成に失敗しても既定の敵で続行。
- 学習経路：Kaggleでオフライン学習・評価 → バージョン付きモデル／JSONパラメータをローカルへ配布。
- MCP経路：ゲーム状態を直接書き換える自由な接続ではなく、許可されたコマンドを検証して実行する。

## 3. ゲームコンセプト

- **主人公**：射的の名人。異世界転生後、宇宙の射的技術で侵略者を迎撃する。
- **基本ループ**：照準を合わせる → 撃つ → 敵の弱点を見抜く → 素材・スコアを獲得 → 次の敵編成へ。
- **入力**：最初はマウス／タッチで遊べるようにし、MediaPipeによる身体入力を追加オプションにする。
- **敵**：Imagenで見た目を生成し、敵の当たり判定・HP・速度・攻撃パターンはJSON定義で管理する。画像そのものにゲームルールを持たせない。
- **敵の生成制御**：VLMや学習モデルはテーマ・難易度の候補を提案し、スポーン可否・上限・攻撃頻度はルールエンジンが検証する。
- **リッチ化**：パララックス背景、弾道トレイル、命中エフェクト、撃破演出、ボスの段階変化、ラウンド間の物語演出。

## 4. 疎結合のデータ契約

### p5.js → Python（入力・操作）

```json
{
  "type": "player.action",
  "seq": 42,
  "action": "fire",
  "aim": { "x": 0.62, "y": 0.31 },
  "source": "mouse"
}
```

### Python → p5.js（描画状態）

```json
{
  "type": "game.snapshot",
  "tick": 120,
  "wave": 2,
  "player": { "hp": 3, "score": 120, "aim": { "x": 0.62, "y": 0.31 } },
  "enemies": [
    { "id": "enemy-001", "kind": "drone", "x": 0.7, "y": 0.2, "hp": 2, "asset": "enemies/drone-001.webp" }
  ]
}
```

座標は0〜1の正規化座標。各メッセージに `type` と `seq` / `tick` を持たせ、再送・古い状態・不正な値を処理できるようにする。具体的なJSON SchemaはMVP実装時に追加する。

## 5. 敵生成 MCP Server

MCPは生成AIそのものではなく、エージェントが使える**制限付きゲーム操作API**として設計する。

最小ツール候補：

- `get_game_state()` — 現在のwave、敵数、難易度、プレイヤー残機を読む
- `propose_enemy(spec)` — 敵定義の候補を検証する（即時スポーンはしない）
- `spawn_enemy(enemy_id, lane)` — 許可された敵を上限内で出現させる
- `set_wave_preset(preset_id)` — 登録済み編成を適用する
- `request_enemy_asset(theme, constraints)` — 画像生成ジョブをキューに登録する
- `get_asset_job(job_id)` — 生成状態を確認する

安全柵：敵数、HP、速度、発射頻度、素材サイズ、生成時間に上限を設ける。AIが返したJSONはスキーマ検証し、未知のコマンドや任意コード実行を許可しない。

## 6. Imagen・VLM・Kaggleの役割分担

- **Imagen**：見た目の素材生成。ゲームの進行や毎フレームの制御には使わない。
- **VLM**：画像・状態の意味的な解釈が必要な時だけ使う。カメラ映像から子どもの感情や性格を推測する設計はしない。
- **Kaggle**：敵の行動パラメータやwave構成をオフラインで学習・比較する。ゲーム本体からKaggle Notebookをリアルタイム呼び出ししない。
- **MediaPipe**：低遅延なランドマーク取得。まずは手や上半身の位置から照準・回避の単純な特徴量を作る。
- **ルールエンジン**：最終的なゲーム状態の更新を担当し、生成AIは候補提案に限定する。

## 7. 実装フェーズ

1. **P0 — Playable core**：p5.jsの既存インベーダー風ゲーム、マウス照準、発射、衝突、HP、スコア、wave。画像素材は仮素材でよい。
2. **P1 — Local backend**：FastAPI、WebSocket、サーバー側状態管理、JSON契約、再接続時のsnapshot。
3. **P2 — MediaPipe**：Pythonカメラ入力と照準／回避特徴量。カメラがなくてもマウスで全機能をテスト可能にする。
4. **P3 — Enemy asset pipeline**：Imagenの非同期ジョブ、生成結果の検証、キャッシュ、フォールバック素材。
5. **P4 — MCP director**：状態取得、編成提案、制限付きspawn、ログとリプレイ。
6. **P5 — Kaggle evaluation**：難易度・敵編成の評価データを用意し、ベースラインと学習モデルを比較。
7. **P6 — Godot decision**：複雑な物理／3D／シーン管理が必要かを実測し、採用ならゲーム状態の所有者をGodotに移す設計を別途決定。

## 8. MVPの受け入れ条件

- カメラや生成APIがなくても、マウス操作で1ラウンド最後まで遊べる。
- ゲーム状態は単一の所有者が更新し、フロントは受け取ったsnapshotを描画する。
- 敵の見た目（asset）とゲーム上の能力（JSON定義）が独立している。
- 生成AIが遅い／失敗した場合はキャッシュまたは仮素材で続行できる。
- MediaPipe、MCP、VLM、Imagenの各機能を個別に無効化してもコアゲームが起動する。
- カメラ映像はデフォルトで保存しない。ログにはゲームイベントと数値のみを記録する。

## 9. 最初に作るファイル構成

```text
shateki-survival/
├── README.md
├── frontend/
│   ├── index.html
│   ├── src/
│   │   ├── main.ts
│   │   ├── render/
│   │   ├── input/
│   │   └── net/
│   └── package.json
├── backend/
│   ├── app.py
│   ├── game/
│   │   ├── state.py
│   │   ├── rules.py
│   │   └── enemy_director.py
│   ├── vision/
│   ├── generation/
│   ├── mcp_server/
│   └── schemas/
├── assets/
│   ├── enemies/
│   ├── backgrounds/
│   └── manifest.json
├── data/
│   ├── enemy_specs/
│   └── eval/
└── notebooks/
    └── kaggle/
```

この構成は初期の論理分割です。実装を始める際は、まずP0/P1に必要なファイルだけを作り、空のディレクトリや将来機能を先に増やさないでください。
