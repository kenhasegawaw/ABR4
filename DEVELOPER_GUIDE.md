# 開発者向け技術手順書

## プロジェクト構造

```
barcode-scanner-js-main/
├── index.html          # メインHTML（UI構造定義）
├── app.js             # メインJavaScript（アプリケーションロジック）
├── styles.css         # スタイルシート（レスポンシブ対応）
├── README.md          # ユーザー向け手順書
├── LICENSE            # ライセンス情報
└── docs/
    └── barcode-scanner-research.md  # 技術調査・設計ドキュメント
```

## アーキテクチャ概要

### 検出戦略
1. **プログレッシブ検出**: BarcodeDetector API → ZXing フォールバック
2. **マルチバーコード対応**: 同時複数検出・処理
3. **リアルタイム処理**: 250ms間隔での検出ループ

### コア機能

#### 1. カメラ管理 (`app.js`)
```javascript
// 主要な関数・変数
let mediaStream = null;
let activeCameraDeviceId = null;

async function enumerateCameras()      // カメラデバイス一覧取得
async function startCamera(deviceId)  // カメラストリーム開始
function stopCamera()                 // カメラストリーム停止
```

#### 2. 検出エンジン
```javascript
// 検出器の管理
const detectorCache = new Map();
const detectorAvailability = new Map();

// 検出アルゴリズム定義
const DETECTOR_DEFINITIONS = [
  { id: 'native', label: 'BarcodeDetector' },
  { id: 'zxing', label: 'ZXing' },
];
```

#### 3. 結果管理
```javascript
// 検出結果の保持・重複排除
let lastResults = new Map();
const RESULT_TTL_MS = 8000;  // 結果保持時間

// 結果の形式
// { value: string, format: string, timestamp: number, rect: DOMRect }
```

## 開発・カスタマイズ手順

### 1. 開発環境セットアップ

#### 必要な環境
- Modern Web Browser (Chrome推奨)
- ローカルHTTPサーバー
- テキストエディタ/IDE (VS Code推奨)

#### 推奨拡張機能（VS Code）
```json
{
  "recommendations": [
    "ms-vscode.vscode-json",
    "esbenp.prettier-vscode",
    "ritwickdey.liveserver",
    "ms-vscode.vscode-typescript-next"
  ]
}
```

### 2. コードの主要セクション

#### メディアストリーム取得
```javascript
// getUserMedia制約の設定
const constraints = {
  video: {
    deviceId: deviceId ? { exact: deviceId } : undefined,
    facingMode: deviceId ? undefined : 'environment',
    width: { ideal: 1920 },
    height: { ideal: 1080 }
  }
};
```

#### 検出ループ制御
```javascript
// 検出間隔の制御
const DETECTION_INTERVAL_MS = 250;
let detectionTimer = null;

function startDetection() {
  detectionTimer = setInterval(async () => {
    await detectBarcodes();
  }, DETECTION_INTERVAL_MS);
}
```

#### オーバーレイ描画
```javascript
// Canvas描画の座標変換
function getOverlayMetrics() {
  const videoWidth = videoEl.videoWidth;
  const videoHeight = videoEl.videoHeight;
  const overlayWidth = overlayEl.clientWidth;
  const overlayHeight = overlayEl.clientHeight;
  
  // アスペクト比を保持した座標変換
  const scale = Math.max(overlayWidth / videoWidth, overlayHeight / videoHeight);
  // ...
}
```

### 3. カスタマイズポイント

#### UI/UXの変更
- `styles.css`: デザイン・レイアウト調整
- `index.html`: UI要素の追加・変更
- CSS変数で色テーマを簡単に変更可能

#### 検出パラメータの調整
```javascript
// app.js 内の定数
const DETECTION_INTERVAL_MS = 250;  // 検出頻度（ms）
const RESULT_TTL_MS = 8000;         // 結果表示時間（ms）
```

#### 対応バーコード形式の変更
```javascript
// ZXing使用時のフォーマット指定
const formats = [
  ZXing.BarcodeFormat.QR_CODE,
  ZXing.BarcodeFormat.CODE_128,
  // 追加したい形式をここに
];
```

