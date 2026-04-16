# コードレビュー

## サマリー

| 重要度 | 件数 |
|---|---|
| 🔴 高（セキュリティ） | 3 |
| 🟡 中（バグ・UX） | 4 |
| 🟢 低（コード品質） | 4 |

**対応状況:**
- [1] GitHub Pages の制約上、クライアントサイド認証は許容済み
- [2] ✅ 修正済み（`data-name` 属性方式に変更）
- [3] GitHub Pages / GAS の構成上、バックエンド認証は構造的に困難なため許容済み

---

## 🔴 高（セキュリティ）

### [1] 管理者パスワードがクライアントサイドに平文で存在する

**対象:** `admin.html`, `view.html`

```javascript
const ADMIN_PASSWORD = "admin123";
```

ブラウザの開発者ツールでソースを見れば誰でもパスワードを知ることができる。
`admin123` という弱いパスワードであることも問題。

**現実的な影響:** イベントの無断削除・データ閲覧が可能。

**改善案:** Google フォームの回答制限、Firebase Authentication、または GAS 側でのセッション管理など、サーバーサイドで認証を実装する。短期的には少なくともパスワードを強化する。

---

### [2] `deleteEvent` の onclick 属性にイベント名を直接埋め込んでいる ✅ 修正済み

**対象:** `admin.html`

**問題:** イベント名にシングルクォート（`'`）が含まれると JavaScript が壊れる。
悪意あるシート名（例: `test', alert('XSS`）を作れる環境では XSS になりうる。

**対応内容:** `data-name` 属性でイベント名を保持し、JS 側で `btn.dataset.name` から取得する方式に変更した。

```javascript
// 修正後
`<button onclick="deleteEvent(this)" data-name="${name.replace(/"/g, '&quot;')}" ...>`

async function deleteEvent(btn) {
    const name = btn.dataset.name;
    // ...
}
```

---

### [3] GAS エンドポイントが公開状態でアクセス制御なし

**対象:** 全ファイル

GAS の Web App URL が HTML にハードコードされており、URL さえ知っていれば誰でも `createEvent` / `deleteEvent` / `updateData` を直接呼び出せる。フロントの認証をバイパスしてもバックエンド側でリジェクトされない。

**改善案:** GAS 側で認証トークンの検証や IP 制限、または「Google ログイン必須」モードでの公開設定を検討する。

---

## 🟡 中（バグ・UX）

### [4] データを読み込んでいない状態でも保存ボタンが押せる

**対象:** `view.html`

ページを開いて「読込」を押さずに「【結果】を作成・保存」を押すと、`currentRows = []` のまま保存され、**結果シートが空データで上書きされる**。

```javascript
let currentRows = [];  // 初期値が空配列

async function saveChanges() {
    // currentRows が空かどうかをチェックしていない
    await fetch(GAS_URL, { ..., body: JSON.stringify({ ..., rows: currentRows }) });
}
```

**改善案:** 保存前にガードを追加する。

```javascript
async function saveChanges() {
    if(currentRows.length === 0) {
        alert("先にデータを読み込んでください");
        return;
    }
    // ...
}
```

---

### [5] `view.html` の `【結果】` シートも読込対象に含まれる

**対象:** `view.html`

`index.html` は `getSheets` の結果から `【結果】` を含むシートを除外しているが、`view.html` は除外していない。
「読込」で `【結果】` シートを選択して保存すると、結果シートを結果シートで上書きするという意図しない操作が起きる。

**改善案:** シートセレクトの表示時に除外するか、視覚的に区別する。

```javascript
sheets.forEach(s => {
    const opt = document.createElement('option');
    opt.value = s;
    opt.textContent = s;
    if(s.includes('【結果】')) opt.style.color = '#9ca3af'; // グレーアウト
    select.appendChild(opt);
});
```

---

### [6] イベント作成後にテキスト入力がクリアされない

**対象:** `admin.html`

`createNewEvent()` 成功後、イベント名の入力フィールドが空にならない。
同じ名前で誤って再作成するリスクがある。

**改善案:**

```javascript
// createNewEvent() の finally ブロックに追加
document.getElementById('eventName').value = '';
```

---

### [7] `copyUrl()` で使われている `execCommand('copy')` が非推奨

**対象:** `admin.html`

```javascript
document.execCommand('copy');  // 非推奨 API
```

モダンブラウザでは Clipboard API の使用が推奨されている。

**改善案:**

```javascript
function copyUrl() {
    const url = document.getElementById('inviteUrl').value;
    navigator.clipboard.writeText(url).then(() => {
        alert("URLをコピーしました！LINEなどで共有してください。");
    });
}
```

---

## 🟢 低（コード品質）

### [8] GAS_URL・ADMIN_PASSWORD・spin() が全ファイルに重複している

3つのファイルそれぞれに同じ定数と関数が定義されている。
URL を変更する際に3箇所を修正する必要があり、修正漏れが起きやすい。

```javascript
// 3ファイル全てに同じ記述
const GAS_URL = "https://script.google.com/.../exec";
const ADMIN_PASSWORD = "admin123";
const spin = (cls = ...) => `<svg ...>`;
```

**改善案（短期）:** 共通ファイル `config.js` を作成して読み込む。

```html
<script src="config.js"></script>
```

```javascript
// config.js
const GAS_URL = "https://...";
const ADMIN_PASSWORD = "admin123";
const spin = (cls = 'h-4 w-4') => `<svg ...>`;
```

---

### [9] `view.html` の `window.onload` でパスワード不一致時に `return` がなかった（修正済み）

元のコードでは `location.href = "index.html"` の後に `return` がなく、リダイレクト中も後続の fetch が実行されていた。
現在のコードでは修正済み。

```javascript
// 修正後（現在のコード）
if(pass !== ADMIN_PASSWORD) { location.href="index.html"; return; }
```

---

### [10] `spin()` のデフォルトサイズがファイルによって異なる

`index.html` では `h-5 w-5`、`admin.html` と `view.html` では `h-4 w-4` がデフォルト。
統一するか、サイズを呼び出し側で常に明示するとよい。

---

### [11] `admin.html` の `loadEventList()` が `【結果】` シートも一覧に表示する

削除一覧に `【結果】xxx` が並ぶと管理者が誤って削除するリスクがある。
視覚的に区別するか、別セクションに分けることを検討する。

---

## 総評

イベント運営用ツールとして実用的な機能が揃っており、ローディング状態の実装も丁寧にされている。
主な懸念はセキュリティで、パスワード認証がクライアントサイドのみであることが最大のリスク。
利用シーン（内輪のスポーツチーム等）であれば許容範囲ともいえるが、意識しておくべき点として挙げた。

バグとしては「空データで保存できる」問題（[4]）が最もダメージが大きく、早めに対処を推奨する。
