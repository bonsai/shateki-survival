# 射的サバイバル — 宇宙転生レール型3D射的

**射的の名人が宇宙に転生し、AIが生み出す宇宙人を撃ち落とすレール型3Dシューティング。**

プレイヤーは自動で進む宇宙空間から照準を合わせて射撃する。AIが宇宙人の設計JSONを提案し、Blender `bpy` がローポリ3Dモデルを生成、検証済みアセットをPanda3Dへ取り込む。まずは固定の敵で遊べる最小ゲームを作り、その後にAI生成をつなぐ。

## 1. 採用STACK

| 層 | 技術 | 責務 |
|---|---|---|
| 3Dゲーム | **Panda3D + Python** | 3Dシーン、レール移動、カメラ、照準、射撃、当たり判定、敵、HP、スコア、wave、ゲーム状態の唯一の所有者 |
| 3Dアセット生成 | **Blender + bpy** | 検証済みAlienSpec JSONからローポリモデル・色・簡単なアニメーションを決定的に生成し、GLB/glTF等で書き出す |
| AI設計 | **LLM / VLM（任意）** | 宇宙人のテーマ・部位・配色・アニメーション候補を構造化JSONで提案。任意コードは生成・実行しない |
| 生成ジョブ/API | **Python + FastAPI（必要になってから）** | 設計・Blender生成の非同期ジョブ、状態、アセットマニフェストを管理。ゲーム状態は所有しない |
| エージェント接続 | **MCP（後段）** | 生成ジョブ作成・状態照会・承認済みアセット一覧など限定ツールを提供 |
| Computer Vision（後段） | **MediaPipe + OpenCV** | カメラから手・身体の位置を取得し、照準や回避入力に変換。マウスでも必ず遊べるようにする |
| 画像生成（任意） | **Imagen等** | コンセプト画像やテクスチャ案を作る。画像だけで完成3Dモデルが得られるとは仮定しない |
| 評価（後段） | **JSONL + pytest。必要ならKaggle** | AlienSpec、生成結果、難易度やゲームプレイの再現可能な評価 |
| データ契約 | **JSON Schema / JSONL** | AlienSpec、アセットmanifest、生成ジョブ、ゲームイベントの記録 |

**Java、Unity、Unreal、Godot、p5.jsはMVPの必須依存にしない。** ゲーム本体と生成ツールをPython中心にまとめ、最初から複数エンジンを接続しない。

## 2. アーキテクチャ

```text
LLM / VLM（設計候補のみ）
        │ AlienSpec JSON
        ▼
JSON Schema / Pydantic 検証
        │ valid spec
        ▼
Blender headless + bpy（固定スクリプト）
        │ low-poly model
        ▼
GLB/glTF 書き出し → 検証 → versioned manifest
        │ ready asset only
        ▼
Panda3D + Python ─────────── ゲーム状態の唯一の所有者
  ├─ レール型3Dカメラ・背景
  ├─ 照準・射撃・弾道
  ├─ 敵の移動・当たり判定・HP・スコア
  ├─ wave・難易度・標準敵へのフォールバック
  └─ 入力アダプター（マウス / MediaPipe）
        └─ MediaPipe + OpenCV: camera → landmarks → normalized aim / gesture events

後段: FastAPIジョブAPI / MCP
      生成依頼と状態確認のみ。フレームごとのゲーム制御はしない。
```

### 重要な境界

- **Panda3D** がゲームの実行時状態を所有する。HP、衝突、スコア、スポーンを別のPython API側で二重管理しない。
- **Blenderはオフライン／バックグラウンドのアセット生成器**。ゲームの各フレームでBlenderを起動しない。
- **AlienSpecはデータ契約**。AIが返した文字列をPythonコードやBlenderスクリプトとして実行しない。
- **見た目とゲーム能力は分離**。モデルは外観だけ、HP・速度・攻撃頻度などは検証済みJSONで管理する。
- **生成とプレイを分離**。生成が遅い・失敗・Blender未導入でも、同梱の標準敵でゲームを続けられる。
- GLB/glTFの読み込みは採用するPanda3Dバージョンとインポータで検証する。直接読み込みが不安定な場合は、Blender側からPanda3D互換形式へ変換するアダプターを追加する。

## 3. ゲームの最小ループ

