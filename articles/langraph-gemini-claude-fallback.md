---
title: "LangGraph StateGraph で Gemini/Claude マルチモデルフォールバックを実装する"
emoji: "🔄"
type: "tech"
topics: ["langgraph", "langchain", "gemini", "claude", "nodejs"]
published: false
---

## はじめに

AIを使ったAPIサーバーを本番運用していると、避けられない問題があります。それは **APIキーのレート制限** です。

Gemini APIは無料枠のリクエスト上限が厳しく、ユーザーが集中するとすぐに `429 Too Many Requests` が返ってきます。「APIキーを複数持てばよい」と思っても、複数キーのフォールバック処理を手書きするのは案外面倒です。さらに「Geminiが全滅したらClaudeに切り替えたい」という要件が加わると、状態管理のコードが複雑になりがちです。

この記事では、**LangGraph StateGraph** を使ってこのフォールバックロジックをエレガントに実装する方法を紹介します。

```
Gemini キー1 → 失敗
Gemini キー2 → 失敗  
Gemini キー3 → 失敗
     ↓ 2秒待機してリトライ
Gemini キー1〜3 → 全滅
     ↓
Claude (Haiku) → 成功 ✅
```

## 完成形のコード概要

今回実装したのは、AI見積もりシステムのバックエンドAPIサーバーです。Expressで作ったAPIエンドポイントに、LangGraphで組んだフォールバックグラフを組み込んでいます。

