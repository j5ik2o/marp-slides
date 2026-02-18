---
marp: true
theme: default
paginate: true
header: "コードレビューを最後の砦にするな"
footer: ""
style: |
  section {
    font-family: 'Noto Sans JP', 'Hiragino Sans', sans-serif;
  }
  section.title {
    text-align: center;
    justify-content: center;
  }
  section.title h1 {
    font-size: 2.2em;
  }
  section.title h2 {
    font-size: 1.2em;
    color: #666;
  }
  blockquote {
    border-left: 4px solid #e74c3c;
    padding-left: 1em;
    color: #555;
    font-style: italic;
  }
  table {
    font-size: 0.85em;
  }
  strong {
    color: #e74c3c;
  }
  code {
    font-size: 0.9em;
  }
---

<!-- _class: title -->
<!-- _paginate: false -->

# コードレビューを最後の砦にするな

## AI駆動開発時代のシフトレフト戦略

## 加藤潤一(@j5ik2o)

---

# プロフィール

![bg right:30% contain](./images/self-profile.jpg)

## 加藤潤一

- 2014-2024年: kubell（旧Chatwork）
- 2025年1月: 独立（IDEO PLUS合同会社 代表）
- DDD / 関数型 / 分散システム設計 / AI-DLC

---

# 今日のテーマ

> レビューが辛いのは、レビューのせいじゃない。

AI駆動開発時代に求められるのは、**開発プロセス全体の再設計**です。

本セッションでは、実プロジェクト（Rustアクターランタイム fraktor-rs）での
実践例をもとに、具体的なシフトレフト戦略を紹介します。

---

# よく聞く声

- 「AIが書いたコード、レビューが大変になった」
- 「生成量が増えて、追いつかない」
- 「AIのコード、微妙に意図と違う」
- 「レビューで指摘しても同じミスが繰り返される」

---

# 本当にAIのせい？

レビュー負荷が増大した原因を整理してみる

| 原因 | 本質的な問題 |
|------|-------------|
| 大量のコードが生成される | 要件の粒度が大きすぎる |
| 意図と違うコードが出る | 仕様が定義されていない |
| 同じ指摘を繰り返す | ルールが機械的に強制されていない |
| 設計が一貫しない | AIへの制約が不足している |

---

# コードレビューに何を期待しているか

## レビューで本来やるべきこと
- 設計判断の妥当性
- ビジネスロジックの正しさ
- アーキテクチャとの整合性

## レビューに押し付けられていること
- 命名規約の確認
- コーディングスタイルの統一
- 仕様との整合性チェック
- 既存パターンとの一致確認

---

# シフトレフトとは

```
従来:  要件 → 設計 → 実装 → テスト → レビュー ← ここで品質確認
                                           😰

シフトレフト:
       要件 → 設計 → 実装 → テスト → レビュー
       ↑      ↑      ↑      ↑
       品質    品質    品質    品質
       確認    確認    確認    確認
```

**品質保証を上流工程に移動する**

---

# AI駆動開発におけるシフトレフト

5つのレイヤーで品質を左にシフトする

1. **要件の分割と絞り込み**
2. **仕様駆動開発（Spec-Driven Development）**
3. **エージェントへのルール注入**
4. **カスタムリンター・静的解析**
5. **コードレビュー** ← 最後の砦ではなく最終確認

---

<!-- _class: title -->

# レイヤー1: 要件の分割と絞り込み

---

# 要件の規模感をコントロールする

最初から適切な粒度は分からない。**初回は外れることが多い。**

- 見積もりが外れたら終わりではなく、**そこから学習する**
- フィボナッチの相対見積もりで規模感を揃える
- 大きすぎたら分割し、次のスプリントで再見積もり

**これはスクラムで僕らがやってきたことと全く同じ。**
AIだからといって、このプロセスをサボってはいけない。

---

# 要件分割の粒度

fraktor-rs では**フィーチャー単位**で要件を定義している

```
要件（フィーチャー単位）      ← 人間が定義・見積もり
  例: "distributed-pubsub"
  例: "graphstage-core-execution"
  例: "cluster-membership-gossip-topology"
        ↓
タスク分解（仕様駆動で分割）  ← AIと人間で共同分解
  例: ClosedShape, Flow::from_function, ...
```

