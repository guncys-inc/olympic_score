# 前半・後半のナイン選択 実装計画

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** ラウンドごとに回る2つの9ホールとその順序を指定できるようにし、コース側のパー情報からニアピン・ドラコンを自動で埋める。

**Architecture:** ラウンドの `cfg` に `holes`（ナイン名の順序対）と `courseKey` を足す。`nph`/`dch` は従来どおり位置1〜18のままなので、保存・同期・集計は無改造。`holes` が無いラウンドは旧形式として従来表示にフォールバックする。

**Tech Stack:** 単一ファイルの静的HTML（`index.html` 2313行）/ Firebase Realtime Database (compat SDK) / ビルド無し / GitHub Pages 配信

**Spec:** `docs/superpowers/specs/2026-08-28-nine-selection-design.md`

## Global Constraints

- `index.html` は単一ファイル。モジュール分割しない（GitHub Pages の配信方法に影響するため）。
- ビルド工程もテストランナーも無い。**動作確認は目視**。
- 新しい外部依存を足さない。Firebase compat SDK と素の DOM のみ。
- `courses` / `courseAliases` は**読み取り専用**。セキュリティルールで書き込みは拒否される。
- 既存30ラウンドのデータ移行は行わない。`cfg.holes` を持たないラウンドは従来どおり動くこと。
- `nph` / `dch` の保存形式（位置1〜18のマップ）を変えない。
- 既存の関数名・グローバル `D` の構造・`save()` / `render()` / `renderWithScroll()` の呼び出し規約を踏襲する。
- コミットメッセージは英語（Conventional Commits）。

## ファイル構成

`index.html` のみを変更する。追加する関数はすべて既存のスクリプトブロック内に置く。

```
index.html
  holeLabel(pos, holes)        新規。位置→表示ラベル
  ninesToPars(nines, holes)    新規。ナイン2つ→18ホールのパー配列
  parsToNphDch(pars)           新規。パー配列→{nph, dch}
  loadCourseNines(courseName)  新規。courseAliases→courses を引く
  applyNineSelection()         新規。選択を D.nph/D.dch に反映
  rSet()                       変更。ナイン選択UIを追加
  rInput()                     変更。ホール番号をラベルに
  rList()                      変更。行見出しと集計見出しをラベルに
```

---

### Task 1: ラベルとパー変換の純関数

**Files:**
- Modify: `index.html`（`function raw(` の直前に3関数を追加）

**Interfaces:**
- Consumes: なし
- Produces:
  - `holeLabel(pos, holes) -> string`
  - `ninesToPars(nines, holes) -> number[]`（18要素）または `null`
  - `parsToNphDch(pars) -> {nph: {}, dch: {}}`

`holes` は `["OUT","IN"]` のようなナイン名の順序対、または `null`/`undefined`（旧形式）。
`nines` は `courses/<キー>/nines` の値（`{ "OUT": [4,3,...], ... }`）。

**ラベルの規則**（仕様書より）:
- `holes` なし → `"1"` ... `"18"`
- ナイン名が `OUT` または `IN`（大小無視）→ その9ホールは `OUT 1`..`OUT 9` / `IN 10`..`IN 18`
- それ以外のナイン名 → 各ナイン `1`..`9`
- 判定はナイン名単位。`["IN","OUT"]` なら前半が `IN 10`..`IN 18`、後半が `OUT 1`..`OUT 9`
- `浅間OUT` は `OUT` に該当しない（完全一致で判定する）

- [ ] **Step 1: 確認用のテストページを書く**

`index.html` は自動テストできないため、純関数だけを検証する使い捨てページを作る。

```bash
cat > /tmp/nine-test.html <<'HTML'
<!doctype html><meta charset="utf-8"><pre id="o"></pre><script>
HTML
```

`index.html` から3関数をコピーして貼り、以下の検証コードを続ける。