GitHubリポジトリ: [keita2399/estimate-ai-api](https://github.com/keita2399/estimate-ai-api)

## LangGraph とは？

LangGraphは、LangChainチームが開発した **ステートマシン形式のAIワークフロー構築ライブラリ** です。

- **ノード**: 処理の単位（AIの呼び出し、変換など）
- **エッジ**: ノード間の遷移
- **条件付きエッジ**: 状態に応じて分岐する遷移
- **State（状態）**: グラフ全体で共有されるデータ

ReActエージェントや複雑なマルチエージェントシステムを構築するのが得意ですが、今回のようなシンプルなフォールバックロジックにも活用できます。

## 実装

### 1. 状態の定義

まず、グラフ全体で共有する状態を定義します。

```javascript
import { StateGraph, END, START, Annotation } from "@langchain/langgraph";

const FallbackStateAnnotation = Annotation.Root({
  systemPrompt: Annotation(),
  userContent:  Annotation(),
  temperature:  Annotation({ default: () => 0.5, reducer: (_, next) => next }),
  model:        Annotation({ default: () => "2.5-flash", reducer: (_, next) => next }),
  result:       Annotation({ default: () => null, reducer: (_, next) => next }),
  keyIndex:     Annotation({ default: () => 0,    reducer: (_, next) => next }),
  round:        Annotation({ default: () => 0,    reducer: (_, next) => next }),
});
```

ポイントは `keyIndex` と `round` です。

- **`keyIndex`**: 現在試しているGeminiキーのインデックス（0, 1, 2 ...）
- **`round`**: リトライ回数（0回目で全キー失敗→1回目リトライ→それでも失敗→Claude）

`reducer: (_, next) => next` は「前の値を無視して、常に新しい値で上書きする」という意味です。

### 2. ノードの実装

#### Gemini を試すノード

```javascript
async function tryGeminiNode(state) {
  const keys = getGeminiKeys(); // 環境変数から最大3つのキーを取得
  if (state.keyIndex >= keys.length) {
    // キーが尽きた → ルーティングでClaudeへ
    return { keyIndex: state.keyIndex };
  }

  const modelName = state.model === "2.0-flash"
    ? "gemini-2.0-flash"
    : "gemini-2.5-flash";

  try {
    const llm = new ChatGoogleGenerativeAI({
      model: modelName,
      temperature: state.temperature ?? 0.5,
      apiKey: keys[state.keyIndex],
    });

    let fullText = "";
    const stream = await llm.stream([
      new SystemMessage(state.systemPrompt),
      new HumanMessage(state.userContent),
    ]);
    for await (const chunk of stream) {
      fullText += typeof chunk.content === "string" ? chunk.content : "";
    }

    if (fullText) {
      const parsed = tryParseJSON(fullText); // JSONパース試行
      if (parsed) {
        return { result: parsed, keyIndex: state.keyIndex }; // 成功
      }
    }
    // JSONパース失敗 → 次のキーへ
    return { keyIndex: state.keyIndex + 1 };
  } catch (error) {
    console.error(`Gemini key ${state.keyIndex + 1} exception:`, error.message);
    return { keyIndex: state.keyIndex + 1 }; // 失敗 → 次のキーへ
  }
}
```

#### Claude フォールバックノード

```javascript
async function tryClaudeNode(state) {
  const claudeKey = process.env.ANTHROPIC_API_KEY;
  if (!claudeKey) return { result: null };

  console.log("Falling back to Claude...");
  try {
    const llm = new ChatAnthropic({
      model: "claude-haiku-4-5-20251001",
      maxTokens: 8192,
      temperature: state.temperature ?? 0.5,
      anthropicApiKey: claudeKey,
    });

    let fullText = "";
    const stream = await llm.stream([
      new SystemMessage(state.systemPrompt + "\n\nJSON以外の文字は一切出力しないこと。"),
      new HumanMessage(state.userContent),
    ]);
    for await (const chunk of stream) {
      fullText += typeof chunk.content === "string" ? chunk.content : "";
    }

    if (fullText) {
      const parsed = tryParseJSON(fullText);
      if (parsed) return { result: parsed };
    }
    return { result: null };
  } catch (error) {
    console.error("Claude exception:", error.message);
    return { result: null };
  }
}
```

#### リトライ遅延ノード

```javascript
async function retryDelayNode(state) {
  await new Promise(r => setTimeout(r, 2000)); // 2秒待機
  return { round: state.round + 1, keyIndex: 0 }; // ラウンド更新・キーを先頭に戻す
}
```

### 3. ルーティングの実装

LangGraphの真価が発揮されるのがここです。`routeAfterGemini` は現在の状態を見て **次にどのノードへ進むか** を決定します。

```javascript
function routeAfterGemini(state) {
  if (state.result !== null) return END;         // 成功 → 終了
  
  const keys = getGeminiKeys();
  if (state.keyIndex < keys.length) return "tryGemini"; // まだキーが残っている → 次のキーへ
  
  // 全キー失敗
  if (state.round < 1) return "retryDelay";     // まだリトライしていない → 2秒後に再試行
  return "tryClaude";                            // リトライも失敗 → Claudeへ
}
```

条件を整理すると:
| 状態 | 遷移先 |
|------|--------|
| `result !== null` | END（成功） |
| `keyIndex < keys.length` | tryGemini（次のキー） |
| `round < 1` | retryDelay（2秒待機→全キー再試行） |
| それ以外 | tryClaude（最終フォールバック） |

### 4. グラフの組み立て

```javascript
const workflow = new StateGraph(FallbackStateAnnotation)
  .addNode("tryGemini",   tryGeminiNode)
  .addNode("tryClaude",   tryClaudeNode)
  .addNode("retryDelay",  retryDelayNode)
  .addEdge(START, "tryGemini")
  .addConditionalEdges("tryGemini", routeAfterGemini, {
    tryGemini:  "tryGemini",
    retryDelay: "retryDelay",
    tryClaude:  "tryClaude",
    [END]:       END,
  })
  .addEdge("retryDelay", "tryGemini")
  .addEdge("tryClaude",  END);

const fallbackGraph = workflow.compile();
```

グラフを図で表すと:

```
START
  ↓
tryGemini ──→ (result あり) ──→ END
  ↓ (失敗・次キーあり)
tryGemini [次のキーで再実行]
  ↓ (全キー失敗・round=0)
retryDelay (2秒待機)
  ↓
tryGemini [全キーを再実行]
  ↓ (全キー失敗・round=1)
tryClaude ──→ END
```

### 5. JSON修復パーサー

LLMのレスポンスは必ずしも完全なJSONではありません。出力が途中で切れたり、マークダウンのコードブロックで囲まれていたりします。そのような場合に対応するパーサーも実装しました。

```javascript
function tryParseJSON(text) {
  // まず素直にパース
  try { return JSON.parse(text); } catch { /* continue */ }
  
  // JSONブロックを正規表現で抽出
  const jsonMatch = text.match(/\{[\s\S]*\}/);
  if (jsonMatch) {
    try { return JSON.parse(jsonMatch[0]); } catch { /* continue */ }
  }

  // ブラケットの補完・クォートの修復を試みる
  const raw = jsonMatch?.[0] || text;
  let fixed = raw;
  // 末尾の不完全なキー・バリューペアを除去
  fixed = fixed.replace(/,\s*"[^"]*"?\s*:?\s*"?[^"]*$/, "");
  fixed = fixed.replace(/,\s*"[^"]*$/, "");
  // クォートが奇数なら閉じる
  const quoteCount = (fixed.match(/(?<!\\)"/g) || []).length;
  if (quoteCount % 2 !== 0) fixed += '"';
  // 未閉じのブラケット・ブレースを閉じる
  const opens  = (fixed.match(/[\[{]/g) || []).length;
  const closes = (fixed.match(/[\]}]/g) || []).length;
  for (let i = 0; i < opens - closes; i++) {
    const lastOpen = fixed.lastIndexOf("[") > fixed.lastIndexOf("{") ? "]" : "}";
    fixed += lastOpen;
  }
  try { return JSON.parse(fixed); } catch { /* continue */ }
  return null;
}
```

### 6. APIエンドポイントへの組み込み

```javascript
async function callGeminiDirect(systemPrompt, userContent, options = {}) {
  const finalState = await fallbackGraph.invoke({
    systemPrompt,
    userContent,
    temperature: options.temperature ?? 0.5,
    model:       options.model ?? "2.5-flash",
    keyIndex: 0,
    round:    0,
  });
  return finalState.result;
}

app.post("/api/ai-call", async (req, res) => {
  const { systemPrompt, userContent, temperature, model } = req.body;
  try {
    const result = await callGeminiDirect(systemPrompt, userContent, { temperature, model });
    if (result) return res.json(result);
    return res.status(502).json({ error: "AI応答を取得できませんでした" });
  } catch (error) {
    console.error("ai-call error:", error);
    return res.status(500).json({ error: "Internal error" });
  }
});
```

## 環境変数の設定

`.env` ファイルに以下を設定します:

```env
# Gemini APIキー（最大3つ）
GEMINI_API_KEY=your_key_1
GEMINI_API_KEY_2=your_key_2
GEMINI_API_KEY_3=your_key_3

# Claude フォールバック用
ANTHROPIC_API_KEY=your_anthropic_key
```

キーが1つしかない場合でも動作します。`getGeminiKeys()` は存在するキーだけ返します。

## フォールバック戦略の考え方

### なぜ2ラウンドなのか

Geminiの429エラーはレート制限によるものがほとんどです。一時的なものであれば2秒待つだけで復帰するケースが多いです。一方、長時間待つとAPIのレスポンスタイムが遅くなりユーザー体験が悪化します。

そのため「1回だけ短い待機→それでもダメならClaudeへ」という2ラウンド戦略を採用しています。

### Claudeのシステムプロンプト追記

Claudeに切り替える際、システムプロンプトに `"JSON以外の文字は一切出力しないこと。"` を追記しています。これはGeminiより律儀に説明文を添えてくることがあるためです。

## まとめ

LangGraphを使うことで、複雑なフォールバックロジックを **宣言的かつ可視化しやすい形** で実装できました。

メリットをまとめると:
- **ロジックが図として理解できる**: ノードとエッジで流れが一目瞭然
- **状態管理が明示的**: `Annotation.Root` で状態の型が明確
- **拡張が容易**: 新しいモデルを追加したい場合、ノードを追加するだけ

今後はGrokやPerplexity APIなどを追加する際も、ノードを1つ足すだけで対応できます。

フォールバック構成のあるAIアプリを作る際の参考になれば幸いです！