人間が出す要件はフィーチャー単位。
型・関数レベルへの分割は**仕様駆動のタスク分解フェーズ**で行う。

---

# 最初の一手: 何から作るか

分割したとして、**最初に何を作るか**が最も難しい

## 2つのアプローチ

**1. MVPで全体性を担保する**
- 後から追加が困難な機能セットを最初に作る
- 骨格（アーキテクチャの背骨）を先に通す

**2. コアロジックファースト**
- 外部依存のないコアロジックを先に作り、インフラは後から差し込む
- DDDならドメイン層、ライブラリなら純粋なアルゴリズム層

---

# 実例: fraktor-rs の core ファースト

fraktor-rs は `core`（no_std）→ `std`（Tokio等）の順で構築

```
modules/actor/src/
├── core/          ← 最初に作る（外部依存なし）
│   ├── actor/        純粋なアクターモデル
│   ├── supervision/  監視戦略
│   ├── messaging/    メッセージング
│   └── ...           DBもネットワークも不要
│
└── std/           ← 後から差し込む（実行環境依存）
    ├── tokio/        Tokioランタイム統合
    └── ...           プラットフォーム固有の実装
```

**コアの正しさを外部依存なしで検証してから、外側を足す**

---

<!-- _class: title -->

# TAKT: AIエージェントオーケストレーション

---

# TAKTとは

**T**AKT **A**gent **K**oordination **T**opology
https://github.com/nrslib/takt

AIエージェントの協調手順・介入ポイント・記録を**YAMLで定義する**

```bash
takt                       # 対話モードで要件を詰めてから実行
takt --task "機能を追加"     # 直接タスク実行
takt #6                    # GitHub Issueをタスクとして実行
takt --pipeline --auto-pr  # CI/CDパイプライン実行
```

**「誰が・何を見て・何を守り・何をするか」をワークフローとして宣言**

---

# Faceted Prompting: プロンプトの関心の分離

モノリシックなプロンプトを**5つの独立した関心**に分解し、宣言的に合成する

| 関心 | 答える問い | 例 |
|------|-----------|-----|
| **Persona** | *誰*として判断するか？ | planner, coder, reviewer |
| **Policy** | *何を*守るか？ | コーディング規約、禁止事項 |
| **Instruction** | *何を*するか？ | ステップ固有の手順 |
| **Knowledge** | *何を*参照するか？ | アーキテクチャ、ドメイン知識 |
| **Output Contract** | *どう*出すか？ | レビューレポートの形式 |

**1ファイル = 1つの関心 → 変更が他のファセットに波及しない**

---

# ピース: ワークフローをYAMLで定義する

<style scoped>
pre { font-size: 0.7em; }
</style>

```yaml
name: default
initial_movement: plan
movements:
  - name: plan
    persona: planner          # WHO
    knowledge: architecture   # CONTEXT
    edit: false
    rules:
      - condition: 要件が明確で実装可能
        next: implement
      - condition: 要件が不明確、情報不足
        next: ABORT
  - name: implement
    persona: coder            # WHO
    policy: [coding, testing] # RULES
    knowledge: architecture   # CONTEXT
    edit: true
    rules:
      - condition: 実装完了
        next: ai_review
      - condition: 進行不可
        next: ABORT
```

---

<!-- _class: title -->

# レイヤー2: 仕様駆動開発

---

# AIにコードを書かせる前に

> 「何を作るか」を定義せずに「作って」と言っていないか？

仕様がなければ：
- AIは「それっぽい」コードを生成する
- レビュアーは「何が正しいか」の基準がない
- 指摘が主観的になる

---

# TAKTを使った仕様駆動開発

---

# SDDの各フェーズをTAKTピースに落とし込む

<style scoped>
table { font-size: 0.75em; }
</style>

| Phase | ピース | ペルソナ | やること |
|-------|--------|---------|---------|
| 1 | `sdd-requirements` | requirements-analyst | EARS形式で要件生成 |
| 1.5 | `sdd-validate-gap` | planner | 既存コードとのギャップ分析 |
| 2 | `sdd-design` | planner | 技術設計 + 発見ログ |
| 2.5 | `sdd-validate-design` | architecture-reviewer | GO/NO-GO 判定 |
| 3 | `sdd-tasks` | planner | 実装タスク分解 |
| 4 | `sdd-impl` | coder + reviewers | 実装 + 多段階レビュー |
| 5 | `sdd-validate-impl` | arch + qa + supervisor | 並列検証 |