```javascript
const out=[];
function eq(name,a,b){const ok=JSON.stringify(a)===JSON.stringify(b);
  out.push((ok?"PASS ":"FAIL ")+name+(ok?"":"\n  got "+JSON.stringify(a)+"\n  want "+JSON.stringify(b)));}

eq("旧形式は通し番号", [1,9,10,18].map(p=>holeLabel(p,null)), ["1","9","10","18"]);
eq("OUT/IN", [1,9,10,18].map(p=>holeLabel(p,["OUT","IN"])), ["OUT 1","OUT 9","IN 10","IN 18"]);
eq("IN/OUT（インスタート）", [1,9,10,18].map(p=>holeLabel(p,["IN","OUT"])), ["IN 10","IN 18","OUT 1","OUT 9"]);
eq("東西は各1-9", [1,9,10,18].map(p=>holeLabel(p,["東コース","西コース"])), ["東コース 1","東コース 9","西コース 1","西コース 9"]);
eq("浅間OUTはOUT扱いしない", [1,10].map(p=>holeLabel(p,["浅間OUT","白樺IN"])), ["浅間OUT 1","白樺IN 1"]);
eq("小文字のoutも拾う", [1].map(p=>holeLabel(p,["out","in"])), ["out 1"]);

const NINES={"OUT":[4,3,4,5,4,4,3,4,5],"IN":[5,3,4,4,4,5,4,3,4],"東":[4,4,4,3,5,4,3,4,5]};
eq("連結", ninesToPars(NINES,["OUT","IN"]), [4,3,4,5,4,4,3,4,5,5,3,4,4,4,5,4,3,4]);
eq("順序が効く", ninesToPars(NINES,["IN","OUT"]).slice(0,3), [5,3,4]);
eq("未知のナインはnull", ninesToPars(NINES,["OUT","無い"]), null);
eq("holesがnullならnull", ninesToPars(NINES,null), null);

const r=parsToNphDch([4,3,4,5,4,4,3,4,5,5,3,4,4,4,5,4,3,4]);
eq("パー3がnph", Object.keys(r.nph).map(Number).sort((a,b)=>a-b), [2,7,11,17]);
eq("パー5以上がdch", Object.keys(r.dch).map(Number).sort((a,b)=>a-b), [4,9,10,15]);
const r6=parsToNphDch([6,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4]);
eq("パー6もdch", Object.keys(r6.dch), ["1"]);
document.getElementById("o").textContent=out.join("\n");
</script>
```

- [ ] **Step 2: ブラウザで開き、全て FAIL することを確認**

`file:///tmp/nine-test.html` を開く。3関数がまだ無いので参照エラーで止まる。
Expected: コンソールに `holeLabel is not defined`

- [ ] **Step 3: 3関数を実装する**

`index.html` の `function raw(h,p){` の直前に挿入する。

```javascript
function holeLabel(pos,holes){
  if(!holes||holes.length!==2)return String(pos);
  const front=pos<=9;
  const name=front?holes[0]:holes[1];
  const isOutIn=String(name).toUpperCase()==="OUT"||String(name).toUpperCase()==="IN";
  if(isOutIn)return name+" "+(String(name).toUpperCase()==="IN"?(front?pos+9:pos):(front?pos:pos-9));
  return name+" "+(front?pos:pos-9);
}
function ninesToPars(nines,holes){
  if(!nines||!holes||holes.length!==2)return null;
  const a=nines[holes[0]],b=nines[holes[1]];
  if(!Array.isArray(a)||!Array.isArray(b)||a.length!==9||b.length!==9)return null;
  return a.concat(b);
}
function parsToNphDch(pars){
  const nph={},dch={};
  (pars||[]).forEach((p,i)=>{const h=i+1;if(p===3)nph[h]=true;else if(p>=5)dch[h]=true;});
  return {nph,dch};
}
```

`holeLabel` の `IN` の分岐に注意。`IN` が後半なら `pos`（10〜18）をそのまま、前半なら
`pos+9`（1〜9 → 10〜18）にする。`OUT` は逆に、前半なら `pos`、後半なら `pos-9`。

- [ ] **Step 4: 関数をテストページに貼り直し、全て PASS することを確認**

Expected: 12行すべて `PASS`

- [ ] **Step 5: コミット**

```bash
git add index.html
git commit -m "feat(holes): add hole label and par derivation helpers"
```

---

### Task 2: コースデータの読み込み

**Files:**
- Modify: `index.html`（Task 1 で足した関数群の直後）

**Interfaces:**
- Consumes: `ninesToPars`, `parsToNphDch`（Task 1）
- Produces:
  - `loadCourseNines(courseName) -> Promise<{key, displayName, nines} | null>`
  - `applyNineSelection()`（`D.courseNines` と `D.cfg.holes` から `D.nph`/`D.dch` を埋める）

`D` に `courseNines: null` を足す（読み込んだコースの `nines`）。
`D.cfg` に `holes` と `courseKey` を足す（既定は `null`）。

読み取りに失敗した場合・コースが未登録の場合はいずれも `null` を返し、**エラーダイアログは出さない**。
ナイン選択UIが出ないだけで、従来どおり手動設定できる状態を保つ。

- [ ] **Step 1: `D` の初期値に3つのフィールドを足す**

`index.html:233` の `let D={...}` の `cfg:{...}` 内に `holes:null,courseKey:null,` を、
`D` 直下に `courseNines:null,` を追加する。既存のフィールドは触らない。

- [ ] **Step 2: 読み込み関数を実装する**

