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

### レースコンディションを潰す

非同期処理の結果が、依存値が変わった後に遅れて返ってきて、新しい状態を古い結果で上書きする現象。`userId`や検索クエリなど、依存配列の値が連続で切り替わるときに起きやすい。

```tsx
// NG: クリーンアップなし
useEffect(() => {
  fetch(`/api/users/${userId}`)
    .then((res) => res.json())
    .then(setUser);
}, [userId]);
```

`userId`が`1 → 2`と素早く切り替わったとき、`/api/users/1`のレスポンスが遅れて返ると、せっかく`2`で上書きした`user`が`1`の結果で再上書きされる。

#### 対策1: AbortController

fetchや`AbortSignal`を受け取れるAPIなら、クリーンアップで`controller.abort()`を呼ぶ。古いリクエスト自体を打ち切る。コード例は前節「クリーンアップが必要なシチュエーション」を参照。

#### 対策2: ignoreフラグ

`AbortSignal`に対応していないAPI（古いSDK、独自のPromise関数）では、クロージャでフラグを持ち、クリーンアップで`true`にして結果を無視する。

```tsx
useEffect(() => {
  let ignore = false;

  fetchUser(userId).then((data) => {
    if (!ignore) setUser(data);
  });

  return () => {
    ignore = true;
  };
}, [userId]);
```

> [!info] どちらを使うか
> fetchやaxiosなど`signal`を渡せるAPIは`AbortController`優先。ネットワーク自体を打ち切れるのでリソース効率が良い。Firebase SDKや独自Promiseなど打ち切り手段がないものはignoreフラグ。「結果を捨てるだけ」なので汎用的に効く。

> [!info] なぜクリーンアップで解決できるか
> 依存配列が変わるとReactは「次のEffect実行前」にクリーンアップを呼ぶ。これは「古い非同期処理の後始末」とそのまま一致する。レースコンディション対策はクリーンアップの本質的な用途の一つ。

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

## useRef

「再レンダーをトリガーしない可変値の入れ物」と「DOMノードへの参照」の二役を担うフック。`ref.current`に値を書き込んでも再レンダーは走らない。

### DOM要素を参照する

focus、scroll、video再生、外部ライブラリ（Mapboxなど）へのDOMノード受け渡しのように、宣言的に表現できない命令的操作で使う。

```tsx
function SearchBox() {
  const inputRef = useRef<HTMLInputElement>(null);

  useEffect(() => {
    inputRef.current?.focus();
  }, []);

  return <input ref={inputRef} />;
}
```

> [!info] なぜstateやpropsで済まないか
> 値や`disabled`はpropsで宣言できるが、focus・scroll・`play()`はDOMノードに対する命令でpropsの形では表現できない。Reactの宣言的世界から「外」に出る必要があるときの出口がref。

### 再レンダーしない可変値を持つ

コンポーネントの寿命の間、値を保持したいが、UIには反映しなくていい場面で使う。

- タイマーID・IntervalID（クリーンアップ時に参照したい）
- 外部ライブラリのインスタンス（WebSocket、Mapインスタンスなど）
- 直前の値の保持（前回propsとの比較）

```tsx
function Stopwatch() {
  const intervalRef = useRef<number | null>(null);

  function start() {
    intervalRef.current = window.setInterval(() => {
      // tick
    }, 1000);
  }

  function stop() {
    if (intervalRef.current !== null) {
      clearInterval(intervalRef.current);
      intervalRef.current = null;
    }
  }

  return <button onClick={start}>start</button>;
}
```

> [!info] なぜstateにしないか
> stateを更新すると再レンダーが走る。タイマーIDは画面に出さないので`setState`は無駄なコスト。「保持はしたいが画面には出さない」値はrefの守備範囲。

### useStateとの使い分け

判断軸は1つだけ。「その値が変わったらUIを再描画する必要があるか」。

| 値の性質 | 使うもの |
| --- | --- |
| 変わったら画面に反映したい | `useState` |
| 保持はしたいが画面に出さない | `useRef` |
| DOMノード自体を触りたい | `useRef` |

> [!info] 迷ったら`useState`から
> 表示しないつもりでも、後から表示が必要になることはよくある。性能上の問題が見えてから`useRef`に倒すほうが事故が少ない。先回りでrefにすると、表示したくなったときに「再レンダーが走らなくて画面が更新されない」バグになる。

### レンダリング中にrefを読み書きしない

`ref.current`はレンダー関数本体（JSXを返すまで）では読まない・書かない。Effectやイベントハンドラの中だけで扱う。

```tsx
// NG: レンダー中にcurrentを書き換える
function Counter() {
  const ref = useRef(0);
  ref.current += 1;
  return <div>{ref.current}</div>;
}

// OK: ハンドラやEffectの中で扱う
function Counter() {
  const ref = useRef(0);

  function handleClick() {
    ref.current += 1;
  }

  return <button onClick={handleClick}>click</button>;
}
```

> [!info] なぜレンダー中がNGか
> Reactはレンダーを中断・再実行・並行実行する前提で設計されている（Concurrent Features）。レンダー中に`ref.current`を書き換えると、再実行のたびに値が累積して結果が予測不能になる。「レンダーは純粋関数」というReactの前提を壊す。