**各フェーズを個別にも、全自動（`sdd`）でも実行可能**

---

# Phase 1: EARS形式で要件を生成する

```bash
takt -w sdd-requirements  # Phase 1 のみ実行
```

```yaml
# sdd-requirements.yaml
movements:
  - name: generate-requirements
    persona: requirements-analyst   # 要件分析の専門家
    policy: ears-format             # EARS形式を強制
    instruction: generate-requirements
    output_contracts:
      report:
        - name: requirements.md
          format: sdd-requirements
    rules:
      - condition: 要件が明確で完全
        next: COMPLETE
      - condition: 情報不足、要件が不明確
        next: ABORT
```

---

# EARS形式: テスト可能な要件を書く

<style scoped>
table { font-size: 0.8em; }
</style>

| パターン | 構文 |
|---------|------|
| イベント駆動 | [イベント]が起きたとき、[システム]は[応答]しなければならない |
| 状態駆動 | [条件]ならば、[システム]は[応答]しなければならない |
| 望ましくない振る舞い | [望ましくないイベント]が発生した場合、[システム]は[応答]しなければならない |
| 常時 | [システム]は常に[応答]しなければならない |

**曖昧な「適切に」「迅速に」は REJECT → 検証可能な基準に置き換える**

---

# Phase 2-3: 設計 → 品質ゲート → タスク分解

```bash
takt -w sdd-design           # 技術設計を生成
takt -w sdd-validate-design  # GO/NO-GO 判定
takt -w sdd-tasks            # タスクに分解
```

```yaml
# sdd-validate-design.yaml
- name: validate-design
  persona: architecture-reviewer    # 設計レビュー専門家
  policy: sdd-design-review         # 設計品質基準
  rules:
    - condition: GO（設計品質十分）
      next: COMPLETE
    - condition: NO-GO（重大な問題あり）
      next: ABORT
```

**設計が品質ゲートを通過しないと次へ進めない**

---

# Phase 4-5: 実装 → 多段階レビュー → 検証

```bash
takt -w sdd-impl             # 実装 + AIレビュー
takt -w sdd-validate-impl    # 並列検証（arch + qa + impl）
```

```yaml
# sdd-validate-impl.yaml — 3名の専門家が並列検証
- name: validate
  parallel:
    - name: arch-review         # アーキテクチャレビュー
      persona: architecture-reviewer
    - name: qa-review           # QAレビュー
      persona: qa-reviewer
    - name: impl-validation     # 要件・設計との整合性検証
      persona: supervisor
  rules:
    - condition: all("approved","approved","検証合格")
      next: COMPLETE
    - condition: any("needs_fix","検証不合格")
      next: fix
```

---

# フルオート: sdd.yaml で全フェーズを自動遷移

```bash
takt -w sdd  # 要件→ギャップ分析→設計→レビュー→タスク→実装→検証
```

```
要件生成 → ギャップ分析 → 設計生成 → 設計レビュー → タスク生成
                                      ↓ NO-GO                ↓
                                    設計やり直し     plan → implement → review
                                                      ↑                   ↓
                                                      └── fix ←── supervise
```

**人間の介入なしで全フェーズを自動実行し、品質ゲートで制御**

---

# 各フェーズで使われるファセット

<style scoped>
table { font-size: 0.75em; }
</style>

| ファセット種別 | ファイル | 役割 |
|--------------|---------|------|
| **Persona** | requirements-analyst.md | EARS要件の専門家 |
| **Policy** | ears-format.md | EARS記述規約の強制 |
| **Policy** | sdd-design-review.md | GO/NO-GO判定基準 |
| **Instruction** | generate-requirements.md | 要件生成の手順 |
| **Knowledge** | design-discovery.md | 設計時の発見記録ガイド |
| **Output Contract** | sdd-requirements.md | 要件ドキュメントの出力形式 |

**ファセットを差し替えるだけで、SDDプロセスをカスタマイズできる**