```javascript
async function loadCourseNines(courseName){
  const raw=(courseName||"").trim();
  if(!raw||!window.database)return null;
  try{
    const aliasSnap=await window.database.ref("courseAliases/"+raw).once("value");
    const key=aliasSnap.val();
    if(!key)return null;
    const courseSnap=await window.database.ref("courses/"+key).once("value");
    const course=courseSnap.val();
    if(!course||!course.nines)return null;
    return {key:key,displayName:course.displayName||key,nines:course.nines};
  }catch(err){
    console.log("コース情報の取得に失敗:",err);
    return null;
  }
}
function applyNineSelection(){
  const pars=ninesToPars(D.courseNines,D.cfg.holes);
  if(!pars)return false;
  const r=parsToNphDch(pars);
  D.nph=r.nph;D.dch=r.dch;
  return true;
}
```

Firebase のキーは生の文字列をそのまま使う。バッチ側が生表記と正規化表記の両方を
`courseAliases` に登録済みなので、全角スペースを含む入力もそのまま解決できる。

- [ ] **Step 3: ブラウザで手動確認**

デバッグ用 Chrome でアプリを開き、サインインしてコンソールで実行する。

```javascript
await loadCourseNines("越生ゴルフクラブ")
// → {key:"越生ゴルフクラブ", displayName:"越生ゴルフクラブ", nines:{OUT:[...],IN:[...]}}
await loadCourseNines("太平洋　成田")
// → 全角スペースでも解決すること
await loadCourseNines("存在しないゴルフ場")
// → null（例外を投げないこと）
```

Expected: 1つ目と2つ目がオブジェクト、3つ目が `null`

- [ ] **Step 4: コミット**

```bash
git add index.html
git commit -m "feat(course): load shared course nine data from Firebase"
```

---

### Task 3: 設定画面のナイン選択UI

**Files:**
- Modify: `index.html`（`rSet()` 1139行付近、コース名入力の直後）

**Interfaces:**
- Consumes: `loadCourseNines`, `applyNineSelection`, `D.courseNines`, `D.cfg.holes`（Task 2）
- Produces: なし（UI のみ）

コース名の `oninput` は `save()` のみを呼んでいる。入力の途中で毎回 Firebase を引かないよう、
**`onchange`（確定時）でコース情報を読み込む**。

3ナイン以上の施設は前半・後半とも選択必須で既定を置かない。2ナインなら既定を出すが、
順序は変更できる。前半と後半に同じナインは選べない。

- [ ] **Step 1: コース名入力に onchange を足す**

現在の記述:

```javascript
oninput="D.cfg.courseName=this.value;save()"
```

これを次に変える。

```javascript
oninput="D.cfg.courseName=this.value;save()" onchange="onCourseNameChanged(this.value)"
```

- [ ] **Step 2: ハンドラとUI描画関数を実装する**

Task 2 の関数群の直後に追加する。

```javascript
async function onCourseNameChanged(name){
  const course=await loadCourseNines(name);
  D.courseNines=course?course.nines:null;
  D.cfg.courseKey=course?course.key:null;
  D.cfg.holes=null;
  const names=course?Object.keys(course.nines):[];
  if(names.length===2){D.cfg.holes=[names[0],names[1]];applyNineSelection();}
  save();render();
}
function setNine(idx,name){
  const cur=D.cfg.holes?D.cfg.holes.slice():[null,null];
  cur[idx]=name||null;
  if(cur[0]&&cur[1]&&cur[0]===cur[1])cur[idx===0?1:0]=null;
  D.cfg.holes=cur;
  if(cur[0]&&cur[1])applyNineSelection();
  save();render();
}
function rNineSelect(){
  if(!D.courseNines)return"";
  const names=Object.keys(D.courseNines);
  const cur=D.cfg.holes||[null,null];
  const sel=(idx)=>`<select class="input-full" onchange="setNine(${idx},this.value)">`
    +`<option value="">選択してください</option>`
    +names.map(n=>`<option value="${n}"${cur[idx]===n?" selected":""}>${n}</option>`).join("")
    +`</select>`;
  return`<div class="mb-6"><label class="label">回るナイン（${names.length}ナインのコース）</label>`
    +`<div style="display:flex;gap:8px;align-items:center">`
    +`<div style="flex:1"><div style="font-size:11px;color:#6b7280;margin-bottom:4px">前半</div>${sel(0)}</div>`
    +`<div style="flex:1"><div style="font-size:11px;color:#6b7280;margin-bottom:4px">後半</div>${sel(1)}</div>`
    +`</div>`
    +`<p style="font-size:11px;color:#6b7280;margin-top:6px">選ぶとニアピン・ドラコンが自動で入ります。個別に変えたい場合は下のボタンで調整できます。</p>`
    +`</div>`;
}
```

- [ ] **Step 3: `rSet()` にナイン選択を差し込む**

コース名の `</div>` とラウンド日の `<div class="mb-6">` の間に `${rNineSelect()}` を入れる。

- [ ] **Step 4: ブラウザで手動確認**

