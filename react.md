# React

## useEffect

レンダリング後に副作用（DOM操作・購読・タイマー・外部システムとの同期）を実行するためのフック。レンダリング中にやってはいけない処理を、レンダリング後に切り出して実行する。

> [!info] そもそも「副作用」とは
> コンポーネントの返り値（JSX）以外でアプリ外部に影響する処理。fetch・サブスクライブ・setIntervalなど、「描画した結果として外の世界をいじるもの」が対象。

### クリーンアップを書く

`useEffect`に渡した関数から関数を返すと、それがクリーンアップ関数になる。次のEffectが走る前と、アンマウント時に呼ばれる。

```tsx
useEffect(() => {
  const id = setInterval(() => {
    console.log("tick");
  }, 1000);

  return () => {
    clearInterval(id);
  };
}, []);
```

> [!info] なぜ「次のEffect実行前」にも呼ばれるか
> 依存配列が変わるたびにEffectは再実行される。そのとき前回のEffectが残したリソース（古いタイマー・古い購読）を畳まないと多重に走る。クリーンアップは「前回の自分の後始末」と「アンマウント時の後始末」を兼ねている。

### クリーンアップが必要なシチュエーション

副作用が「外部に何かを残す」ときは必ずクリーンアップする。

- `setInterval` / `setTimeout` → `clearInterval` / `clearTimeout`
- `addEventListener` → `removeEventListener`
- WebSocket・SSE・Firebaseなどの購読 → `unsubscribe()` / `close()`
- fetch中のリクエスト → `AbortController.abort()`
- DOMへの直接操作で要素を追加した場合 → 削除

```tsx
useEffect(() => {
  const controller = new AbortController();

  fetch(`/api/users/${userId}`, { signal: controller.signal })
    .then((res) => res.json())
    .then(setUser);

  return () => controller.abort();
}, [userId]);
```

> [!info] なぜfetchもクリーンアップするか
> `userId`が連続で切り替わると、古いリクエストが遅れて返ってきて新しい状態を上書きする「レースコンディション」が起きる。`AbortController`で前回のリクエストを打ち切れば、古い結果は捨てられる。

### クリーンアップの正しい書き方

クリーンアップの中で参照する値は、Effect本体で取得した値を**クロージャで閉じ込める**。クリーンアップ実行時に再取得しない。

```tsx
useEffect(() => {
  const ws = new WebSocket(url);
  ws.addEventListener("message", handleMessage);

  return () => {
    ws.removeEventListener("message", handleMessage);
    ws.close();
  };
}, [url]);
```

> [!info] なぜ閉じ込めるか
> クリーンアップが呼ばれる時点では、すでに`url`が新しい値に変わっている可能性がある。`new WebSocket(url)`でもう一度作り直すと、閉じるべき「古いWebSocket」とは別物を閉じてしまう。Effect本体で作ったインスタンスをそのまま閉じる。

### 依存配列を正しく書く

Effect内で参照しているリアクティブな値（props・state・それらから派生した値）は全て依存配列に入れる。

- `[]` → マウント時に1回だけ
- `[a, b]` → `a`か`b`が変わったとき再実行
- 省略 → 毎レンダー実行（基本使わない）

> [!info] なぜ`[]`を安易に使わないか
> 「1回だけ動かしたい」気持ちで`[]`にすると、本当はpropsの変化に追従すべきEffectが古い値で固まる。依存を省くのではなく、「そもそもEffectでやるべきか」を疑う。

### useEffectを使わない判断

- レンダリング中に計算できる値 → そのままJSX内で計算する。`useMemo`で十分なら`useEffect` + `useState`にしない
- イベントに反応する処理 → イベントハンドラに書く（クリックで送信、など）
- 親に通知する状態変化 → 親のイベントハンドラ側で計算してpropsで渡す

> [!info] なぜ避けるか
> Effectは「レンダー結果を外部に同期する」ための仕組み。ユーザー操作や算出値の用途で使うと、無駄な再レンダー・状態のズレ・複雑化を生む。「外部システムと同期しているか？」をチェックリストにする。

### StrictModeで2回呼ばれる

開発モード（`React.StrictMode`）では、Effectがマウント時に意図的に2回実行される。クリーンアップが正しく書けていないとバグが顕在化する。

> [!info] これは仕様
> 「マウント→アンマウント→再マウント」を擬似的に再現して、クリーンアップ漏れを早期に検出するため。本番ビルドでは1回だけ。2回呼ばれて困る実装は、そもそもクリーンアップが不完全。

## useState

### Lazy initializerで初期値計算を遅延する

`useState`・`useReducer`の初期値に**関数**を渡すと、初回レンダー時にだけその関数が実行される。値を直接渡すと毎レンダー評価される（結果は捨てられるが計算自体は走る）。

```tsx
// NG: 毎レンダーでlocalStorageを読む
const [items, setItems] = useState(JSON.parse(localStorage.getItem("items") ?? "[]"));

// OK: 初回レンダーでだけ実行
const [items, setItems] = useState(() => JSON.parse(localStorage.getItem("items") ?? "[]"));
```

> [!info] なぜ毎レンダー走るのか
> `useState(expr)`の`expr`は通常のJS式なので、関数呼び出し時点で必ず評価される。Reactが「初回だけ使う」と判断するのは引数が**関数**のときだけ。重い計算・I/O・JSON.parseは関数で包む。

### Lazy initializerを使うシチュエーション

初期値の計算が「重い」または「副作用に近い」とき。

- `localStorage` / `sessionStorage`からの読み出し
- 大きな配列・オブジェクトの生成（`Array.from({ length: 10000 })`など）
- JSON.parse・正規表現コンパイル
- propsから派生した複雑な初期構造の構築

```tsx
function TodoList({ initialTodos }: { initialTodos: Todo[] }) {
  const [todos, setTodos] = useState(() =>
    initialTodos.map((t) => ({ ...t, normalized: normalize(t.text) })),
  );
}
```

> [!info] propsを直接初期値にする場合は不要
> `useState(props.value)`のように単に値を代入するだけなら、毎レンダー評価されてもコストはほぼゼロ。Lazy化は「計算が重い」「副作用がある」ときの最適化であって、習慣的に全部関数化する必要はない。

> [!info] propsの変化には追従しない点に注意
> `useState`の初期値は初回レンダーのみ使われる。あとからpropsが変わっても状態は更新されない。propsに同期したいなら`useEffect`か、そもそもstateを持たずpropsをそのまま使うかを検討する。
