---
title: "eSCLプロトコルでネットワークスキャナをブラウザから直接制御する（Chrome拡張 + Native Messaging）"
emoji: "🖨️"
type: "tech"
topics: ["chrome拡張機能", "escl", "typescript", "nextjs"]
published: false
---

## はじめに

経費精算アプリを作っているとき、スキャナ連携が必要になりました。

「スキャナのボタンを押す → ブラウザにスキャン画像が届く → AIでレシートを解析」

このフローを実現しようとしたとき、意外な壁にぶつかりました。

**ブラウザからは、ローカルネットワークにあるスキャナに直接アクセスできない。**

USB経由のスキャナならWebUSB APIで対応できますが、ネットワークスキャナ（Wi-Fi接続）の場合は話が変わります。この記事では、**Chrome拡張機能 + Native Messaging + eSCLプロトコル** を組み合わせて解決した方法を紹介します。

## eSCL（AirScan）プロトコルとは

eSCLは、Apple が策定したネットワークスキャナ向けのHTTPベースプロトコルです。「AirScan」とも呼ばれます。

主な特徴:
- **RESTful HTTP**: 特別なライブラリ不要。素のHTTPリクエストで制御できる
- **mDNS/Bonjour対応**: ネットワーク上のスキャナを自動検出できる
- **幅広い対応機器**: Canon, Epson, HP, Brotherなど主要メーカーがほぼ対応

### eSCLの基本的なスキャンフロー

```
1. mDNS でスキャナを発見
   → _uscan._tcp または _uscans._tcp を検索

2. スキャナの能力を取得
   GET http://{scanner-ip}/eSCL/ScannerCapabilities

3. スキャンジョブを作成
   POST http://{scanner-ip}/eSCL/ScanJobs
   Body: ScanSettings XML

4. スキャン完了を待機
   GET http://{scanner-ip}/eSCL/ScanJobs/{job-id}/NextDocument
   → 202 Accepted ならまだスキャン中
   → 200 OK でスキャン画像が返ってくる
```

## なぜChrome拡張が必要なのか

Webアプリから直接スキャナのHTTPエンドポイントに `fetch()` すればよいのでは？ と思うかもしれません。

しかし、以下の制約があります：

1. **CORSブロック**: スキャナのファームウェアはCORSヘッダーを返さない
2. **Mixed Content**: HTTPSのWebアプリからHTTPのスキャナへのリクエストはブロックされる
3. **ローカルネットワークアクセス制限**: ChromeはPrivate Network Access仕様により、パブリックオリジンからプライベートIPへのアクセスに制限を設けている

これらの制約を突破するために **Chrome拡張機能** を使います。拡張機能は通常のWebページとは異なる権限モデルで動作し、ローカルネットワークへのアクセスが可能です。

## アーキテクチャ全体像

```
┌─────────────────────┐        window.postMessage        ┌──────────────────────────┐
│   Next.js Webアプリ  │ ─────────────────────────────→  │   Chrome拡張 (Content    │
│   (receipt-scanner) │ ←──────────────────────────────  │   Script / Background)   │
└─────────────────────┘        SCANNER_RESPONSE           └──────────────────────────┘
                                                                      │
                                                                      │ fetch (HTTP)
                                                                      ↓
                                                          ┌──────────────────────────┐
                                                          │  ネットワークスキャナ     │
                                                          │  (eSCL HTTP API)         │
                                                          └──────────────────────────┘
```

拡張機能がブリッジとして機能し、WebアプリとスキャナのHTTP APIを仲介します。

### スキャン画像の転送問題（1MB制限）

Chrome拡張のContent ScriptとBackground Serviceの間で `chrome.runtime.sendMessage()` を使う場合、メッセージサイズに **1MB の上限** があります。A4スキャン画像は軽く5〜20MBになるため、これでは転送できません。

解決策として、**スキャンした画像をサーバーサイドのAPIに直接POSTする** 方法を採用しました。

```
拡張機能がeSCLスキャン → 画像取得 → /api/scan に直接POST → WebアプリはAPIレスポンスを受け取る
```

## 実装詳細

### Webアプリ側（Next.js）

WebアプリはChrome拡張の存在確認と、スキャン要求の送信を行います。