---

<!-- _class: title -->

# レイヤー3: エージェントへのルール注入

---

# AIは「教科書的に正しい」コードを書く

しかし、プロジェクト固有のパターンは知らない

- 命名規約
- エラー処理のスタイル
- アーキテクチャの層構造
- 使用するライブラリ・パターン

→ **レビューで毎回指摘する羽目になる**

---

# ルール注入の3層構造

| 層 | 仕組み | 役割 | 例 |
|----|--------|------|-----|
| 1 | CLAUDE.md | プロジェクト全体の方針 | 言語、設計哲学、CI手順 |
| 2 | Rules | ファイルパターン別の規約 | 不変性、命名、CQS |
| 3 | Skills | 特定タスクの専門知識 | 型設計、ギャップ分析 |

fraktor-rs の実装：
- **CLAUDE.md**: 30行の簡潔な方針（日本語、Less is more、YAGNI）
- **Rules**: 13個（共有8 + Rust固有5）
- **Skills**: 31個（プロジェクト固有含む）

---

# ルールの実例: learning-before-coding

```markdown
# コーディング前の学習

新しいコードを書く前に既存の実装を分析する。

## 必須ワークフロー
1. 類似コードの特定（同じレイヤー、同じ種類）
2. 2〜3個の類似実装を分析
3. パターンに従って実装

## 禁止事項
- プロジェクトが直接クラスなのにインターフェースを追加
- 手動DIなのにDIフレームワークを使用
- コメントがないのにJSDocを追加
```

**「既存のコードベースこそがプロジェクト規約のドキュメント」**

---

# ルールの実例: Rust固有ルール

fraktor-rs では言語固有の設計ルールも定義

| ルール | 内容 |
|--------|------|
| immutability-policy | `&mut self` 優先、interior mutability は原則禁止 |
| cqs-principle | Query は `&self` + 戻り値、Command は `&mut self` |
| naming-conventions | Manager/Util/Service 等の曖昧サフィックス禁止 |
| type-organization | 1公開型 = 1ファイル（例外条件も明文化） |
| reference-implementation | Pekko/protoactor-go からの移植ルール |

**型シグネチャ自体が設計意図を表現する**

---

# プロジェクト固有のスキル

汎用スキルだけでなく、**プロジェクト固有のスキル**が威力を発揮する

| スキル | 用途 |
|--------|------|
| designing-fraktor-shared-types | 共有型の設計支援（ArcSharedパターン） |
| creating-fraktor-modules | 7つのDylintルールに準拠したモジュール生成 |
| reviewing-fraktor-types | 過剰設計チェック（参照実装の1.5倍ルール） |
| pekko-gap-analysis | Pekko との差分分析と優先度付け |

**AIに「このプロジェクトの専門家」としての知識を与える**

---

# ルール注入の効果

| 観点 | ルールなし | ルールあり |
|------|-----------|-----------|
| 命名 | バラバラ | 統一される |
| 構造 | AIの好み | プロジェクトに準拠 |
| パターン | 教科書的 | 既存コードと一致 |
| レビュー指摘 | 毎回同じ | 大幅に減少 |

**「同じ指摘を繰り返す」問題の根本解決**

---

<!-- _class: title -->

# レイヤー4: カスタムリンター・静的解析

---

# 人間がレビューで指摘すべきでないもの

- コーディングスタイル → **フォーマッター**
- 型安全性 → **型チェッカー**
- 命名規約 → **カスタムリンター**
- 構造パターン → **カスタムリンター**
- セキュリティ → **静的解析**

**機械的に検出できるものは、機械に任せる**

---

# 実例: fraktor-rs の8つのDylintリンター

| リンター | 強制する設計ルール |
|---------|------------------|
| type-per-file-lint | 1ファイル1公開型 |
| mod-file-lint | mod.rs 禁止（2018モジュール規約） |
| module-wiring-lint | FQCN import 強制 |
| tests-location-lint | テストは `<name>/tests.rs` に配置 |
| use-placement-lint | use 文はファイル先頭 |
| rustdoc-lint | rustdocは英語、コメントは日本語 |
| cfg-std-forbid-lint | coreモジュールで `#[cfg(feature="std")]` 禁止 |
| ambiguous-suffix-lint | Manager/Util/Service 等の命名禁止 |

