---
title: "スマホで思いついたアイデアが、PCの開発環境に自動で届く仕組みを作った"
emoji: "📱"
type: "tech"
topics: ["LINE", "ClaudeCode", "GeminiAPI", "GitHub", "開発効率化"]
published: true
---

## はじめに

開発中に「あ、あの機能こうした方がいい」と思いつくのは、たいていPCの前じゃない時です。電車の中、散歩中、寝る前のベッド。LINEにメモしても、翌朝PCを開いた時には忘れている。

この問題を解決するために、**LINEでAIに話しかけると、PCのClaude Code開発環境に会話ログが自動同期される仕組み**を作りました。

```
スマホ（LINE）→ AIと会話 → GitHub Gist → PC（5分おきに同期）→ CLAUDE.md
```

Claude Codeは起動時に `~/.claude/CLAUDE.md` を読み込みます。ここにLINEの会話ログが書き込まれていれば、PCでClaude Codeを開いた瞬間に「さっきLINEで話してたこと」を知っている状態になります。

DBなし、ほぼ無料、232行のTypeScript + 60行のシェルスクリプトで動いています。

## 全体アーキテクチャ

```
┌─────────────┐     ┌──────────────────┐     ┌────────────┐     ┌──────────────┐
│  LINE App   │────▶│  Vercel          │────▶│  GitHub    │────▶│  PC          │
│  (スマホ)   │◀────│  (Webhook+AI)    │◀────│  Gist      │     │  (5分同期)   │
└─────────────┘     └──────────────────┘     └────────────┘     └──────────────┘
   ユーザー          署名検証                   conversation.json    sync-gist.ps1
   テキスト入力      AI応答(Claude/Gemini)      claude-memo.md       ↓
   AI返信受信        Gist読み書き               settings.json     ~/.claude/CLAUDE.md
                                                                     ↓
                                                                  Claude Code CLI
```

4つのサービスが連携していますが、自分で書くコードは2ファイルだけです。

## なぜGitHub Gistなのか

普通ならDBを使うところですが、GitHub Gistをクラウドストレージとして使っています。

| 比較 | DB（PostgreSQL等） | GitHub Gist |
|------|-------------------|-------------|
| コスト | 月額$5〜 | 無料 |
| セットアップ | スキーマ設計、マイグレーション | Gist作成だけ |
| バージョン管理 | 自分で実装 | Gitで自動管理 |
| 管理画面 | 自分で作る | GitHubで直接見れる |
| スケーリング | 高 | 低（個人用途なら十分） |

必要なのは「会話ログをJSON/Markdownで保存して、PCから読み取る」だけ。DBは明らかにオーバーキルです。

Gistには3つのファイルを置いています。

```
conversation.json  ← AI APIに渡す会話コンテキスト（最新20件）
claude-memo.md     ← 人間が読めるログ（タイムスタンプ付き）
settings.json      ← 現在のAIモデル設定（claude or gemini）
```

## Webhookハンドラー — 232行の全機能

LINE Botが受け取ったメッセージを処理する部分です。Next.jsのAPIルートで実装しています。

### 処理の流れ

```typescript
export async function POST(req: NextRequest) {
  // 1. LINE署名検証（偽リクエスト排除）
  const signature = req.headers.get("x-line-signature") || "";
  if (!verifySignature(body, signature)) {
    return NextResponse.json({ error: "Invalid signature" }, { status: 401 });
  }

  // 2. Gistから会話履歴・設定を読み取り
  const { conversation, log, model } = await readGist();

  // 3. モデル切替コマンドのチェック（「Claude」「Gemini」で切替）
  const switchTo = parseModelSwitch(userText);
  if (switchTo) {
    await updateGist(conversation, log, switchTo);
    await replyToLine(replyToken, `${switchTo} に切り替えました`);
    return;
  }

  // 4. AIに問い合わせ
  const aiResponse = await askAI(conversation, model);

  // 5. Gist更新（会話 + ログ + 設定）
  await updateGist(updatedConversation, updatedLog);

  // 6. LINEに返信
  await replyToLine(replyToken, aiResponse);
}
```

### 署名検証 — Webhookの玄関