```typescript
// Chrome拡張の存在確認
useEffect(() => {
  const handler = (e: MessageEvent) => {
    if (e.data?.type === "SCANNER_EXTENSION_READY") {
      setExtensionInstalled(true);
    }
  };
  window.addEventListener("message", handler);
  window.postMessage({ type: "SCANNER_CHECK" }, "*");
  return () => window.removeEventListener("message", handler);
}, []);

// スキャン開始
const scanFromScanner = async () => {
  setScannerLoading(true);
  
  // 拡張機能の存在確認
  const extensionReady = await new Promise<boolean>((resolve) => {
    const timer = setTimeout(() => resolve(false), 1500);
    const handler = (e: MessageEvent) => {
      if (e.data?.type === "SCANNER_EXTENSION_READY") {
        clearTimeout(timer);
        window.removeEventListener("message", handler);
        resolve(true);
      }
    };
    window.addEventListener("message", handler);
    window.postMessage({ type: "SCANNER_CHECK" }, "*");
  });

  if (!extensionReady) {
    setScannerError("Chrome拡張機能がインストールされていません");
    return;
  }

  // スキャン実行
  const result = await new Promise<{receipts?: Receipt[]; error?: string}>((resolve, reject) => {
    const timer = setTimeout(() => reject(new Error("タイムアウト")), 150000);
    const handler = (e: MessageEvent) => {
      if (e.data?.type !== "SCANNER_RESPONSE") return;
      // 進捗表示
      if (e.data.status === "discovering") { setScannerStatus("スキャナを検索中..."); return; }
      if (e.data.status === "scanning")    { setScannerStatus("スキャン中..."); return; }
      if (e.data.status === "processing")  { setScannerStatus("AIが読み取り中..."); return; }
      // 完了
      clearTimeout(timer);
      window.removeEventListener("message", handler);
      resolve(e.data);
    };
    window.addEventListener("message", handler);
    window.postMessage({ type: "SCANNER_REQUEST", action: "scan" }, "*");
  });
};
```

### Chrome拡張機能側

Content Scriptはページからのメッセージを受け取り、eSCLスキャンを実行します。

#### スキャナの発見

