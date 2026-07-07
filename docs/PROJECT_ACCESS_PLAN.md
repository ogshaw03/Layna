# プロジェクトのアクセス制御 — 現状と今後の方針メモ

## 現状（実装済み）
### 2層のアクセス制御
1. **ツールに入れるか（アクセス権）**
   - `ogshaw03.github.io` 上では Googleログイン必須（`access.json` / Firebase の `authRequired`）。
   - 許可リスト（operator / admin / member / allowedDomains）で判定（`roleFor`）。
   - このロールは「入れる／入れない」の判定のみで、ツール内機能の制限には未使用。

2. **各プロジェクトを開けるか（パスワード保護）** ← 今回追加
   - プロジェクト設定（`openRename`・ルートのみ）で**パスワードを設定/変更/解除**。
   - 保存先: `root.passHash`（SHA-256 16進）+ `root.passSalt`（`layna.project.json`）。**平文は保存しない。**
   - 未解錠でプロジェクトを開くと解錠画面（`renderProjectLock`）を表示し配下すべてを保護。
   - 解錠状態はタブ内のみ（`sessionStorage`／`state._unlocked`）。リロードや別タブでは再入力。
   - ホームのタイルに「🔒 保護」バッジ表示。

### 重要な限界（UIレベルの制限）
- データ本体（`layna.project.json` と `media/`）は共有フォルダに**平文で存在**する。
- パスワードは**画面上のアクセスを制限するだけ**で、フォルダに直接アクセスできる人には無力。
- 暗号学的なデータ保護ではない（あくまで抑止・区分けの手段）。

---

## 今後：アクセス権付与式（検討中）
プロジェクトごとに「誰が開けるか」を**付与式**で管理する案。パスワードと併用/置換できる。

### 案A: メンバー名簿ベース（ローカル完結・低コスト）
- `root.members` に既にある名簿へ「アクセス可否」フラグ or ロール（viewer/editor/owner）を追加。
- ログインユーザーの `authUser.email` が名簿に含まれるか＆権限で開閉を判定。
- 長所: 追加インフラ不要。短所: 名簿自体が共有フォルダにあるため改ざん耐性は低い。

### 案B: Firebase 付与式（厳密・推奨の最終形）
- Firestore に `projects/<projectId>/access`（uid/email → role）を持たせる。
- プロジェクトを開く前に Firestore を参照して認可（サーバー側ルールで強制）。
- メディアの実体保護まで求めるなら、共有フォルダ配布をやめ **Firebase Storage + 署名付きURL**へ。
  - ここまでやると「フォルダ直アクセス」の抜け道が塞がり、真のアクセス制御になる。
- アクセス権の付与UIは既存の **access-console.html** を拡張して一元管理するのが自然。

### 移行の考え方
- まずは今回のパスワード保護で「誰でも入れる」不安を緩和（済）。
- 次段階で案A（名簿＋ロール）を入れ、UIレベルの区分けを整える。
- 厳密性が要るタイミングで案B（Firebase 認可＋Storage）へ。パスワードは補助/廃止。

---

## 実装メモ（現状コードの該当箇所）
- パスワード設定UI: `openRename` のルートブロック（`pwNew`/`pwConf`/`pwRemove`）。
- ハッシュ/解錠: `hashPass` / `setProjectPassword` / `verifyPass` / `unlockedSet` / `markUnlocked` / `projectLocked`。
- 解錠画面: `renderProjectLock`（`renderBodyMain` の冒頭でゲート）。
- ホームのバッジ: `renderHome`（`r.passHash` 判定）。