```typescript
function verifySignature(body: string, signature: string): boolean {
  const hash = crypto
    .createHmac("SHA256", LINE_CHANNEL_SECRET)
    .update(body)
    .digest("base64");
  return hash === signature;
}
```

LINE Messaging APIは、リクエストのbodyをチャネルシークレットでHMAC-SHA256署名して `x-line-signature` ヘッダーに付与します。サーバー側で同じ計算をして一致を確認。これがないと、誰でも偽のWebhookを送れてしまいます。

### Gistの読み書き — DBの代わり

```typescript
async function readGist() {
  const res = await fetch(`https://api.github.com/gists/${GIST_ID}`, {
    headers: { Authorization: `token ${GITHUB_TOKEN}` },
  });
  const data = await res.json();

  // 3ファイルからデータ抽出
  const conversation = JSON.parse(data.files?.["conversation.json"]?.content || "[]");
  const log = data.files?.["claude-memo.md"]?.content || "";
  const model = JSON.parse(data.files?.["settings.json"]?.content || '{"model":"gemini"}').model;

  return { conversation, log, model };
}
```

GitHub Gist APIは `GET` で全ファイルの内容を一括取得、`PATCH` で一括更新できます。1リクエストで3ファイルを原子的に更新できるので、整合性の問題も起きません。

## Claude / Gemini の切替 — 2つのAIを1行で

LINEで「Claude」と送ればClaude API、「Gemini」と送ればGemini APIに切り替わります。

```typescript
function parseModelSwitch(text: string): ModelType | null {
  if (/^(gemini|ジェミニ)$/i.test(text.trim())) return "gemini";
  if (/^(claude|クロード)$/i.test(text.trim())) return "claude";
  return null;
}
```

日本語でも切り替えられます。「ジェミニ」「クロード」でOK。

### APIの形式差を吸収する

Claude APIとGemini APIは、メッセージの形式が全く違います。

```typescript
// Claude API — role/content形式
{
  model: "claude-sonnet-4-20250514",
  messages: [
    { role: "user", content: "こんにちは" },
    { role: "assistant", content: "こんにちは！" }
  ]
}

// Gemini API — role(model)/parts形式
{
  contents: [
    { role: "user", parts: [{ text: "こんにちは" }] },
    { role: "model", parts: [{ text: "こんにちは！" }] }
  ]
}
```

これを抽象化レイヤーで隠蔽しています。

```typescript
async function askAI(conversation: Message[], model: ModelType): Promise<string> {
  if (model === "gemini") return askGemini(conversation);
  return askClaude(conversation);
}
```

内部的にはそれぞれのAPI仕様に合わせて変換しますが、呼び出し側は1行で済みます。

Geminiは無料枠（`gemini-2.5-flash`）があるので、普段はGeminiで会話して、精度が必要な時だけClaudeに切り替える運用をしています。

## PC同期スクリプト — マーカーベースの部分置換

5分おきにGistの内容を取得して、`~/.claude/CLAUDE.md` に書き込むスクリプトです。Windows（PowerShell）とUnix（Bash）の両方を用意しています。

### 鍵となる設計：HTMLコメントマーカー

CLAUDE.mdにはLINEログ以外のコンテンツ（開発ルールなど）も書いてあります。全体を上書きするわけにはいかないので、**マーカーで囲んだ範囲だけを置換**します。

```markdown
# 共通開発ルール（全プロジェクト適用）        ← これは保護される
...

<!-- MOBILE_SYNC_START -->                     ← ここから
# Mobile Sync

**[2026/3/15 21:20:26]**
**ユーザー:** 印象派さんぽのボットに画像が表示されないです。

**Gemini:** 承知しました！CLI Claudeさんに伝えておきますね！
<!-- MOBILE_SYNC_END -->                       ← ここまで置換
```

HTMLコメントなのでMarkdownの表示に影響せず、正規表現で安全に検出・置換できます。

### PowerShell版の実装

```powershell
$MarkerStart = "<!-- MOBILE_SYNC_START -->"
$MarkerEnd = "<!-- MOBILE_SYNC_END -->"

