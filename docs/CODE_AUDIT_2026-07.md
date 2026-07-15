# laycat.html コード監査（2026-07 / 検証済み）

対象コミット: `9d12e25`（APP_VERSION `2026.07.12.93`） / 対象: `laycat.html`（6667行・単一HTML）

本監査は、以前テキストで挙がっていた指摘 #1〜#14 を**実コードに対して1件ずつ裏取り**し、
**確認できたものだけ**を「確認済み」として掲載する。行番号は上記コミット時点。
各修正は未適用（本ドキュメントは所見のみ。実装は別途）。

---

## A. 確認済み（実在・コード内で対処可能）

### A1. 【High/セキュリティ】XSS（HTMLインジェクション）— `laycat.html:2620-2621`
```
if(firstShot)msg='この設定では 「'+firstShot.name+'」＝<b>ショット</b>'+(fk?'／「'+fk.name+'」＝<b>工程</b>':'（工程なし）')+' として扱われます。';
hint.innerHTML=msg+(shotsAuto(cur)?'　※現在は<b>自動判定</b>です…':'');
```
- ノード名（`firstShot.name` / `fk.name`）は**ローカルフォルダ名などのユーザー入力由来**。エスケープせず `innerHTML` に連結しているため、名前に `<img src=x onerror=...>` を含めるとスクリプトが実行される。
- 影響範囲: コード全体で「名前→`innerHTML` 連結」はこの1箇所のみ（他の `innerHTML` は `ICONS`/`LOGIN_LOGO` 等の静的定数で安全）。
- 修正方針: `<b>` を要素で組み立て、名前は `textContent`/テキストノードで挿入（`innerHTML` 連結をやめる）。

### A2. 【High/整合性】進捗の過大表示（部分完了が「完了」に見える）— `laycat.html:1165-1168`
```
const sts=ch.map(nodeStatus).filter(s=>s!=='empty');
if(!sts.length)return 'empty';
if(sts.every(s=>s===sts[0]))return sts[0];
return projStatuses(node)[0].id; /* 混在は先頭ステータス扱い */
```
- 子集計で `empty`（未着手）を**除外してから**「全一致」を見るため、例えば3工程中1工程だけ `approved`・残り2工程が未着手のショットが **`approved`（完了）** と表示される。
- 修正方針: 未着手/未完了の子が残る間は「完了」を返さない（全子完了時のみ完了）。

### A3. 【Med-High/堅牢性】genThumb がタイムアウト無しでハング — `laycat.html:4349`
```
vid.onseeked=()=>{...done(...)};
vid.onerror=()=>done(null)
// ← タイムアウトが無い
```
- 動画サムネ生成が `onseeked`/`onerror` 待ちのみ。破損・未対応コーデックで**どちらも発火しない**と `resolve` が呼ばれず Promise が永久ペンディング → 呼び出し側（アップロード）が固まる。
- 修正方針: `setTimeout(()=>done(null), 15000)` 等のフォールバックを追加。

### A4. 【Med/メモリ】getURL の Blob URL キャッシュが単調増加 — `laycat.html:1045`
```
const url=URL.createObjectURL(blob);this.urlCache.set(ref,url);return url
```
- `urlCache` に Blob URL を無制限に保持。自発的な `revokeObjectURL` は無く、破棄は `delMedia`（削除時, 1044行）のみ。長時間セッションで多数の動画/サムネを閲覧するとメモリが単調増加。
- 修正方針: LRU 上限（例: 250）を設け、あふれた分を `revokeObjectURL`＋`delete`。

### A5. 【Low-Med/堅牢性】比較再生B のシークにウォッチドッグが無い — `laycat.html:4699-4712`
```
// A側(4627): seekGuard タイムアウトで seekBusy を必ず復帰
media.addEventListener('seeked',()=>{clearTimeout(seekGuard);seekBusy=false;...});
// B側(4712): 'seeked' 依存のみ・タイムアウト無し
if(bSeekBusy)return;                       // 4699
bVid.addEventListener('seeked',()=>{bSeekBusy=false}); // 4712
```
- A側は `seekGuard` タイムアウトで `seekBusy` を復帰させるが、B側 `bSeekBusy` は `seeked` イベント依存のみ。イベントを取りこぼすと `bSeekBusy` が **true 固定**になり（4699で早期 return）、比較動画Bの追従が停止する。
- 修正方針: B側にも A側同等のタイムアウトガードを追加。

### A6. 【Low/堅牢性】比較再生B のロードが getURL（リトライ/失敗通知なし）— `laycat.html:4719`
```
storage.getURL(bv.file).then(u=>{if(u&&bVid){bVid.src=u;...}});
```
- 他所（`downloadVersion` 3331、メイン動画 5056）は `getURLRetry`＋失敗トーストなのに、比較Bのロードは素の `getURL`。フォルダ同期待ち等の一時失敗時に**無言で表示されない**。
- 修正方針: `getURLRetry` に変更し、失敗時にトースト通知。

---

## B. 以前の指摘のうち、実コードで再現しない／該当なし

- **isAwaitingCheck の「並べ替え誤判定」→ 再現せず**（`laycat.html:1185-1189`）。実コードは
  「先頭ステータスID との単純一致」であり、並べ替えロジックは無い。完了系は別途 `isDoneStatus` が扱う。
- **招待の「有効期限チェック追加」→ 該当なし（moot）**。招待データに `expires` フィールドが存在しない
  （`redeemInvite` 6653-6662 は `active===false` のみ判定）。
  ※ 本質的な防御はクライアントではなく **Firestore セキュリティルール**（`laynaAccess/invited` への
    自メールキー以外の書込み禁止、招待トークンの列挙禁止 等）。これは**このリポジトリのコード外**であり、
    Firebase コンソールでの対応が必要。`access-console.html` にもルール定義テキストは無い。
- **ショットタブ/進捗タブの「代表工程の不一致」→ 欠陥として確認できず**。進捗タブ
  （`renderProjProgress` 2600-）は単一 repOf を使わず `nodeStatus`／円グラフで集計する設計で、
  ショット一覧の `repOf`（2472）とは役割が異なる。明確なバグではない。

---

## C. 軽微（確認済み・優先度低）

- **latestVideoTimeUnder（`laycat.html:5189`）**: `uploadedAt` が欠落した動画は最新判定に寄与せず、
  そのような動画しか持たないノードは「動画なし」相当に見える端ケース。実害は小さい。

---

## 参考: 監査の進め方（次セッションで踏襲）

1. `git rev-parse HEAD` と `grep APP_VERSION laycat.html` で現状を実データ確認。
2. 各指摘は**アンカー文字列を `grep -n` で実在確認 → 現物を読む → 判断**の順で1件ずつ裏取り。
   巨大な base64 行があるため `Read` が重い場合は `awk 'NR>=A&&NR<=B{print NR": "substr($0,1,240)}'` で読む。
3. 修正を入れる場合は `laycat.html` を1箇所ずつ Edit → 即 `grep -c` で反映確認 → 構文チェック。
   コードを変更したら `APP_VERSION` を上げる（本ドキュメントのみの変更では上げない）。
4. 既存の所見は `docs/KNOWN_ISSUES.md` も参照。