1. プレイヤーは自動で進むレール型3D空間にいる。
2. マウスで照準を動かし、クリックで射撃する。
3. 敵が奥から出現し、決められたレーン／軌道を移動する。
4. 命中・撃破でスコアを獲得。敵の通過や攻撃で残機が減る。
5. waveごとに敵の組合せを変える。
6. まず固定のローポリ敵で完成させ、次にAlienSpecから生成した敵を追加する。

初期MVPではフル自由移動や複雑な物理を避ける。レール移動、照準、射撃、命中判定、敵の出現、スコアが遊べることを優先する。

## 4. AlienSpec（概念契約）

設計JSONは最低限、次の情報を持つ。

- `schema_version`, `alien_id`, `theme`
- `body_parts`: 許可されたプリミティブ形状と寸法
- `palette`: 色・発光色
- `animation_profile`: 許可済みの簡単な動き
- `gameplay`: HP、速度、スコア、攻撃頻度などの制約付きパラメータ

形状・部位数・寸法・ポリゴン数・マテリアル数・アニメーション種別に上限を設ける。最初は球、円錐、カプセル、楕円体、触手状パーツなどの組合せで生成し、AIによる任意トポロジー生成や自動リギングは後回しにする。

## 5. アセットのライフサイクル

`queued → designing → modeling → validating → ready | failed`

- 出力例：`assets/enemies/<alien_id>/<version>.glb` と対応するmanifest。
- manifestにはID、バージョン、spec hash、生成スクリプトのバージョン、ファイルサイズ、検証結果を記録。
- 検証前のファイルをゲームから参照しない。同名ファイルを無言で上書きしない。
- `failed` または未知のIDは、Panda3D内蔵の標準ローポリ敵へフォールバックする。

## 6. 実装順序

1. **P0 — Panda3Dのレール型3D射的コア**：固定敵、カメラ、照準、射撃、命中判定、wave、HP、スコア。マウスだけで遊べる。
2. **P1 — MediaPipe身体入力**：OpenCVでカメラを取得し、MediaPipeで手・身体ランドマークを推定。座標を正規化して照準へ変換し、必要なら明示的なジェスチャーで射撃する。カメラ利用不可時はマウスへ戻る。
3. **P2 — AlienSpecと検証**：JSON Schema/Pydantic、制約、drone/tentacle/bossのfixture。
4. **P3 — Blender headless生成**：固定の`bpy`スクリプトがfixtureからローポリ敵を生成。
5. **P4 — アセット検証・登録**：形式・サイズ・複雑度・manifestを検証し、versioned assetとして保存。
6. **P5 — Panda3D Asset Registry**：検証済みアセットをロードし、敵として表示。失敗時は標準敵に戻す。
7. **P6 — 非同期生成API / MCP**：必要になった時点でジョブキューと状態照会を追加。
8. **P7 — E2Eと評価**：AlienSpec → Blender → 検証 → Panda3D表示、失敗時のフォールバックをテスト。
9. **後段 — VLM / Imagen / Kaggle**：コアゲーム、身体入力、生成パイプラインが安定してから個別に追加.

## 7. 最初のファイル構成

```text
shateki-survival/
├── README.md
├── game/
│   ├── main.py
│   ├── scenes/
│   ├── actors/
│   ├── systems/
│   └── assets/
├── alien_pipeline/
│   ├── schemas/alien_spec.schema.json
│   ├── alien_spec.py
│   ├── blender_worker.py
│   └── validate_asset.py
├── blender/
│   └── generate_alien.py
├── assets/
│   ├── enemies/
│   └── manifest.json
├── data/
│   ├── enemy_specs/
│   └── eval/
└── tests/
```

まずはP0に必要なファイルだけを作る。MediaPipeはP1で独立した入力アダプターとして追加する。FastAPI、MCP、VLM、Imagen、Kaggleは後段の依存とし、コアゲームを動かすための必須条件にしない。

## 8. MVP受け入れ条件

- Panda3Dでレール型の3D射的を起動し、マウス照準と射撃で敵を撃破できる。
- MediaPipe入力を有効化すると、カメラから得たランドマークで照準を操作できる。カメラ停止時はマウスにフォールバックする。
- 敵の当たり判定、HP、スコア、waveが動作する。
- Blenderを起動しなくても、同梱の標準敵でプレイできる。
- 固定AlienSpecからローポリモデルを生成し、検証済みアセットだけをゲームに追加できる。
- 不正JSON、生成失敗、未登録アセット、読み込み失敗でゲーム全体が停止しない。
- ゲーム状態の所有者はPanda3Dだけで、生成APIはジョブとアセット情報だけを管理する。
- AI出力の任意コード実行を許可しない。
