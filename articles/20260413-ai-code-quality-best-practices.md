---
title: "AIと一緒に作るコードの品質をどう担保するか、Best Practicesから学んだこと"
emoji: "🔧"
type: "tech"
topics: ["claudecode", "ai", "typescript", "nextjs", "githooks"]
published: true
---

## 今日やったこと

Claude Code の公式 Best Practices ドキュメントを読んで、
今作っているアプリの実装を見直した。

「AIが書いたコードだからなんとなく不安」という感覚があったので、
その不安を言語化してちゃんと対処できたのが今日の収穫だった。

## 見直して直したこと

### 1. race condition の修正

マップ上のマーカーを非同期で描画する処理で、こういうコードになっていた。

```typescript
useEffect(() => {
  const updateMarkers = async () => {
    const { AdvancedMarkerElement } = await importLibrary("marker");
    // ← ここで await している間にコンポーネントがアンマウントされると？
    markers.forEach(m => m.setMap(mapInstance));
  };
  updateMarkers();
}, [restaurants]);
```

`importLibrary` の await 中にコンポーネントがアンマウントされると、
存在しない DOM に対して操作しようとしてエラーになる。

修正はシンプルで、`cancelled` フラグを入れるだけ。

```typescript
useEffect(() => {
  let cancelled = false;
  const updateMarkers = async () => {
    const { AdvancedMarkerElement } = await importLibrary("marker");
    if (cancelled) return; // アンマウント済みなら何もしない
    markers.forEach(m => m.setMap(mapInstance));
  };
  updateMarkers();
  return () => { cancelled = true; };
}, [restaurants]);
```

Claude が最初に書いたコードには `cancelled` フラグがなかった。
動作確認だけでは発見しにくいバグで、Best Practices の
「実装前に探索してから計画を立てる」という観点で改めて眺めたときに気づいた。

### 2. APIキー未設定時のフォールバックUI

`NEXT_PUBLIC_GOOGLE_MAPS_API_KEY` が空のまま動かすと、
マップコンポーネントが真っ白になるだけで何のメッセージも出なかった。

```typescript
if (!process.env.NEXT_PUBLIC_GOOGLE_MAPS_API_KEY) {
  return (
    <div className="w-full h-full flex items-center justify-center bg-gray-100">
      <p className="text-gray-500 text-sm">
        Google Maps API キーが設定されていません
      </p>
    </div>
  );
}
```

地味だけど、開発者体験として重要。環境変数を設定し忘れたときに
「なぜ動かないのか」がすぐわかる。

### 3. PostToolUse hook で型チェックを自動化

Claude Code には hook 機能がある。ファイルを編集するたびに
自動でコマンドを走らせることができる。

`.claude/settings.json` に以下を追加した。

```json
{
  "hooks": {
    "PostToolUse": [{
      "matcher": "Edit|Write",
      "hooks": [{
        "type": "command",
        "command": "cd /workspace/myapp && npx tsc --noEmit --pretty false 2>&1 | head -20"
      }]
    }]
  }
}
```

これで、Claude がファイルを編集するたびに TypeScript の型チェックが走る。
エラーがあれば Claude がそのまま気づいて修正する流れになる。

「AIが書いたコードの型安全性をどう確認するか」という問題を、
仕組みで解決できた感じがして気持ちよかった。

### 4. CLAUDE.md に必須事項を明文化

プロジェクトルートの `CLAUDE.md` に、
環境変数の一覧と型チェックコマンドを追記した。

Claude Code はセッション開始時に `CLAUDE.md` を読み込むので、
「このプロジェクトで守るべきこと」をここに書いておくと
毎回言わなくても共有できる。

## 学んだこと

AIが書いたコードは「動く」かどうかは比較的確認しやすいが、
「壊れないか」はちゃんと意識しないと見落とす。

Best Practices を読んで一番刺さったのは
**「Claude を信頼しすぎず、仕組みで品質を担保せよ」** という考え方だった。

hook で自動テスト、CLAUDE.md でルール共有、
フォールバック UI でエラーを可視化——これらは全部
「AIに任せつつ、人間がコントロールを握る」ための設計だと思う。

完全に任せるのでも、全部自分でやるのでもない。
そのバランスをどう設計するかが、AI 時代の開発者の仕事なのかもしれない。
