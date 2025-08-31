## はじめに

先日、社内プロダクトのフロントエンド基盤を Next.js v14 から v15 にアップデートしました。
本記事では、実施した進め方の全体像と、今回のNext15破壊的変更の背景をご紹介します。

## Next15へのアップデートの背景

対象アプリは、社内エージェントの生産性向上や、採用企業の活動を高度化するための Web システムです。候補者検索、スカウト送信、メッセージ送受信などの機能を備え、Next.js 14／React 18 系で運用していました。

しかし課題として、Next.js のデフォルトキャッシュや状態管理設計に起因する「使いづらさ」がありました。
UX 改善のためリファクタを進めたい一方で、先にメジャーアップデートを済ませておく方が手戻りが減り、最新機能を取り込んだ設計も可能になると判断し、まず Next.js 15／React 19 への移行に着手しました。

## 進め方

まずは方針・スコープを明確にしました。
既存仕様を保ったまま Next.js 15／React 19 に上げることを目的とし、新機能追加や最適化といったリファクタはスコープ外とします。スコープを広げすぎると見落としやレビュー負荷が増えるためです。

### 破壊的変更の整理と事前調査

次に、破壊的変更を洗い出し、概要・背景・移行方法を一次情報で確認してドキュメント化しました。
App Router は過渡期にあり、表層の変更だけでなく「なぜそうなったか」の背景を理解しておくと、その後の設計や実装で手戻りを極力避けられます。
変更の背景については以降の章で解説するのでここでは割愛します。

今回のNext.jsの破壊的変更は２つです。

#### 1. リクエスト API の非同期化

リクエスト固有のデータに依存する API（`headers`、`cookies`、`params`、`searchParams`）が非同期化されました。
移行はシンプルで、例えば `cookies` は次のように `await` が必要になります。

```ts
import { cookies } from 'next/headers'

export default async function Page() {
  const cookieStore = await cookies()
  // ここで cookieStore を利用
  return null
}
```

同期版は v15 では警告付きで残りますが、v16 で削除予定です。
公式の codemod（`next-async-request-api`）で大部分は自動移行できます。

#### 2. キャッシュのデフォルト無効化

変更は３つです。
1. Data Cache のデフォルトが `force-cache` から `no-store` へ変更。
2. Router Cache の `staleTime` のデフォルトが静的 5 分／動的 30 秒から双方 0 秒に変更（静的な `layout.tsx` や `loading.tsx` などは従来どおりキャッシュ対象）。
3. Route Handler（GET）のデフォルトがキャッシュなしに変更。

既存挙動を保ちたい箇所では、例えば次のように明示します。

```ts
// Data Cache 相当
await fetch(url, { cache: 'force-cache' })

// Route Handler を静的化
export const dynamic = 'force-static'
```

#### 補足：React 19 の破壊的変更

基本的には、これまで非推奨だった React API の削除が中心で、関数コンポーネントでは通常影響しません。
たとえば旧来のクラスコンポーネント向けライフサイクル API（例: `componentWillMount` 系）などは削除対象です。

ただしアプリ側で未使用であっても、ライブラリ側がクラスコンポーネントの場合は内部で非推奨 API を使っている可能性があるため、依存の更新とリリースノート確認を推奨します。

### 既存プロダクトへの影響調査と対応方針の策定

整理した資料をもとに、「プロダクトコード」と「ライブラリ」に分けて既存影響と対応方針を整理しました。

#### プロダクトコード

リクエストAPIの非同期化については、公式のcodemod（`next-async-request-api`）で自動移行し、手動で最終調整する方針にしました。

キャッシュのデフォルト無効化に対しては、下記方針にしました。
1. Data Cacheは、fetchをラップしているAPIクライアントの関数で既存挙動を維持したいリクエストに `cache: 'force-cache'` 明示的に指定。（ `cache: 'no-store'` はすでに指定済み）
2. Router Cacheは、`next.config` で Router Cache の `staleTime` を静的 300 秒、動的 30 秒に明示的に指定し、v14の挙動を維持。
3. Route Handler（GET）は、既存挙動を維持したい箇所に `export const dynamic = 'force-static'` を明示的に指定。