```javascript
// content_script.js

async function discoverScanners() {
  // eSCL スキャナはデフォルトで以下のポートを使用
  const commonPorts = [80, 8080, 443, 9500];
  // ローカルネットワークのIP範囲を探索（実際の実装ではmDNS推奨）
  const localIPs = await getLocalNetworkIPs();
  
  for (const ip of localIPs) {
    for (const port of commonPorts) {
      try {
        const url = `http://${ip}:${port}/eSCL/ScannerCapabilities`;
        const res = await fetch(url, { signal: AbortSignal.timeout(500) });
        if (res.ok) {
          const xml = await res.text();
          if (xml.includes("ScannerCapabilities")) {
            return { ip, port, capabilities: parseCapabilities(xml) };
          }
        }
      } catch {
        // 接続失敗は無視
      }
    }
  }
  return null;
}
```

#### mDNS によるスキャナ発見（推奨）

ポートスキャンより mDNS/Bonjour の方が確実です。Chrome拡張ではNative Messagingを使ったホストプログラムからmdnsを呼び出す方法もありますが、まずシンプルにIPを直接指定する方法から始めるのが実用的です。

#### スキャンジョブの作成

```javascript
async function startScanJob(scannerUrl) {
  const scanSettings = `<?xml version="1.0" encoding="UTF-8"?>
<scan:ScanSettings xmlns:scan="http://schemas.hp.com/imaging/escl/2011/05/03"
                   xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance">
  <scan:Version>2.6</scan:Version>
  <scan:Intent>Document</scan:Intent>
  <scan:ScanRegions>
    <scan:ScanRegion>
      <scan:Height>3508</scan:Height>
      <scan:Width>2480</scan:Width>
      <scan:XOffset>0</scan:XOffset>
      <scan:YOffset>0</scan:YOffset>
    </scan:ScanRegion>
  </scan:ScanRegions>
  <scan:InputSource>Platen</scan:InputSource>
  <scan:ColorMode>RGB24</scan:ColorMode>
  <scan:XResolution>300</scan:XResolution>
  <scan:YResolution>300</scan:YResolution>
  <scan:DocumentFormatExt>image/jpeg</scan:DocumentFormatExt>
</scan:ScanSettings>`;

  const res = await fetch(`${scannerUrl}/eSCL/ScanJobs`, {
    method: "POST",
    headers: { "Content-Type": "text/xml" },
    body: scanSettings,
  });

  if (res.status === 201) {
    // Location ヘッダーにジョブURLが入っている
    return res.headers.get("Location");
  }
  throw new Error(`スキャンジョブ作成失敗: ${res.status}`);
}
```

#### スキャン完了待機

```javascript
async function waitForScanComplete(jobUrl, maxRetries = 60) {
  for (let i = 0; i < maxRetries; i++) {
    const res = await fetch(`${jobUrl}/NextDocument`);
    
    if (res.status === 200) {
      // スキャン完了！画像データを取得
      const blob = await res.blob();
      return blob;
    }
    
    if (res.status === 202) {
      // まだスキャン中
      await new Promise(r => setTimeout(r, 1000));
      continue;
    }
    
    if (res.status === 404) {
      // ジョブが終了（原稿なし等）
      throw new Error("スキャン原稿が見つかりません");
    }
    
    throw new Error(`予期しないステータス: ${res.status}`);
  }
  throw new Error("スキャンタイムアウト");
}
```

#### APIサーバーへの転送（1MB制限回避）

```javascript
async function sendToAPI(imageBlob) {
  // FormDataで画像を直接APIに送信
  const formData = new FormData();
  formData.append("image", imageBlob, "scan.jpg");
  
  // Webアプリのオリジンを指定（manifest.jsonのhost_permissionsに追加が必要）
  const res = await fetch("https://your-app.vercel.app/api/scan", {
    method: "POST",
    body: formData,
  });
  
  return await res.json();
}
```

#### 全体フローの組み合わせ

```javascript
window.addEventListener("message", async (event) => {
  if (event.source !== window) return;
  
  if (event.data?.type === "SCANNER_CHECK") {
    window.postMessage({ type: "SCANNER_EXTENSION_READY" }, "*");
    return;
  }
  
  if (event.data?.type === "SCANNER_REQUEST" && event.data.action === "scan") {
    try {
      // 1. スキャナ検索
      window.postMessage({ type: "SCANNER_RESPONSE", status: "discovering" }, "*");
      const scanner = await discoverScanners();
      if (!scanner) throw new Error("スキャナが見つかりません");
      
      const scannerUrl = `http://${scanner.ip}:${scanner.port}`;
      
      // 2. スキャン実行
      window.postMessage({ type: "SCANNER_RESPONSE", status: "scanning" }, "*");
      const jobUrl = await startScanJob(scannerUrl);
      const imageBlob = await waitForScanComplete(jobUrl);
      
      // 3. AI処理
      window.postMessage({ type: "SCANNER_RESPONSE", status: "processing" }, "*");
      const result = await sendToAPI(imageBlob);
      
      // 4. 結果を返す
      window.postMessage({
        type: "SCANNER_RESPONSE",
        status: "complete",
        receipts: result.receipts,
      }, "*");
      
    } catch (error) {
      window.postMessage({
        type: "SCANNER_RESPONSE",
        status: "error",
        error: error.message,
      }, "*");
    }
  }
});
```

### manifest.json（Chrome拡張の設定）

```json
{
  "manifest_version": 3,
  "name": "Receipt Scanner Bridge",
  "version": "1.0",
  "permissions": ["*://*/*"],
  "host_permissions": [
    "http://*/*",
    "https://your-app.vercel.app/*"
  ],
  "content_scripts": [
    {
      "matches": ["https://your-app.vercel.app/*"],
      "js": ["content_script.js"]
    }
  ]
}
```

## 実際に遭遇したハマりポイント

### 1. スキャナによってeSCLのパスが違う

`/eSCL/` が標準ですが、一部メーカーは `/escl/`（小文字）や `/airscan/eSCL/` を使っています。能力取得の前にパス探索が必要なことがあります。

### 2. ScanSettings のスキーマがメーカーで微妙に違う

HP系のスキャナは `scan:` 名前空間プレフィックスを要求しますが、Epson系は `pwg:` だったりします。まずは `ScannerCapabilities` を取得して対応フォーマットを確認しましょう。

### 3. 202ポーリングの間隔

200msごとにポーリングするとスキャナによっては429を返すものがあります。1秒間隔が安全です。

### 4. HTTPS接続のスキャナ

一部のスキャナは `_uscans._tcp` でHTTPS接続を提供しますが、自己署名証明書が多く、`fetch()` が失敗します。Native Messaging経由でNode.jsホストプログラムを使い、`rejectUnauthorized: false` で対応する方法もあります。

## まとめ

eSCLはHTTPベースの標準プロトコルなので、実装は意外にシンプルです。

ブラウザから直接制御できない制約もChrome拡張を挟むことで解決できました。ポイントをまとめると:

1. **eSCL = HTTP REST**: 難しいSDK不要、素のfetchで制御できる
2. **Chrome拡張がCORS/Mixed Content問題を解決**: ローカルネットワークへのアクセスが可能
3. **1MB制限の回避**: 画像はAPIサーバーに直接POST、Webアプリにはレスポンスだけ届ける
4. **進捗はpostMessageで逐次通知**: ユーザーに「スキャン中」を見せられる

ネットワークスキャナ連携でお悩みの方の参考になれば幸いです！