# Gistから取得
$response = Invoke-RestMethod -Uri "https://api.github.com/gists/$GistId" -Headers $headers
$memo = $response.files.'claude-memo.md'.content

# マーカー間を置換（なければ末尾に追加）
$syncBlock = "$MarkerStart`n# Mobile Sync`n`n$memo`n$MarkerEnd"
if ($content -match [regex]::Escape($MarkerStart)) {
    $pattern = '(?s)' + [regex]::Escape($MarkerStart) + '.*?' + [regex]::Escape($MarkerEnd)
    $content = [regex]::Replace($content, $pattern, $syncBlock)
} else {
    $content = $content.TrimEnd() + "`n`n" + $syncBlock + "`n"
}

# UTF-8 BOMなしで保存（重要）
$utf8NoBom = New-Object System.Text.UTF8Encoding($false)
[System.IO.File]::WriteAllText($ClaudeMd, $content, $utf8NoBom)
```

PowerShellのデフォルトはBOM付きUTF-8ですが、Claude CodeはBOMなしを期待するため、`.NET`の`UTF8Encoding($false)`を直接使っています。

### 5分間隔の自動実行

Windowsタスクスケジューラに登録するスクリプトも用意しています。

```powershell
$trigger = New-ScheduledTaskTrigger -AtLogOn
$trigger.Repetition = (New-ScheduledTaskTrigger -Once -At "00:00" `
    -RepetitionInterval (New-TimeSpan -Minutes 5)).Repetition

$settings = New-ScheduledTaskSettingsSet `
    -AllowStartIfOnBatteries `      # バッテリー駆動でも実行
    -DontStopIfGoingOnBatteries `   # バッテリーに切り替わっても停止しない
    -StartWhenAvailable             # PC復帰時に実行
```

Unix/Linuxなら `crontab -e` で `*/5 * * * * /path/to/sync-gist.sh` を追加するだけです。

## 実際の使い方

### 移動中にLINEでアイデアを伝える

```
私：印象派さんぽのボットとトップ画面に画像が表示されないです。
    CLIに伝えてください。

Gemini：承知しました！CLI Claudeさんへのメモとして控えておきますね。
        「ボットとトップ画面に画像が表示されない」問題を
        CLI Claudeさんに伝えます！
```

### PCに戻ってClaude Codeを開く

Claude Codeは `~/.claude/CLAUDE.md` を自動的に読み込むので、何も説明しなくても「さっきLINEで話してたこと」を知っています。

```
私：さっきLINEで伝えた画像の件、直して。

Claude Code：CLAUDE.mdを確認しました。ボットとトップ画面の
             画像表示の問題ですね。調査します。
```

**LINEでの会話がそのままClaude Codeへの指示書になる**のが、このシステムの本質です。

## コスト

| サービス | 費用 |
|---------|------|
| Vercel（ホスティング） | 無料（Hobby） |
| GitHub Gist（ストレージ） | 無料 |
| Gemini API（AI応答） | 無料（2.5 Flash） |
| LINE Messaging API | 無料（月200通まで） |
| Claude API（必要時のみ） | 従量課金 |

**普段はGemini（無料）でメモ・相談して、精度が必要な時だけClaudeに切り替える。** ほぼ無料で運用できています。

## まとめ

| 構成要素 | 技術 | 行数 |
|---------|------|------|
| Webhook + AI応答 | Next.js API Route | 232行 |
| PC同期（Windows） | PowerShell | 61行 |
| PC同期（Unix） | Bash | 66行 |
| タスク登録 | PowerShell | 35行 |

全部合わせて400行弱です。DBなし、インフラ管理なし、ほぼ無料。

「スマホで思いついたことをPCの開発環境に届ける」という問題に対して、GitHub Gistをハブにした非同期同期という解を選びました。リアルタイム性は5分遅れますが、「PCに戻った時に会話が届いている」という用途には十分です。

移動時間が多い開発者にとって、スマホとPCの間にある「伝えたかったけど忘れた」という溝を埋めるツールになっています。

---

### ソースコード

実装の全体は[GitHub](https://github.com/keita2399/line-claude-sync)で公開しています。LINE BotとGistの準備ができれば、自分用の同期環境を構築できます。