**1. Data Cache**
```ts:app/lib/getUser.ts
export async function getUser(id: string) {
  const res = await fetch(`https://api.example.com/users/${id}`, {
    // 既存挙動維持のための例: 明示的に Data Cache を使う
    cache: 'force-cache',
    next: { tags: ['user', id] },
  });
  if (!res.ok) throw new Error('Failed');
  return res.json();
}
```

**２. Router Cache**
```ts:next.config.ts
const nextConfig = {
  experimental: {
    // TODO: 既存動作維持のため、一旦v14でのデフォルト値を設定。別途最適化する
    staleTimes: {
      static: 300, // 5分
      dynamic: 30, // 30秒
    },
  },
};
export default nextConfig;
```

**３. Route Handler（GET）**
```ts:app/api/hello/route.ts
export const dynamic = 'force-static';

export async function GET() {
  return Response.json({ ok: true });
}
```

#### ライブラリ

ライブラリは lock ファイルの `peerDependencies` を起点に、Next15／React19 への対応状況を確認し、定義が緩いもの（例: `"react": ">=16"`）については、公式やリリースノートで確認しました。

調査内容を、「現行バージョン」「Next15・React19 対応状況」「補足」の 3 点で一覧化し、アップデートかリプレイスかの方針を決めます。
以下は一例です。

| ライブラリ | 現バージョン | 対応状況 | 補足 |
| --- | --- | --- | --- |
| react-dnd | 16.0.1 | ❌ 未対応 | React 19 を peer に含めず Issue #3655 が未解決。3年間未更新。`dnd-kit` 3.x への置換を推奨。 |
| @radix-ui/* | 1.1.x | ⚠️ 要アップグレード | 1.2.11+ で React 19 の peer 警告を解消。`shadcn/ui@latest` で一括更新可。 |
| react-hook-form | 7.50.1 | ⚠️ 要アップグレード | 7.56.0+ で React 19 peer 追加。 

### アップデート手順整理と実施

手順を下記の様に整理し、実施作業はAIに任せました。
Cursorを使用し、下記手順と今まで整理した資料を読み込ませ、ステップごとに確認を挟みながら進めました。

:::details cursor-rule（サンプル）

```md

▪️概要
agentモードでコードを修正する際に、認識齟齬や手戻りを防ぐために必ず守るべき内容

▪️ルール
・下記修正手順を１つずつ順番に行ってください。
・ステップごとにユーザーに確認をして、OKと言われたら次のステップを行ってください。
・OKという前に勝手に次のステップに行くことは禁止です。
・不明点やわからないことがあれば勝手に修正せずユーザーに確認してください。
・指示されていないが修正が必要なことがある場合は、一度ユーザーに確認をしてください。
・確認事項や質問がある場合は回答しやすいように採番してください。

▪️実装修正手順
１）背景・目的・要件の理解（細かい仕様や修正内容はここでは確認せず、3の設計の段階で聞いてください）
２）既存仕様・既存コード・周辺コードの理解
３）実装・修正方法の設計・計画（詳細にコードまで出して）
４）コードの修正（修正が多い場合は４での計画に沿って順番に修正し、都度ユーザーに確認を取る）
５）5の後、ユーザーから修正依頼の対応（追加の修正依頼についても修正計画を立ててから修正してください）
```

:::


#### 手順
- TypeScript を 5.8.3 に上げる（React 19 の型エラー検出に v5.5+ が必要）。
- React を 18.3 系へ上げ、codemod で v19 に向けた警告を解消します（移行の助走）。
- Next を 14 の最新（v14.2.28）へ更新して差分を最小化。
- Next 15／React 19と依存ライブラリをまとめて更新。
- codemod で自動更新。
- codemod で対応できない破壊的変更は、あらかじめ定めた方針どおり個別に修正。
- 未対応ライブラリのリプレイス。

### 最終確認

仕上げはコードレビューと動作確認です。現状フロントエンドの CI は ESLint、Prettier、tsc、build チェックのみで自動テストがないため、主要導線は目視で確認しました。
また、ローカルと Vercel環境ではキャッシュやレンダリングの挙動が異なるため、必ず Vercel の環境で確認します。

## Next15破壊的変更の背景
※背景についてはファクトだけではなく自分の見解も含まれております。ご容赦いただけますと幸いです。

### 1. リクエスト API の非同期化

今までは `headers`、`cookies`、`params`、`searchParams` などの動的APIやfetchの `cache: 'no-store' ` があると、 Full Route Cache（≒SSG）から外れ、ルート全体が動的レンダリングになっていました。
しかし、すべてのコンポーネントがリクエストに依存するわけではなく、同じルート内でも事前にレンダリングできる部分はあります。

現在実験版であるPPR（Partial Pre-rendering）を見据え、静的部分と動的部分を完全に分離し、Next側にいつリクエストを待つべきかを伝えるため、今回リクエスト依存 API が非同期化されました。

これにより同じルート内で、静的レンダリング（≒SSG）と動的レンダリング（≒SSR）が併用できるようになり、静的部分はビルド時にレンダリングして即表示、動的部分はリクエストごとにレンダリングするという設計ができるようになります。

簡単な例:

```tsx:app/page.tsx
export const experimental_ppr = true;
import { Suspense } from 'react';

