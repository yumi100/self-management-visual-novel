# 星屑ノベル ― エンジンリファレンス

単一HTMLファイルで完結するノベルゲームエンジン。シナリオモード（起承転結のある一本道〜分岐構造）と、エンドレスモード（学習型AIが同一/巡回シーンを無限生成し続ける開発者向け検証モード）の2種類のプレイモードを持つ。

このドキュメントは、シナリオを書く「使用者」と、エンジン本体を改造する「開発者」の両方に向けたリファレンス。

---

## 目次

1. [起動と開発者モード](#1-起動と開発者モード)
2. [全体アーキテクチャ](#2-全体アーキテクチャ)
3. [localStorage キー一覧](#3-localstorage-キー一覧)
4. [シナリオJSONの書き方（使用者向けリファレンス）](#4-シナリオjsonの書き方使用者向けリファレンス)
5. [隠しタグ（[[key:value]]）システム](#5-隠しタグkeyvalueシステム)
6. [状況タグカテゴリ（tagCategories）](#6-状況タグカテゴリtagcategories)
7. [学習型ダイアログメモリシステム（DialogueMemorySystem）](#7-学習型ダイアログメモリシステムdialoguememorysystem)
8. [CGアンロック条件（unlockIf）](#8-cgアンロック条件unlockif)
9. [好感度システム（affection）](#9-好感度システムaffection)
10. [表情差分（faceタグ）](#10-表情差分facetグ)
11. [エンドレスモード](#11-エンドレスモード)
12. [アセット管理（キャラ/背景/BGM/SE/CG）](#12-アセット管理キャラ背景bgmsecg)
13. [セーブ/ロード](#13-セーブロード)
14. [開発者パネル機能一覧](#14-開発者パネル機能一覧)
15. [主要関数リファレンス（開発者向け）](#15-主要関数リファレンス開発者向け)
16. [既知の設計上の注意点・制約](#16-既知の設計上の注意点制約)

---

## 1. 起動と開発者モード

- 単一の `.html` ファイルをブラウザで開くだけで動作する（ビルド不要）。
- **開発者モードは URL に `?dev=1` または `?dev=true` を付けてアクセスした場合のみ有効**になる。セッションやlocalStorageには記憶されない（毎回URLパラメータで指定する必要がある）。
  ```
  file:///path/to/game.html?dev=1
  ```
- 開発者モードでは：
  - タイトル画面左上に `DEV` バッジが表示される
  - タイトル画面に「開発者パネル」ボタンが出現する
  - 画面左上に好感度・mood HUD（`#statusHud`）が表示される
  - セリフ表示中に隠しタグが `[[key:value]]` の形でそのまま可視化される（通常プレイでは非表示）
  - `aiChoice` 行に♡/×フィードバックUI（調教バー）が出せる

---

## 2. 全体アーキテクチャ

エンジンは1つのIIFE（即時実行関数）内に定義されており、大きく以下のモジュールで構成される（コード内の見出しコメント番号に対応）。

| # | モジュール | 役割 |
|---|---|---|
| 0 | 開発者モード判定 | `?dev=1` の解決 |
| 1 | サウンドエンジン | Web Audio APIベースのBGM/SE再生（`SoundEngine`） |
| 2 | 学習型ダイアログメモリシステム | `DialogueMemorySystem` クラス（aiChoiceの学習） |
| 3 | アセット定義 | キャラ・背景・BGM・SE・CGギャラリーの定義と永続化 |
| 4 | シナリオデータ | `defaultScript`（デフォルトシナリオJSON）と読み込み/保存 |
| 5 | 隠しタグパーサー | `[[key:value]]` の検出・タグハンドラーレジストリ |
| 5.5 | 状況タグカテゴリ | `tagCategories`（mood/affectionなど排他グループの定義） |
| 6 | 設定 | BGM/SE音量、テキスト速度、調教UI表示トグル |
| 7 | モード管理 | シナリオモード / エンドレスモードの切り替え |
| 8 | エンジン本体 | DOM要素参照、ゲーム進行ロジック本体 |
| 9 | 開発者パネル配線 | 開発者パネルの各UIとイベントハンドラ |

### 状態の置き場所（3系統）

このエンジンには「状態」を持つ場所が3系統あり、役割が異なる。改造する際に混同しやすいので明記する。

| 状態 | 変数 | 永続化キー | 役割 |
|---|---|---|---|
| グローバル進行状態 | `state`（`label`, `index`, `backlog`, `flags`, `onStage`, `bgKey` など） | セーブスロット内（`SAVE_KEY`） | 「今どのラベルの何行目か」「画面に誰が立っているか」等、1回のプレイスルーに紐づく進行状況 |
| キャラごとの内部状態 | `hubs[charKey].coreState`（`mood`, `affection`） | `MEMORY_KEY` | 好感度・moodなど、キャラごとに持つパラメータ。**好感度の単一の真実源はここ。** |
| 学習スコア | `hubs[charKey].scenes[sceneName]`（`DialogueMemorySystem`のインスタンス） | `MEMORY_KEY`（`hubs`と同じキーにまとめて保存） | `aiChoice`行のセリフ候補ごとの好みスコア |

> **重要**: 好感度（affection）はかつて `state.affection[charKey]` にも複製されていたが、リファクタにより廃止された。現在は `hub.coreState.affection` のみが真実源であり、読み取りは `getAffection(charKey)` 関数を使う。

---

## 3. localStorage キー一覧

全キーは `stardustNovel_` プレフィックスで統一されている（他サイトのデータと混ざらない）。

| キー | 内容 | 書き込み関数 | 読み込み関数 |
|---|---|---|---|
| `stardustNovel_tagCategories_v1` | `tagCategories`（mood/affection等のカテゴリ定義） | `persistTagCategories()` | `loadTagCategories()` |
| `stardustNovel_dialogueMemory_v1` | 全キャラの `hubs`（coreState + 学習スコア） | `persistAllHubs()` | `loadAllHubs()` |
| `stardustNovel_assets_v1` | 背景・キャラ・CG・BGM・SEの定義（画像パス等） | `persistAssets()` | `loadAssets()` |
| `stardustNovel_scenario_v1` | シナリオJSON全体（`script`） | `persistScenario()`（開発者パネル経由） | `loadScenario()` |
| `stardustNovel_options_v1` | 音量・テキスト速度・調教UI表示トグル等 | `persistOptions()` | `loadOptions()` |
| `stardustNovel_gamemode_v1` | `'scenario'` または `'endless'` | `persistGameMode()` | `loadGameMode()` |
| `stardustNovel_saves_v2` | セーブスロットのデータ一式 | `writeAllSaves()` | `loadAllSaves()` |
| `stardustNovel_gallery_v1` | CG解禁状況（`unlocked`フラグのみ） | `persistGalleryUnlocks()` | `loadGalleryUnlocks()` |
| `stardustNovel_slotcount_v1` | セーブスロット数 | `persistSlotCount()` | `loadSlotCount()` |

### 全データ削除

開発者パネル「セーブ設定」タブの「全セーブ/学習データを消去」ボタンは、以下の3キーのみを削除する（シナリオ編集・アセット設定・オプション設定は保持される）：

```js
[SAVE_KEY, MEMORY_KEY, GALLERY_KEY].forEach(k => localStorage.removeItem(k));
```

完全初期化したい場合は、ブラウザのコンソールで以下を実行する：

```js
Object.keys(localStorage)
  .filter(k => k.startsWith('stardustNovel_'))
  .forEach(k => localStorage.removeItem(k));
```

---

## 4. シナリオJSONの書き方（使用者向けリファレンス）

シナリオは `script` オブジェクト（`{ ラベル名: [行の配列], ... }`）で構成される。各ラベルは行（line）の配列を持ち、`state.label` と `state.index` で現在位置を管理する。

### 行（line）の型一覧

| 型 | 記法 | 説明 |
|---|---|---|
| 通常セリフ | `{ name, text }` | `name`省略で地の文になる |
| 演出行（併記可） | `{ bg, bgm, se, show:[{char,pos}], hide:[pos] }` | 他のプロパティと同時に書ける（`applyStageDirections`が処理） |
| ジャンプ | `{ text: "jump", goto: "label名" }` | 即座に指定ラベルの0行目へ移動 |
| プレイヤー選択肢 | `{ text, choices:[{text, goto, affection, mood}] }` | `choices`が**オブジェクト配列**。ボタンとして表示され、クリックで`goto`先へ、`affection`/`mood`があれば`bumpAffection(MAIN_CHAR, ...)`が呼ばれる |
| AI駆動セリフ | `{ aiChoice:true, char, scene, timeOfDay, choices:[...], name?, extraTags? }` | `choices`が**文字列配列**。学習型に自動選択される（詳細は[7章](#7-学習型ダイアログメモリシステムdialoguememorysystem)） |
| CGアンロック | `{ text: "cgUnlock", cg, se?, unlockIf? }` | CGギャラリーを解禁する特殊行（詳細は[8章](#8-cgアンロック条件unlockif)） |
| 好感度分岐ジャンプ | `{ text: "jump-affection-check", goto: null }` | `pickEndingLabel()`の閾値判定で`ending_good/normal/cool`のいずれかへ自動遷移する特殊行 |
| 終了 | `{ text: "end" }` | タイトル画面へ戻る |

### `choices` の型に注意（2種類ある）

同じ`choices`というキー名だが、**行の種類によって中身の型が違う**。

- **プレイヤー選択肢**（`line.choices`、`aiChoice`なし）：オブジェクト配列 `[{text, goto, affection, mood}, ...]`
- **AI駆動セリフ**（`line.aiChoice === true`）：文字列配列 `["セリフ1", "セリフ2", ...]`

この2つを混同すると正しく動作しないので注意する。

### `show` / `hide`（キャラの表示制御）

```js
{ show: [{ char: "akari", pos: "right" }], name: "アカリ", text: "……こんなところで、何してるの。" }
```

- `pos` は `"left" | "center" | "right"` の3スロット
- `hide: ["right"]` のように書くとそのポジションのキャラを退場させる

### 演出プロパティ一覧（`applyStageDirections`が処理）

| プロパティ | 効果 |
|---|---|
| `bg` | 背景を切り替え（0.48秒クロスフェード） |
| `bgm` | BGMを切り替え（0.8秒クロスフェード、ループ再生） |
| `se` | 効果音を1回再生 |
| `show` | キャラをそのポジションに表示 |
| `hide` | キャラをそのポジションから退場 |

---

## 5. 隠しタグ（`[[key:value]]`）システム

セリフの`text`文字列末尾などに `[[key:value]]` の形式で埋め込むと、表示前に自動的に検出・除去され、対応する効果が発動する。

```js
{ name: "アカリ", text: "……たまにね。誰もいないから、落ち着くの。[[affection:+2]]" }
```

- シナリオモードのプレイヤー画面では**タグ文字列は自動で除去されて表示**される
- **開発者モードでは**タグが `[[key:value]]` の形のまま行末に可視表示される（`finishTyping`内、`IS_DEV`判定）
- 開発者パネルの「隠しタグ シミュレーター」でテスト入力してパース結果を確認できる（`extractHiddenTags`をそのまま呼ぶだけ）

### タグ・ハンドラーレジストリ（`HIDDEN_TAG_HANDLERS`）

タグの種類と挙動は `HIDDEN_TAG_HANDLERS` オブジェクト1箇所に一元管理されている。新しいタグを追加する場合はここにエントリを足すだけでよい。

```js
const HIDDEN_TAG_HANDLERS = {
  affection: {
    kind: 'state',
    apply(value, charKey){ /* bumpAffection(charKey, delta) */ }
  },
  flag: {
    kind: 'state',
    apply(value){ /* state.flags[value] = true */ }
  },
  face: {
    kind: 'display'   // stateを書き換えない。呼び出し側が個別に読んで使うだけ
  }
  // tagCategories登録済みキー（mood等）は resolveTagHandler() が動的に 'state' ハンドラを生成する
};
```

| `kind` | 意味 |
|---|---|
| `state` | `apply(value, charKey)` が呼ばれ、ゲームの状態（`hub.coreState`や`state.flags`）を書き換える |
| `display` | 状態には一切触れない。呼び出し側のコードが `extractHiddenTags().tags` を直接見て個別に使う（例: `face`は`aiChoice`処理側でのみ読まれる） |

未登録のキー（レジストリにも`tagCategories`にも無いキー）は無視される（無効なタグ扱い）。

### 組み込みタグ一覧

| タグ | 例 | 効果 |
|---|---|---|
| `affection` | `[[affection:+2]]` | 対象キャラ（`targetCharKey`、省略時`MAIN_CHAR`）の好感度を増減。`bumpAffection`経由でHUDも更新 |
| `flag` | `[[flag:asked_name]]` | グローバルフラグ `state.flags[value] = true` を立てる（現状は分岐条件には使われておらず、記録のみの簡易実装） |
| `mood` | `[[mood:happy]]` | `tagCategories.mood`に登録された値であれば、そのキャラの`coreState.mood`を更新（未登録値は`fallback`に丸められる） |
| `face` | `[[face:happy]]` | **状態には反映されない表示専用タグ**。`aiChoice`のセリフ候補に埋め込むと、選ばれた時だけそのキャラのスプライトを表情差分に切り替える（詳細は[10章](#10-表情差分facetグ)） |

`targetCharKey`（誰の状態に適用するか）は、通常セリフ行なら`line.char`（無ければ`MAIN_CHAR`）、`aiChoice`行なら`line.char`が使われる。

---

## 6. 状況タグカテゴリ（`tagCategories`）

`aiChoice`のセリフ選択や、CGアンロック条件の判定に使われる「状況タグ」を生成する仕組み。開発者パネルの「タグカテゴリ編集」から追加・編集できる。

```js
let tagCategories = {
  mood: {
    type: 'enum',
    values: ['normal', 'work_stress', 'tired', 'happy'],
    fallback: 'normal'
  },
  affection: {
    type: 'threshold',
    thresholds: [
      { lt: 30, label: 'low' },
      { lt: 70, label: 'mid' },
      { lt: Infinity, label: 'high' }
    ]
  }
};
```

### カテゴリの型（`type`）

| type | 説明 | 生成されるタグ例 |
|---|---|---|
| `enum` | 決められた値のリストから1つ選ぶ。リストに無い値は`fallback`に丸められる | `mood_happy`, `mood_normal` |
| `threshold` | 数値をしきい値で区切ってラベル化する。`lt`（未満）を昇順に並べ、最後の要素の`lt`は無視され「それ以上」を意味する | `affection_low`, `affection_mid`, `affection_high` |

### タグの組み立て（`buildState`）

`aiChoice`の判定やCGアンロック判定は、いずれも`buildState()`が生成するタグ配列を使う：

```js
function buildState(opts){
  const tags = ['time_' + opts.timeOfDay];
  // tagCategoriesに登録された全カテゴリを、coreStateの値から自動でタグ化
  Object.keys(tagCategories).forEach(catKey => {
    if(!(catKey in coreState)) return;
    tags.push(resolveCategoryTag(catKey, coreState[catKey]));
  });
  if(opts.extraTags) tags.push(...opts.extraTags);
  return tags;
}
```

生成されるタグの例: `["time_day", "mood_happy", "affection_mid", "rainy"]`（最後の`rainy`は`extraTags`で追加した例）

- `time_xxx` は`timeOfDay`から常に自動生成される
- `extraTags`（`aiChoice`行や`cgUnlock`行に書ける任意のタグ文字列配列）を併用すると、`tagCategories`に登録されていない一時的な条件（天候など）も追加できる

新しいカテゴリを増やすと、シナリオJSON側で `[[新カテゴリ名:値]]` のタグがそのまま使えるようになる（`resolveTagHandler`が動的に対応するため、エンジンのコードを変更する必要はない）。

---

## 7. 学習型ダイアログメモリシステム（`DialogueMemorySystem`）

`aiChoice`行のセリフ候補を、プレイヤーのフィードバック（♡/×）に基づいて重み付き学習していく仕組み。

### 仕組みの概要

1. `aiChoice`行に到達すると、`hub.decide(scene, timeOfDay, choices, extraTags)`が呼ばれる
2. 現在の状況タグ（`buildState()`の結果）と各候補の組み合わせでスコアを引き、**スコアに応じた重み付きランダム**で1つを選ぶ
3. 選ばれたセリフが表示され、開発者モード＋調教UIオンの場合は♡/×ボタンが出る
4. フィードバックすると `giveFeedback(state, choice, isGood, weight)` が呼ばれ、スコアが±1される

### キー生成とスコアリング

```js
class DialogueMemorySystem{
  static makeKey(stateCombo, choice){
    return stateCombo.slice().sort().join(',') + '|||' + choice;
  }
  getScore(stateCombo, choice){ return this.scores[makeKey(stateCombo, choice)] || 0; }
  giveFeedback(stateCombo, choice, isGood, weight){
    this.scores[key] += isGood ? weight : -weight;
  }
}
```

- スコアのキーは「**状況タグの組み合わせ（ソート済み）＋セリフ文字列そのもの**」
- つまり学習は「**この状況の時、このセリフはどれくらい好まれるか**」という粒度で行われる。シーン単位やキャラ単位の大雑把な学習ではない
- `choice`は生文字列（`[[face:happy]]`等のタグ込み文字列）がそのままキーの一部になる。**タグの有無で別のキーとして扱われる**ので、同じ意味のセリフでもタグの付け方を変えると学習データが引き継がれない点に注意

### 選択ロジック（`decideChoice`）

```js
decideChoice(currentState, availableChoices){
  const candidates = availableChoices.filter(c => getScore(currentState, c) > 0);
  if(candidates.length === 0){
    return ランダムに1つ選ぶ; // まだ学習データが無い場合
  }
  // 直近の履歴（historyLimit件）に無いものを優先しつつ、スコア比例の重み付き抽選
}
```

- スコアが0（未学習 or ちょうど相殺）の候補は、まず全てが0点なら完全ランダムに選ばれる
- 1つでもプラススコアの候補があれば、**スコアがプラスの候補の中から**、直近履歴と被らないよう調整しつつ重み付きランダムで選ばれる
- マイナスに評価された候補（スコア<0）は、他に候補が無い限り選ばれない
- `recentHistory`（直近`historyLimit`件）に入っている候補は、他に選択肢があれば避けられる（同じセリフの連発を防ぐ）

### `CharacterDialogueHub`（キャラ単位のラッパー）

```js
class CharacterDialogueHub{
  constructor(characterKey){
    this.scenes = {};                          // シーン名 → DialogueMemorySystem
    this.coreState = { mood: 'normal', affection: 20 }; // このキャラの状態（真実源）
  }
  decide(sceneName, timeOfDay, availableChoices, extraTags){
    const currentState = buildState({ timeOfDay, coreState: this.coreState, extraTags });
    return { chosen: scene.decideChoice(...), state: currentState };
  }
}
```

- キャラごとに`hubs[charKey]`として保持され、`getHub(charKey)`で取得・遅延生成される
- シーン名（`scene`プロパティ）ごとに別々の`DialogueMemorySystem`インスタンスを持つ（＝学習データはシーン単位で独立）

### フィードバックの意味（設計上の注意）

```js
// 調教フィードバックは「その状況・状態にその応答がふさわしかったか」の評価であり、
// プレイヤーとキャラの関係性（好感度）とは別のシステム。
```

♡/×フィードバックは**好感度には一切影響しない**。あくまで「その状況でそのセリフ候補が選ばれる確率」を調整するだけの、表示ロジック側の学習である。

---

## 8. CGアンロック条件（`unlockIf`）

`cgUnlock`行にたどり着いた時、CGを解禁するかどうかの条件を指定できる。

```js
{ text: "cgUnlock", cg: "cg_rooftop_smile", se: "unlock", unlockIf: "affection_mid" }
```

### 判定ロジック（`checkUnlockCondition`）

```js
function checkUnlockCondition(line){
  if(!line.unlockIf) return true; // unlockIf未指定なら常に無条件で解禁（後方互換）
  const charKey = line.char || MAIN_CHAR;
  const hub = getHub(charKey);
  const tags = buildState({ timeOfDay: line.timeOfDay || 'day', coreState: hub.coreState, extraTags: line.extraTags });
  const required = Array.isArray(line.unlockIf) ? line.unlockIf : [line.unlockIf];
  return required.every(req => tags.indexOf(req) !== -1); // AND条件
}
```

| `unlockIf`の書き方 | 意味 |
|---|---|
| 省略 | 常に無条件で解禁（その行に到達するだけで解禁される） |
| `"affection_high"` | 単一タグ条件。そのタグが現在の状況タグに含まれている場合のみ解禁 |
| `["affection_high", "mood_happy"]` | 配列で複数タグを指定するとAND条件になる（**全て**満たす必要がある） |

- 判定対象のキャラは`line.char`（省略時`MAIN_CHAR`）
- `tagCategories`に登録されている任意のカテゴリ（`affection`, `mood`, または独自に追加したカテゴリ）がそのまま条件として使える
- `line.extraTags`を併用すれば、`tagCategories`に登録されていない一時的なタグ（例: `rainy`, `first_time`）も条件に含められる
- 条件を満たさなかった場合、CGは解禁されないが**行自体はスキップされて先へ進む**（エラーにはならない）

---

## 9. 好感度システム（`affection`）

### 真実源は `hub.coreState.affection` のみ

以前は`state.affection[charKey]`にも同じ値を複製して持っていたが、二重管理になっていたため廃止された。現在は以下の1本の経路のみで管理される。

```js
function getAffection(charKey){ return getHub(charKey).coreState.affection; }

function bumpAffection(charKey, delta, mood){
  const hub = getHub(charKey);
  const current = (hub.coreState.affection === undefined) ? 20 : hub.coreState.affection;
  const next = Math.max(0, Math.min(100, current + (delta || 0)));
  hub.updateCoreState({ affection: next, mood: mood || hub.coreState.mood });
  updateStatusHud();
}
```

- 好感度の範囲は **0〜100** にクランプされる
- 初期値は各キャラ **20**（`CharacterDialogueHub`のコンストラクタ）
- 読み取りたい場合は必ず `getAffection(charKey)` を使う（`hub.coreState.affection`への直接参照でも動くが、一貫性のため関数経由を推奨）

### 好感度を変更する3つの入口

| 入口 | 記法 | 備考 |
|---|---|---|
| プレイヤー選択肢 | `{ text, goto, affection: 5, mood: "happy" }` | `renderChoices`内でクリック時に`bumpAffection(MAIN_CHAR, c.affection, c.mood)`が呼ばれる |
| 隠しタグ | `[[affection:+2]]` | `HIDDEN_TAG_HANDLERS.affection.apply`経由。`targetCharKey`（行の`char`）に対して加算 |
| コード直呼び出し | `bumpAffection(charKey, delta, mood)` | 開発者が独自ロジックを足す場合 |

### エンディング分岐との関係

```js
function pickEndingLabel(affection){
  if(affection >= 70) return 'ending_good';
  if(affection >= 30) return 'ending_normal';
  return 'ending_cool';
}
```

`{ text: "jump-affection-check", goto: null }`という特殊行に到達すると、`getAffection(MAIN_CHAR)`の値で上記の3分岐にジャンプする。この閾値（70, 30）は`tagCategories.affection.thresholds`（30, 70）と**意図的に近い値だが、コード上は別々に定義されており連動していない**点に注意（片方を変更してももう片方には反映されない）。

### HUD表示

開発者モード時のみ、画面左上に好感度バーとmoodタグが表示される。表示対象のキャラ一覧は「これまで`getHub()`が呼ばれて`hubs`に登録された（＝一度でも登場・会話した）キャラ」から自動的に決まる（`Object.keys(hubs)`）。

---

## 10. 表情差分（`face`タグ）

`aiChoice`のセリフ候補ごとに、選ばれた時だけキャラの表情（スプライト画像）を切り替える仕組み。

### 書き方

セリフ文字列の末尾に`[[face:xxx]]`を埋め込むだけで、`choices`配列自体は今まで通りの文字列配列のまま変更不要。

```js
{
  aiChoice: true, char: "akari", scene: "morning_greeting", timeOfDay: "day",
  choices: [
    "変わってる。でも――嫌いじゃないよ、そういうの。[[face:happy]]",
    "……ふうん。趣味悪くない？[[face:normal]]",
    "私も、ちょっとだけ見てみようかな。[[face:shy]]"
  ]
}
```

### 仕組み（学習ロジックへの影響なし）

- `face`タグは`HIDDEN_TAG_HANDLERS`に`kind: 'display'`として登録されているため、`applyHiddenTags`（`state`を書き換える処理）では**何もされない**
- `showLine()`の`aiChoice`処理内で、選ばれたセリフから個別に`face`タグだけを読み取り、`renderSprite`に渡す

```js
const faceTags = extractHiddenTags(result.chosen).tags.filter(t => t.key === 'face');
if(faceTags.length){
  const pos = findStagePos(line.char);
  if(pos) renderSprite(pos, line.char, false, faceTags[faceTags.length - 1].value);
}
```

- `DialogueMemorySystem`のスコアリングキーには、タグ込みの生文字列がそのまま使われる。つまり`face`タグの有無・値も学習キーの一部になるが、学習ロジック自体（`makeKey`, `decideChoice`）は一切変更されていない

### キャラ側の設定（`variants`）

```js
characters.akari = {
  name: "アカリ",
  img: "",  // デフォルト画像（faceタグが無い場合や、該当variantsが無い場合のフォールバック）
  variants: { happy: "path/to/happy.png", normal: "path/to/normal.png", shy: "path/to/shy.png" }
};
```

`renderSprite(pos, charKey, instant, face)`は、`face`引数が指定され、かつ`characters[charKey].variants[face]`が存在すればそちらを使い、無ければ通常の`img`にフォールバックする。**`variants`未設定のキャラでも今まで通り動作する**（後方互換）。

---

## 11. エンドレスモード

開発者向け・クリック/タップ送り駆動で、`aiChoice`のような学習型セリフ生成を無限に繰り返し、調教（フィードバック学習）の効果を検証し続けるためのモード。開発者パネルからのみ開始できる。

### サブモード

| サブモード | 内部変数 | 挙動 |
|---|---|---|
| 固定 | `endlessSubMode = 'fixed'` | 同一シーン（`'endless_fixed'`）・同一状況のまま、送るたびに再抽選され続ける。同一状況への重み付けの効果をはっきり体感できる |
| 周回 | `endlessSubMode = 'cycle'` | そのキャラに登録済みのシーン名を順番に巡回しながら進行する。通しプレイに近い体験 |

### 起動フロー（`startEndlessMode`）

```js
function startEndlessMode(){
  resetStage();
  endlessCharKey = (選択されたキャラ) || MAIN_CHAR;
  if(endlessSubMode === 'cycle'){
    endlessScenePool = Object.keys(getHub(endlessCharKey).scenes); // 既存シーン一覧を巡回
  } else {
    endlessScenePool = ['endless_fixed'];
    getHub(endlessCharKey).registerScene('endless_fixed', 4);
  }
  endlessActive = true;
  endlessEmitLine();
}
```

- セリフ候補には`ENDLESS_LINE_POOL`（コード内に固定で5つ定義されている汎用セリフ）が使われる。シナリオJSON側の`choices`は参照されない
- `endlessAdvance()`：クリック/タップやAUTOで送るたびに呼ばれる。周回モードなら`endlessSceneIdx`を進めてから、固定モードならそのまま`endlessEmitLine()`を再度呼ぶ
- `advance()`関数内で`gameMode === 'endless'`かどうかで通常のシナリオ進行ロジックとは別経路に分岐している

### 注意

- エンドレスモードは`gameMode`というグローバル変数で管理されており、シナリオモードとは**別のコードパス**（`showLine()` vs `endlessEmitLine()`）で動く。片方だけ機能追加すると、もう片方には反映されない点に注意（例: `face`タグ対応やCGアンロックは、エンドレスモード側の`endlessEmitLine()`には組み込まれていない）

---

## 12. アセット管理（キャラ/背景/BGM/SE/CG）

開発者パネルの各セクションから、キー名と実体（画像/音声のURLまたはパス）を編集できる。全て`persistAssets()`で`ASSETS_KEY`にまとめて保存される。

### デフォルト定義

```js
let backgrounds = { rooftop: "", classroom: "", ending_room: "" };
let characters = {
  mina: { name: "ミナ", img: "" },
  akari: { name: "アカリ", img: "", variants: { happy: "", normal: "", shy: "" } },
  narrator: { name: "", img: "" }
};
let cgGallery = {
  cg_rooftop_smile: { label: "屋上の微笑み", img: "", unlocked: false },
  cg_classroom_seat: { label: "隣の席", img: "", unlocked: false },
  cg_ending_good: { label: "エンディングCG（Good）", img: "", unlocked: false }
};
let bgmList = { rooftop_theme: "", classroom_theme: "", ending_theme: "" };
let seList = { select: "", type: "", unlock: "", feedback_good: "", feedback_bad: "" };
```

- `img`/`bg`/`bgm`/`se`が空文字の場合、キャラは名前だけのフォールバック表示（`.sprite-fallback`）になり、音声は再生されない（エラーにはならず静かにスキップされる）
- `cgGallery`の`unlocked`は**アセット定義とは別に**`GALLERY_KEY`で管理されている。アセットを読み込む際（`loadAssets`）は、既存の`unlocked`状態を上書きしないよう配慮されている

### 追加・削除

開発者パネルの各セクションにある「＋追加」ボタンで`prompt()`によりキー名を入力し、新規エントリを追加できる。削除は各行の削除ボタンから。

### MAIN_CHARの保護

```js
if(!characters[MAIN_CHAR]){ characters[MAIN_CHAR] = { name: 'アカリ', img: '' }; }
```

`MAIN_CHAR`（`'akari'`固定）が誤って削除された場合、`loadAssets()`実行時に自動的に復元される。

---

## 13. セーブ/ロード

- セーブスロット数は可変（開発者パネルで1〜30の範囲に設定可能、デフォルト6）
- 1スロットのデータ構造：

```js
{
  label, index, bgKey, onStage, backlog, flags,  // 進行状態
  preview,                                        // 直前に表示していた行のプレビュー文字列
  thumb,                                          // 背景画像URL（サムネイル用）
  time                                            // セーブ日時（日本語ロケール表記）
}
```

- **好感度・mood・学習データはセーブスロットには含まれない。** これらは`hubs`全体としてセーブ時に`persistAllHubs()`が別途、常に最新の`MEMORY_KEY`へ保存する仕組みになっている。つまり「どのスロットをロードしても、好感度や学習の進み具合は共有される」設計（意図的な仕様。プレイスルーの進行位置とキャラの記憶は別物として扱われている）
- ロード時、`state`はスロットのデータで復元されるが、好感度は起動時に`loadAllHubs()`で既に読み込まれている`hubs`側がそのまま使われる

---

## 14. 開発者パネル機能一覧

タイトル画面の「開発者パネル」ボタン（`?dev=1`時のみ表示）から開く。タブ構成：

| タブ/セクション | 機能 |
|---|---|
| モード切替 | シナリオモード / エンドレスモードの選択、エンドレスのサブモード（固定/周回）・対象キャラ選択、開始/停止ボタン |
| シナリオ | シナリオJSONをテキストエリアで直接編集。「反映」（JSON検証してscriptに反映）、「整形」（インデント整形）、「ラベル追加」「選択肢テンプレ挿入」ボタン |
| アセット | キャラ・背景・BGM・SE・CGの一覧編集（画像パス・ラベル名の変更、追加、削除） |
| タグカテゴリ | `tagCategories`の追加・編集・削除。新規カテゴリはデフォルトで`enum`型（`values: ['value1','value2']`）として追加される |
| 隠しタグ シミュレーター | 任意の文字列を入力し、`extractHiddenTags()`のパース結果（除去後テキストと検出タグ一覧）を確認できる |
| セーブ設定 | セーブスロット数の変更、「全セーブ/学習データを消去」ボタン |

### シナリオエディタの反映フロー

1. テキストエリアの内容を編集
2. 「反映」ボタン → `JSON.parse`で検証 → 成功すれば`script`に反映され`persistScenario()`で永続化。失敗すれば`scenarioStatus`にエラーメッセージ表示
3. タイトルへ戻って「はじめから」or 該当ラベルへ移動して動作確認

---

## 15. 主要関数リファレンス（開発者向け）

進行ロジックの中核となる関数を、呼び出し関係が分かるように列挙する。

| 関数 | 役割 |
|---|---|
| `showLine()` | 現在の`state.label`/`state.index`が指す行を表示する中核関数。`jump`/`jump-affection-check`/`cgUnlock`/`end`/`aiChoice`/通常セリフを分岐処理する |
| `renderLineText(rawText, displayName, isAiLine, targetCharKey)` | 隠しタグの除去・適用（`applyHiddenTags`）を行った上で、タイプライター表示を開始する共通関数。通常セリフとAI駆動セリフの両方から呼ばれる |
| `applyHiddenTags(tags, targetCharKey)` | `resolveTagHandler`経由でレジストリを引き、`kind==='state'`のタグだけ副作用を適用する薄いディスパッチャ |
| `resolveTagHandler(key)` | `HIDDEN_TAG_HANDLERS`または動的生成される`tagCategories`ハンドラを返す |
| `applyStageDirections(line)` | `bg`/`bgm`/`se`/`show`/`hide`プロパティを演出として適用 |
| `advance()` | クリック/タップ/スペースキー等で呼ばれる「送り」処理。タイプ中なら即表示完了、選択肢待ちなら何もしない、フィードバック待ちなら何もしない、それ以外は次の行へ |
| `renderChoices(choices)` | プレイヤー選択肢（オブジェクト配列）をボタンとして描画し、クリック時に`bumpAffection`と`goto`遷移を行う |
| `bumpAffection(charKey, delta, mood)` | 好感度を増減し`hub.coreState`を更新、HUDを再描画 |
| `getAffection(charKey)` | 好感度の読み取り専用アクセサ |
| `checkUnlockCondition(line)` | CGアンロック条件（`unlockIf`）の判定 |
| `unlockCg(key)` | CGギャラリーの該当キーを解禁済みにする（条件判定はここでは行わない） |
| `getHub(charKey)` | `hubs[charKey]`を遅延生成しつつ返す |
| `buildState(opts)` | `timeOfDay` + `tagCategories`登録済みカテゴリ + `extraTags`から、状況タグの配列を組み立てる |
| `submitFeedback(isGood)` | 保留中の`aiChoice`フィードバックを`DialogueMemorySystem`に記録する |
| `startGame()` / `startEndlessMode()` | それぞれシナリオモード / エンドレスモードの開始処理 |
| `endlessEmitLine()` / `endlessAdvance()` | エンドレスモード専用の行生成・送り処理（`showLine()`とは別経路） |

---

## 16. 既知の設計上の注意点・制約

改造・拡張する際に踏み抜きやすいポイントをまとめる。

1. **エンディング閾値の二重定義**：`pickEndingLabel()`内の閾値（70, 30）と`tagCategories.affection.thresholds`（同じく70, 30）は値が近いが**コード上は連動していない独立した定義**。片方だけ変更すると挙動が食い違う。
2. **シナリオモードとエンドレスモードのコードパスが分離している**：`showLine()`系（シナリオ）と`endlessEmitLine()`系（エンドレス）は別々の実装。片方に機能を足しても他方には自動反映されない（例: 現状、`face`タグやCGアンロック条件はエンドレスモード側では使われていない）。
3. **`choices`プロパティの型が行の種類で変わる**：プレイヤー選択肢はオブジェクト配列、`aiChoice`は文字列配列。同名プロパティで型が違うため、シナリオJSON編集時に混同しやすい。
4. **`flag`タグは分岐条件として使われていない**：`[[flag:xxx]]`で`state.flags`に値は立つが、現状これを参照して条件分岐する仕組みはエンジン内に存在しない（記録のみの簡易実装）。分岐条件として使いたい場合は別途実装が必要。
5. **学習キーはタグ込み文字列に依存する**：`face`タグ等をセリフに追加すると、そのセリフの学習キー（`DialogueMemorySystem`のスコア）は別物として扱われる。既存の学習データを引き継ぎたい場合は、タグの追加前後でセリフ文字列を変えない設計が必要。
6. **`localStorage`前提の永続化**：プライベートブラウジングモードや、ブラウザの保存容量制限次第ではセーブ・学習データが保持されない可能性がある。
7. **CGアンロックはあくまで「行への到達」がトリガー**：`unlockIf`はその行に到達した瞬間の状態のみを見る。到達後に状態が変化しても再判定はされない。