---

# AIエージェント向けリンターの特徴

通常のリンターとの決定的な違い：**エラーメッセージがAIへの修正指示**

```
// 通常のリンター
❌ "Naming convention violation"

// AI向けリンター（fraktor-rs の type-per-file-lint）
✅ "このファイルには複数の公開型が含まれています。
    PublicStruct1 と PublicStruct2 はそれぞれ別ファイルに
    分離してください。ファイル名は snake_case の型名に
    一致させてください。"
```

**AIが自律的に修正できるエラーメッセージ設計**

---

# Pre-Edit Lint: 編集前にリンターを実行する

fraktor-rs 独自のプラクティス

```bash
# ステップ1: 編集する前にリンターを実行
./scripts/ci-check.sh dylint -m actor

# ステップ2: 失敗したルールだけ読む（ルール全部は読まない）

# ステップ3: 違反を修正してからコードを書く

# ステップ4: 全タスク完了後
./scripts/ci-check.sh all
```

**構造的な問題を「書く前」に検出し、修正コストを最小化する**

---

# 自動化のピラミッド

```
           /\
          /  \        ← コードレビュー（人間の判断）
         /    \          設計妥当性・ビジネスロジック
        /------\
       /        \     ← カスタムリンター 8個 + Clippy
      /          \       命名・構造・モジュール規約
     /------------\
    /              \  ← Rules 13個 + Skills 31個
   /                \    不変性・CQS・既存パターン準拠
  /------------------\
 /                    \← TAKT Knowledge + Steering
/______________________\  要件定義・受入基準・プロダクトビジョン
```

**下層が堅固なほど、レビューで議論すべきことが減る**

---

<!-- _class: title -->

# まとめ: 開発プロセスの再設計

---

# シフトレフト戦略の全体像

| レイヤー | やること | fraktor-rs での実装 |
|---------|---------|-------------------|
| 要件分割 | タスクを小さく絞り込む | TAKT 対話モード + ピース実行 |
| 仕様定義 | TAKTピースで基準を明確化 | ピースYAML + ファセット |
| コンテキスト | AIに永続的知識を与える | TAKT Knowledge ファセット |
| ルール注入 | Rules/Skillsで規約を強制 | 13 Rules + 31 Skills |
| 自動検査 | カスタムリンター | 8 Dylint + Clippy |
| レビュー | 設計判断に集中 | **本来の役割に集中** |

---

# fraktor-rs の実績

| 指標 | 数値 |
|------|------|
| 完了した仕様 | 32件（TAKT ピース実行） |
| ルール | 13個（共有8 + Rust固有5） |
| スキル | 31個（うちプロジェクト固有4個） |
| カスタムリンター | 8個（Dylint） |
| CI チェック項目 | 7カテゴリ（format, lint, dylint, clippy, test, doc, embedded） |

**32件の仕様をTAKTで駆動し、レビュー負荷を最小化**

---

# レビュアーが集中すべきこと

## 自動化で排除した後に残るもの

- この設計判断は妥当か？
- ビジネスロジックは正しいか？
- 参照実装との乖離は許容範囲か？
- セキュリティ上の懸念はないか？
- 過剰設計になっていないか？（型数が参照の1.5倍を超えていないか）

**これこそが人間にしかできないレビュー**

---

# 実践のステップ

1. **まず**: CLAUDE.md / Rulesを整備する（即効性が高い）
2. **次に**: TAKT Knowledge ファセットでプロジェクト知識を整理する
3. **そして**: 要件を小さく分割し、TAKTピースで仕様化する
4. **さらに**: カスタムリンターで機械的チェックを自動化する
5. **加えて**: プロジェクト固有のスキルを作成する
6. **最後に**: レビュープロセスを再定義する

---

# 最後に

> レビューが辛いのは、レビューのせいじゃない。

AI駆動開発時代のコードレビューは、**最後の砦**ではなく**最終確認**であるべき。

そのために必要なのは、レビューの改善ではなく、**開発プロセス全体の再設計**。

品質は上流で作り込む。

---

<!-- _class: title -->

# ありがとうございました

ご質問・ご意見をお待ちしています