| 操作 | 期待 |
|---|---|
| コース名に `越生ゴルフクラブ` を入れてフォーカスを外す | 前半 OUT / 後半 IN が自動で入り、ニアピン・ドラコンが埋まる |
| 前半を IN、後半を OUT に変える | ニアピン・ドラコンが入れ替わる |
| コース名に `川越グリーンクロス` を入れる（3ナイン） | 両方「選択してください」。ニアピン等は変わらない |
| 前半と後半に同じナインを選ぶ | もう一方が空になる |
| コース名に未登録の名前を入れる | ナイン選択が出ない。従来どおり手動設定できる |
| 自動適用後にニアピンのボタンを手で押す | トグルでき、その値が保存される |

- [ ] **Step 5: コミット**

```bash
git add index.html
git commit -m "feat(settings): add front/back nine selection"
```

---

### Task 4: ホール番号の表示

**Files:**
- Modify: `index.html`（`rInput()` 1111行、`rList()` 1112〜1139行）

**Interfaces:**
- Consumes: `holeLabel`（Task 1）、`D.cfg.holes`
- Produces: なし（表示のみ）

`tOut()` / `tIn()` の**実装は変更しない**。位置基準の集計は正しいままで、
表の見出しに出す名前だけをナイン名にする。

- [ ] **Step 1: `rInput()` のホールボタンをラベルに差し替える**

`rInput()`（1111行）にはホール選択ボタンが2列ある。マークアップはこの形。

```javascript
[1,2,3,4,5,6,7,8,9].map(h=>`<button class="btn btn-hole ${D.hole===h?'active':''}"
  onclick="D.hole=${h};renderWithScroll()"
  style="background:${D.hole===h?'#16a34a':D.nph[h]?'#cffafe':D.dch[h]?'#f3e8ff':'#...'}">${h}</button>`)
```

ボタンの**表示文字だけ**を差し替える。

```javascript
>${h}</button>          →  >${holeLabel(h,D.cfg.holes)}</button>
```

`[10,...,18]` の列も同様。`onclick` の `D.hole=${h}` と `D.nph[h]` / `D.dch[h]` の参照は
**位置基準のままで正しいので触らない**。配列そのもの（位置の列挙）も変えない。

ボタン幅が足りずラベルが折り返す場合は、`btn-hole` の `min-width` を広げるより
ラベルを短くする方を選ぶ。ここは目視で判断してよい。

- [ ] **Step 2: `rList()` の行見出しをラベルに差し替える**

`[1,2,3,4,5,6,7,8,9].map(h=>...)` と `[10,...,18].map(h=>...)` の中で
`<td>${h}` としている箇所を `<td>${holeLabel(h,D.cfg.holes)}` にする。
配列そのもの（位置の列挙）は変えない。

- [ ] **Step 3: OUT/IN の集計見出しをナイン名にする**

`<b>OUT</b>` を `<b>${D.cfg.holes?D.cfg.holes[0]:"OUT"}</b>`、
`<b>IN</b>` を `<b>${D.cfg.holes?D.cfg.holes[1]:"IN"}</b>` にする。
`tOut(p)` / `tIn(p)` の呼び出しはそのまま。

- [ ] **Step 4: ブラウザで手動確認**

| ラウンド | 期待 |
|---|---|
| 既存の旧ラウンド（`holes` なし）を開く | 従来どおり `1`〜`18`、見出しは OUT / IN |
| `["OUT","IN"]` のラウンド | `OUT 1`〜`OUT 9` / `IN 10`〜`IN 18` |
| `["IN","OUT"]` のラウンド | 前半が `IN 10`〜`IN 18`、後半が `OUT 1`〜`OUT 9` |
| `["東コース","西コース"]` | 各ナイン `1`〜`9`。見出しが 東コース / 西コース |
| スコア入力・合計 | 旧ラウンドと同じ値になること（集計は位置基準のまま） |

- [ ] **Step 5: コミット**

```bash
git add index.html
git commit -m "feat(display): label holes by nine name and number"
```

---

### Task 5: 通しでの動作確認とPR

**Files:**
- Modify: なし（確認のみ）

- [ ] **Step 1: 新規ラウンドを作って通しで確認する**

デバッグ用 Chrome で、実際に新規ラウンドを作成し、コース名に `太平洋クラブ軽井沢リゾート`
（4ナイン）を入れて `浅間OUT` / `白樺IN` を選ぶ。スコアを数ホール入力し、
一覧表・合計・リアルタイム同期が壊れていないことを確認する。

- [ ] **Step 2: 既存ラウンドを開いて壊れていないことを確認する**

「📂 ラウンド管理」から過去のラウンドを開き、スコアと合計が従来どおり表示されることを確認する。

- [ ] **Step 3: PR を作る**

```bash
git push -u origin feat/nine-selection
gh pr create --base main --title "前半・後半のナイン選択とホール表示の変更"
```
