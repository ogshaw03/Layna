# Layna プラグイン制度 — 今後の方針メモ

Layna（単一HTML・ビルド無し・Vanilla JS）にプラグイン制度を導入する際の方針。
段階的に導入でき、まず **Phase 1 + 2** から着手するのが費用対効果が高い。

## 目的
- 通知連携（Slack等）、メンション連携、独自集計タブ、独自バッジなどを
  **本体コードを分岐させずに**追加できるようにする。
- プロジェクトごと／導入先ごとに機能を出し分けられるようにする。

---

## Phase 1: JSフックAPI（最優先・低コスト）
グローバルに `window.Layna` を用意し、本体の要所で**フック（イベント）**を発火する。

```js
window.Layna = {
  version: '3',
  plugins: [],
  register(p){ this.plugins.push(p); p.setup?.(this.api) },
  emit(evt, ctx){ for(const p of this.plugins) p.hooks?.[evt]?.(ctx) },
  api: { getNode, projectMembers, memberById, persist, toast, /* 公開する関数だけ */ }
};
```

プラグイン側:
```js
Layna.register({
  name: 'slack-notify',
  setup(api){ /* 初期化 */ },
  hooks: {
    'note:submit'({node, note}){ /* 送信時にSlackへ */ }
  }
});
```

### 発火ポイント候補（本体に `Layna.emit(...)` を1行挿すだけ）
- `version:upload`   … `uploadVersion` 完了後
- `note:submit`      … レビュー送信 record()／Reel 送信後
- `status:change`    … ステータス切替時（buildStatusSelect）
- `shot:assign`      … 作業/チェック担当の変更時（shotAssignPickers）
- `message:post`     … メッセージ（shotCommentInput）送信時
- `mention`          … メンション検出時（将来）
- `tab:render`       … 各タブ描画時
- `project:open` / `project:save`

---

## Phase 2: UI拡張スロット
描画関数に「拡張スロット」を設け、プラグインが要素を差し込めるようにする。

- `Layna.registerTab(id, label, renderFn)`        → 進捗/ショットの隣に独自タブ
- `Layna.registerTileBadge(fn)`                   → ショットタイルに独自バッジ
- `Layna.registerTopbarAction(fn)`                → タスクページ右上に独自ボタン
- `Layna.registerNoteDecorator(fn)`               → FBログ項目に独自表示

実装上は、既存の描画箇所（buildProjectTabsHead / renderProjShots のタイル /
renderReviewBody の topbar など）でスロット配列を走査して差し込む。

---

## Phase 3: 外部ファイル読み込み（本格運用・後回しでよい）
プラグインを**共有フォルダ内の別JSファイル**として配置し、起動時に読み込む。

- File System Access API で既にプロジェクトフォルダを掴んでいるので、
  `media/` と同じ要領で `plugins/*.js` を列挙 → `import()` / `<script>` 注入で登録。
- プロジェクトごとに配布・追加が可能になる。

---

## 設計上の留意点
- **サンドボックス**: `file://`／githack 上では外部プラグインが本体と同一権限で動く。
  信頼できるコードのみ許可。厳密に隔離するなら `iframe` + `postMessage` でAPI化
  （実装コストは上がるので本格運用時に検討）。
- **API安定性**: 公開する `Layna.api` は関数を絞り、「契約」として固定する。
  内部関数を直接触らせない。本体改修でプラグインが壊れないようにする。
- **データ互換**: プラグインが書き込むフィールドは名前空間にまとめる
  （例 `node.ext.<pluginId>`）。`layna.project.json` を汚さない。
- **バージョニング**: `Layna.version` と各プラグインの `requires` を突き合わせ、
  非互換時は警告して読み込みスキップ。

---

## おすすめの着手順
1. `window.Layna`（register / emit / api）の骨組みを追加
2. 主要フックの `emit` を本体要所に挿入（Phase 1）
3. タブ／バッジ／topbarアクションのスロット（Phase 2）
4. 運用が固まったら外部ファイル読み込み（Phase 3）