export default function Page() {
  return (
    <>
      <StaticSection /> {/* ビルド時にレンダリング */}
      <Suspense fallback={<Loading />}> {/* 部分的にローディングUI表示 */}
        <DynamicSection /> {/* リクエスト時にレンダリング */}
      </Suspense>
    </>
  );
}
```

### 2. キャッシュのデフォルト無効化

Next.jsのキャッシュの複雑さ（特に Data Cache と Router Cache ）は、開発体験やデータ鮮度の観点でしばしばコミュニティなどで問題になっていました。

今後は従来の4つのキャッシュから、`use cache` （実験版） による境界ごとの制御で、キャッシュ戦略のシンプル化に舵を切っていると感じます。
その前段階として、キャッシュのデフォルト無効化の変更に繋がったと考えています。

`use cache` は、「ルート」「コンポーネント」「関数」単位で指定可能です。
先ほどのPPRと併用し、「動的レンダリング +  `use cache` 」にすることで、二回目以降の実行はキャッシュから返したり（≒従来の Data Cache や Router Cache ）、オンデマンドや時間ベースでの再検証（従来の `revalidateTag` やISR相当）をすることも可能です。

サーバー側については、PPRと `use cache` で静的レンダリングと動的レンダリングを最適化し、フロント側については、React Compiler（実験版）やTanStack Queryなどを組み合わせて最適化していく、という方向性に感じます。

```ts:next.config.ts
import type { NextConfig } from 'next'
 
const nextConfig: NextConfig = {
  experimental: {
    useCache: true, // 有効化
  },
}
 
export default nextConfig
```

```ts
// File level
'use cache'
 
export default async function Page() {
  // ...
}
 
// Component level
export async function MyComponent() {
  'use cache'
  return <></>
}
 
// Function level
export async function getData() {
  'use cache'
  const data = await fetch('/api/data')
  return data
}
```

## 最後に

App Router 版の Next.js はリリースからまだ 2 年ほどの過渡期にあります。設計や実装で手戻りを避けるためには、表層の移行内容だけでなく、破壊的変更の狙いや追加機能の価値を理解しておくことがいっそう重要だと感じています。

特に AI 時代のいま、ベストプラクティスが固まりきっていない領域では、AI の出力をうのみにせず、一次情報を踏まえた上で適切な判断ができる知識は、これからも欠かせないなと思いました。
最後まで読んでいただき、ありがとうございました。

## 参考

- [Next.js 15 アップデート（公式ブログ）](https://nextjs.org/blog/next-15)
- [Async Request APIs と codemod ガイド](https://nextjs.org/docs/app/building-your-application/rendering/server-components#async-request-apis)
- [Caching: Data Cache と Router Cache](https://nextjs.org/docs/app/building-your-application/caching)
- [Route Handlers とキャッシュ](https://nextjs.org/docs/app/building-your-application/routing/route-handlers)
- [Partial Pre-rendering (PPR)](https://nextjs.org/docs/app/building-your-application/rendering/partial-prerendering)
- [use cache](https://nextjs.org/docs/app/api-reference/directives/use-cache)
