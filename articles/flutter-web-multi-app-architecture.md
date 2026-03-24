---
title: "1リポジトリで6つのアートアプリを生み出すFlutter Web設計術"
emoji: "🎨"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Flutter", "Dart", "GLSL", "設計", "LINE"]
published: true
---

## はじめに

[前回の記事](https://zenn.dev/keita2399/articles/ai-collaboration-veteran-engineer)で、AI協働開発で2週間に複数のアプリを作った話を書きました。

その中の一つ「アートさんぽシリーズ」は、世界の美術館APIを使ってスマホで名画を楽しめるFlutter Webアプリです。メトロポリタン美術館、シカゴ美術館、クリーブランド美術館、スミソニアン博物館、さらにフェルメールやレンブラントといった画家シリーズまで、**1つのリポジトリから6つのアプリ**をビルドしています。

| カテゴリ | アプリ名 | API |
|---------|---------|-----|
| 美術館さんぽ | メトロポリタン美術館さんぽ | Met Museum API |
| 美術館さんぽ | クリーブランド美術館さんぽ | Cleveland Museum API |
| 美術館さんぽ | スミソニアン博物館さんぽ | Smithsonian API |
| 特集さんぽ | 印象派さんぽ | Art Institute of Chicago API |
| 画家さんぽ | フェルメールさんぽ | Wikidata SPARQL |
| 画家さんぽ | レンブラントさんぽ | Wikidata SPARQL |

この記事では、「なぜ1リポジトリで複数アプリなのか」「どう設計したのか」「新シリーズの追加がなぜ3ファイルで済むのか」を、実際のコードとともに解説します。

## なぜ1リポジトリ・マルチアプリなのか

最初に作ったのはメトロポリタン美術館さんぽでした。次に印象派さんぽを作るとき、2つの選択肢がありました。

1. **リポジトリを分ける** — UIもロジックも同じなのにコードが重複する
2. **1リポジトリで切り替える** — 設定とAPI実装だけ差し替えれば済む

ギャラリー画面、作品詳細、お気に入り、クイズ、ガチャ——UI層は全アプリで共通です。違うのは「どの美術館のAPIを叩くか」と「テーマカラーやカテゴリの設定」だけ。

であれば、**設定とAPI実装を差し替え可能にして、1リポジトリから複数アプリをビルドする**のが自然な設計です。

## 設計の全体像

アーキテクチャは3つの層で構成されています。

```
┌──────────────────────────────────────────────┐
│  エントリポイント (main_met.dart 等)         │  ← アプリごとに1つ（3行）
├──────────────────────────────────────────────┤
│  AppConfig (met_config.dart 等)              │  ← 設定（テーマ色、カテゴリ等）
│  ArtApi (met_api.dart 等)                    │  ← API実装（データ取得・変換）
├──────────────────────────────────────────────┤
│  共通UI層 (screens/, widgets/, models/)      │  ← 全アプリ共通
│  共通サービス (翻訳, 色彩分析, TTS 等)       │
└──────────────────────────────────────────────┘
```

## AppConfig — アプリの「個性」を定義する

各アプリの違いは、すべて `AppConfig` クラスに集約されています。

```dart
/// アプリ設定の基底クラス
class AppConfig {
  final String appName;        // 「メトロポリタン美術館さんぽ」
  final String appNameEn;      // 「Metropolitan Museum Walk」
  final Color themeColor;      // テーマカラー
  final String appUrl;         // デプロイ先URL
  final List<FilterCategory> filterCategories;  // ギャラリーのフィルター
  final bool hasTimeline;      // 年表タブの有無
  final bool hasColorPalette;  // 色彩パレット表示の有無
  final String artworkLabel;   // 「名画」or「展示品」
  // ...
}

/// グローバルにアクセスできる現在の設定
late final AppConfig appConfig;
```

たとえば、メトロポリタン美術館さんぽの設定はこうです。

```dart
const metConfig = AppConfig(
  appName: 'メトロポリタン美術館さんぽ',
  appNameEn: 'Metropolitan Museum Walk',
  themeColor: Color(0xFF8B0000),  // 赤
  appUrl: 'https://sanpo-met.vercel.app',
  filterCategories: [
    FilterCategory(label: 'すべて'),
    FilterCategory(label: '絵画 (Paintings)', query: 'painting'),
    FilterCategory(label: '彫刻 (Sculpture)', query: 'sculpture'),
    FilterCategory(label: '日本美術 (Japanese)', query: 'japanese'),
    // ...
  ],
);
```

スミソニアン博物館は美術品以外（恐竜の化石、宇宙船など）も含むため、色彩パレットを非表示にしています。

```dart
const smithsonianConfig = AppConfig(
  appName: 'スミソニアン博物館さんぽ',
  themeColor: Color(0xFF1B5E20),  // 深緑
  hasColorPalette: false,          // 色彩分析は非表示
  artworkLabel: '展示品',          // 「名画」ではなく「展示品」
  filterCategories: [
    FilterCategory(label: 'すべて'),
    FilterCategory(label: '恐竜・化石', query: 'dinosaur fossil'),
    FilterCategory(label: '宝石・鉱物', query: 'gem mineral'),
    FilterCategory(label: '航空・宇宙', query: 'aerospace'),
    // ...
  ],
);
```

UI側は `appConfig` を参照するだけなので、美術館の違いを意識する必要がありません。

```dart
// テーマカラーの適用（共通コード）
ThemeData(
  colorScheme: ColorScheme.dark(primary: appConfig.themeColor),
)

// 色彩パレットの表示制御（共通コード）
if (appConfig.hasColorPalette) ColorPaletteWidget(artwork: artwork),
```

## ArtApi — データ取得を抽象化する

美術館ごとにAPIの仕様はまったく異なります。しかし、アプリが必要とするデータは同じです。「作品一覧を取得する」「作品詳細を取得する」——この共通インターフェースを `ArtApi` 抽象クラスで定義しています。

```dart
/// 美術館API共通インターフェース
abstract class ArtApi {
  Map<String, String> get imageHeaders;
  Future<List<Artwork>> fetchHighlights({String? query, int limit = 20});
  Future<Artwork?> fetchArtworkDetail(int id);
  Future<List<Artwork>> fetchPublicDomainWorks({String? query, int limit = 100});
}

/// グローバルにアクセスできるAPIインスタンス
late final ArtApi artApi;
```

各美術館はこのインターフェースを実装します。MetApi、AicApi、ClevelandApi、SmithsonianApi——それぞれ内部のAPI呼び出しは全く異なりますが、返すデータ型は同じ `Artwork` です。

### 画家シリーズの効率的な実装

面白いのは画家シリーズ（フェルメール、レンブラント）です。Wikidata SPARQLで特定画家の全作品を取得する `WikidataArtistApi` を作り、画家ごとのサブクラスは**たった数行**で完結します。

```dart
/// フェルメール全作品API
class VermeerApi extends WikidataArtistApi {
  VermeerApi()
      : super(
          artistQid: 'Q41264',       // WikidataでのフェルメールのID
          artistNameJa: 'ヨハネス・フェルメール',
          artistCountry: 'オランダ',
          filters: {
            '1650s': (Artwork w) => w.date.startsWith('165'),
            '1660s': (Artwork w) => w.date.startsWith('166'),
            '1670s': (Artwork w) => w.date.startsWith('167'),
          },
        );
}
```

WikidataのIDと画家名を指定するだけ。フィルター条件も年代でシンプルに定義しています。新しい画家を追加するなら、このファイルをコピーしてIDと名前を変えるだけです。

## エントリポイント — たった3行

各アプリのエントリポイントは3行です。設定とAPIを差し込んで、共通の `startApp()` を呼ぶだけ。

```dart
// main_met.dart（メトロポリタン美術館さんぽ）
void main() {
  appConfig = metConfig;
  artApi = MetApi();
  startApp();
}
```

```dart
// main_vermeer.dart（フェルメールさんぽ）
void main() {
  appConfig = vermeerConfig;
  artApi = VermeerApi();
  startApp();
}
```

DIコンテナやサービスロケータのような仕組みは使っていません。グローバルな `late final` 変数に代入するだけ。アプリの起動時に一度だけ設定されて以後変わらないので、これで十分です。

## ビルドスクリプト — 6アプリを1コマンドで

Flutter Webのビルドは `-t` オプションでエントリポイントを指定できます。これとスプラッシュ画面用のHTMLの切り替えを、シェルスクリプトにまとめました。

```bash
#!/bin/bash
TARGET=${1:-met}

case $TARGET in
  met)
    cp web/index_met.html web/index.html
    flutter build web --release -t lib/main_met.dart
    ;;
  vermeer)
    cp web/index_vermeer.html web/index.html
    flutter build web --release -t lib/main_vermeer.dart
    ;;
  smithsonian)
    cp web/index_smithsonian.html web/index.html
    flutter build web --release -t lib/main_smithsonian.dart
    ;;
  # ... cleveland, aic, rembrandt も同様
esac
```

```bash
# メトロポリタン美術館さんぽをビルド
./build_web.sh met

# フェルメールさんぽをビルド
./build_web.sh vermeer
```

各アプリはVercelの別プロジェクトとしてデプロイしています。ビルド → `vercel deploy` の2ステップで完了します。

## GPUシェーダーで美術館の光を再現する

このシリーズの一番の特徴は、GLSLフラグメントシェーダーによるスポットライト演出です。スマホを傾けるように指でライトを動かすと、絵画の光と影がリアルタイムに変化します。

```glsl
// lighting.frag（抜粋）
float calcLight(vec2 uv, vec2 lightPos, float radius) {
    vec2 dir = uv - lightPos;
    dir.x *= uSize.x / uSize.y;  // アスペクト比補正
    float dist = length(dir);
    return 1.0 - smoothstep(0.0, radius, dist);
}

void main() {
    vec2 uv = FlutterFragCoord().xy / uSize;
    vec4 texColor = texture(uTexture, uv);

    // 光源の減衰を計算
    float att = calcLight(uv, uLight1Pos, uLightRadius);
    float brightness = uAmbient + uLight1Intensity * att * uFlicker;

    // スペキュラハイライト（油絵の具の光沢感）
    float specular = pow(att, 4.0) * 0.15 * uFlicker;

    vec3 litColor = texColor.rgb * brightness * uLightColor + specular * uLightColor;
    fragColor = vec4(litColor, texColor.a);
}
```

さらに、額縁の内側の影も物理的にシミュレーションしています。光源の位置に応じて、額縁が作る影の方向と深さが変わります。

```glsl
// 額縁の影（光源位置に応じた方向性のある影）
vec2 lightOffset = uLight1Pos - vec2(0.5, 0.5);
float leftShadow = pow(smoothstep(0.15, 0.0, uv.x), 1.5) * max(0.0, -lightOffset.x) * 2.5;
float rightShadow = pow(smoothstep(0.85, 1.0, uv.x), 1.5) * max(0.0, lightOffset.x) * 2.5;
```

Flutter Webでカスタムフラグメントシェーダーを使うには、`FragmentProgram` APIでシェーダーをロードし、`CustomPainter` で描画します。Flutterの `ui.Image` としてテクスチャを渡し、GPUで直接レンダリングするため、スマホでも60fpsで滑らかに動きます。

光の色温度も切り替えられます。ろうそくの暖かい光、自然光、蛍光灯——光を変えるだけで、同じ絵画がまったく違う表情を見せます。

## プロキシサーバー — CORS問題の一元解決

美術館APIをブラウザから直接叩くと、CORS（Cross-Origin Resource Sharing）の壁にぶつかります。特に画像CDNはCORSヘッダーを返さないことが多く、`canvas` 要素での描画がブロックされます。

この問題を、Next.jsのAPIルートで作ったプロキシサーバーで一元解決しています。

```
Flutter Web → プロキシサーバー → 各美術館API / 画像CDN
              (impressionist-bot.vercel.app)
```

| エンドポイント | 用途 |
|---------------|------|
| `/api/image?met=URL` | 画像プロキシ（CORSヘッダー付与） |
| `/api/met-proxy?path=...` | Met Museum APIプロキシ |
| `/api/cleveland?...` | Cleveland Museum APIプロキシ |
| `/api/smithsonian?...` | Smithsonian APIプロキシ（APIキー管理） |
| `/api/sparql?query=...` | Wikidata SPARQLプロキシ |
| `/api/tts` | Google Cloud TTS（音声合成） |

Smithsonian APIのようにAPIキーが必要なサービスは、キーをプロキシサーバーの環境変数で管理しています。フロントエンドにAPIキーが露出しない設計です。

## LINEボット — 毎朝6作品を届ける

プロキシサーバーにはLINEボットの機能も載せています。Vercel Cronで毎朝6時に自動実行され、6つの美術館APIから1作品ずつ取得して、LINEのカルーセルメッセージで配信します。

```typescript
// 6作品を並列取得
const [impressionist, met, cleveland, smithsonian, vermeer, rembrandt] =
  await Promise.all([
    fetchImpressionistArtwork(),
    fetchMetArtwork(),
    fetchClevelandArtwork(),
    fetchSmithsonianArtwork(),
    fetchWikidataArtwork('Q41264', 'フェルメール', 'https://sanpo-vermeer.vercel.app/'),
    fetchWikidataArtwork('Q5598', 'レンブラント', 'https://sanpo-rembrandt.vercel.app/'),
  ]);
```

各カードをタップすると、対応するさんぽアプリでその作品を開きます。毎朝のカルーセルがアプリへの導線になっている仕組みです。

## 新シリーズを追加する手順

ここまでの設計により、新しいシリーズの追加は3ファイルで完結します。

```
1. lib/config/xxx_config.dart   ← テーマカラー、カテゴリ等の設定
2. lib/services/xxx_api.dart    ← API実装（既存クラスの継承も可）
3. lib/main_xxx.dart            ← エントリポイント（3行）
```

画家シリーズなら `WikidataArtistApi` を継承して、WikidataのIDを指定するだけです。たとえばゴッホさんぽを追加するなら：

```dart
// lib/services/gogh_api.dart
class GoghApi extends WikidataArtistApi {
  GoghApi() : super(
    artistQid: 'Q5582',
    artistNameJa: 'フィンセント・ファン・ゴッホ',
    artistCountry: 'オランダ',
  );
}
```

```dart
// lib/main_gogh.dart
void main() {
  appConfig = goghConfig;
  artApi = GoghApi();
  startApp();
}
```

あとはビルドスクリプトにケースを追加して、Vercelプロジェクトを作ってデプロイ。所要時間は数十分です。

## 設計判断のまとめ

この設計で意識したのは、**「変わるもの」と「変わらないもの」を分離する**ことです。

| 変わるもの | 変わらないもの |
|-----------|-------------|
| テーマカラー、カテゴリ | ギャラリー画面、詳細画面 |
| API呼び出し、データ変換 | お気に入り、クイズ、ガチャ |
| スプラッシュ画面 | GPUシェーダー演出 |
| 作品の呼称（名画/展示品） | 色彩分析、翻訳、TTS |

DIコンテナのようなフレームワークは導入せず、Dartの `late final` グローバル変数とシンプルな抽象クラスで実現しています。6アプリ程度の規模なら、この素朴なアプローチで十分です。

大事なのは「正しいアーキテクチャ」ではなく、**「この規模に合った設計」**を選ぶことだと思います。

## おわりに

1リポジトリ・マルチアプリの設計により、最初のアプリで苦労した部分（UI、シェーダー、CORS対策）が、2つ目以降はほぼゼロコストで再利用できるようになりました。

現在6アプリを公開していますが、新しい美術館や画家を追加する際の作業量は最小限です。この「拡張のしやすさ」は、最初から狙って設計したわけではなく、2つ目のアプリを作るときに「同じコードをもう一度書きたくない」という自然な動機から生まれたものです。

アートさんぽシリーズは以下のURLで公開しています。スマホでアクセスして、スポットライトの光で名画を眺めてみてください。

| カテゴリ | アプリ | URL |
|---------|-------|-----|
| 美術館さんぽ | メトロポリタン美術館さんぽ | https://sanpo-met.vercel.app |
| 美術館さんぽ | クリーブランド美術館さんぽ | https://sanpo-cleveland.vercel.app |
| 美術館さんぽ | スミソニアン博物館さんぽ | https://sanpo-smithsonian.vercel.app |
| 特集さんぽ | 印象派さんぽ | https://sanpo-impressionist.vercel.app |
| 画家さんぽ | フェルメールさんぽ | https://sanpo-vermeer.vercel.app |
| 画家さんぽ | レンブラントさんぽ | https://sanpo-rembrandt.vercel.app |

### LINEボットで毎朝名画が届きます

友だち追加すると、毎朝6時に6つの美術館から選ばれた作品がカルーセルで届きます。画家名を送ると作品検索もできます。

[![友だち追加](https://scdn.line-apps.com/n/line_add_friends/btn/ja.png)](https://line.me/R/ti/p/@keita2399)