### 4. デバッグ・テスト手順

#### ログ出力の有効化
```javascript
// デバッグ用ログ関数を追加
function debugLog(message, data = null) {
  if (window.DEBUG) {
    console.log(`[BarcodeScanner] ${message}`, data);
  }
}

// 使用例
window.DEBUG = true;  // コンソールで有効化
```

#### テスト用バーコード
- オンラインQRコードジェネレーター使用
- 印刷したテストバーコードを準備
- 画面上のバーコード画像でもテスト可能

#### パフォーマンス測定
```javascript
// 検出処理時間の測定
async function detectBarcodes() {
  const startTime = performance.now();
  // ... 検出処理
  const endTime = performance.now();
  console.log(`Detection took ${endTime - startTime} ms`);
}
```

### 5. トラブルシューティング

#### よくある開発時の問題

**問題**: カメラアクセスエラー
```javascript
// エラーハンドリングの改善
navigator.mediaDevices.getUserMedia(constraints)
  .catch(error => {
    console.error('Camera access error:', error);
    if (error.name === 'NotAllowedError') {
      // 許可拒否時の処理
    } else if (error.name === 'NotFoundError') {
      // カメラ未検出時の処理
    }
  });
```

**問題**: ZXingライブラリの読み込みエラー
```javascript
// 動的インポートのエラーハンドリング
try {
  const ZXing = await import('https://unpkg.com/@zxing/library@latest/esm/index.js');
  // ZXing使用可能
} catch (error) {
  console.error('ZXing library failed to load:', error);
  // フォールバック処理
}
```

### 6. ビルド・デプロイ

#### 本番環境への配置
1. **静的ファイルとして配置**
   - 全ファイルをWebサーバーにアップロード
   - HTTPS環境必須（カメラアクセスのため）

2. **CDN使用時の注意**
   - ZXingライブラリのバージョン固定推奨
   - CORS設定の確認

#### セキュリティ考慮事項
- Content Security Policy (CSP) の設定
- カメラアクセス権限の適切な管理
- 機密データの漏洩防止

### 7. 拡張機能の実装例

#### 検出結果のエクスポート機能
```javascript
function exportResults() {
  const results = Array.from(lastResults.values());
  const csvContent = results.map(r => `${r.value},${r.format}`).join('\n');
  
  const blob = new Blob([csvContent], { type: 'text/csv' });
  const url = URL.createObjectURL(blob);
  
  const a = document.createElement('a');
  a.href = url;
  a.download = 'barcode-results.csv';
  a.click();
}
```

#### 音声フィードバック
```javascript
function playBeep() {
  const audioContext = new AudioContext();
  const oscillator = audioContext.createOscillator();
  oscillator.frequency.setValueAtTime(800, audioContext.currentTime);
  oscillator.connect(audioContext.destination);
  oscillator.start();
  oscillator.stop(audioContext.currentTime + 0.1);
}
```

## API リファレンス

### 主要関数

#### `startCamera(deviceId: string): Promise<void>`
指定されたカメラデバイスでストリームを開始

#### `stopCamera(): void`
現在のカメラストリームを停止

#### `detectBarcodes(): Promise<void>`
現在のフレームでバーコード検出を実行

#### `clearResults(): void`
検出結果をクリア

### イベントハンドラ

#### カメラ関連
- カメラ開始/停止
- デバイス切り替え

#### 検出関連
- 検出開始/停止
- アルゴリズム切り替え

#### UI関連
- 結果クリア
- 設定変更

## ライセンス・依存関係

### 外部ライブラリ
- **ZXing-js**: Apache License 2.0
- **Web APIs**: ブラウザ標準（無料）

### ブラウザ互換性
- Chrome 88+: ✅ 完全対応
- Edge 88+: ✅ 完全対応  
- Firefox: ⚠️ ZXingフォールバック
- Safari: ⚠️ 部分対応

---

このドキュメントは技術者向けの詳細な実装ガイドです。ユーザー向けの基本的な使用方法については `README.md` を参照してください。