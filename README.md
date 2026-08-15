# 光あわせ — 露出調整

ブラウザだけで動く露出調整ツール。JPEG/HEIC と、RAW（ARW・CR2・NEF・DNG ほか）の埋め込みプレビューを読み込み、露出・ハイライト・シャドウ・コントラスト・彩度・色温度を調整して JPEG で書き出す。

画像処理はすべて端末内で完結し、サーバーには何も送信しない。バックエンドなし、依存パッケージなし、ビルド不要。

## 構成

```
index.html              アプリ本体（HTML/CSS/JS すべて内包）
manifest.webmanifest    PWA 設定（ホーム画面に追加できる）
sw.js                   Service Worker（オフライン動作）
_headers                Cloudflare Pages 用のヘッダ設定
icons/                  アイコン一式
```

## デプロイ（Cloudflare Pages）

### Wrangler で直接

```bash
npx wrangler pages deploy . --project-name hikari-awase
```

初回はプロジェクトが作られる。以降は同じコマンドで更新。

### GitHub 連携

1. このディレクトリをリポジトリにして push
2. Cloudflare ダッシュボード → Workers & Pages → Create → Pages → Connect to Git
3. ビルド設定:
   - Framework preset: **None**
   - Build command: **空欄**
   - Build output directory: **/**

## 更新するとき

Service Worker がキャッシュを持つので、`index.html` を書き換えたら `sw.js` の版数を必ず上げる。

```js
const CACHE = 'hikari-awase-v1';   // → v2, v3 …
```

上げ忘れると、既に開いたことのある端末に古い画面が残り続ける。

## 動作の要点

- **露出はリニア光で処理**：sRGB → linear → `×2^EV` → sRGB。カメラの露出補正と同じ挙動になる
- **LUT 方式**：色温度・露出・ハイライト/シャドウ・コントラストを 256 段の LUT 3 本に畳み込み、1 ピクセルにつき 3 回引くだけ。彩度のみピクセル単位で輝度と混合
- **RAW**：ファイル内の JPEG ブロック（FFD8〜FFD9）を全部拾い、最大のものを埋め込みプレビューとして採用。形式に依存しない
- **向き**：プレビューに EXIF がなければ RAW の TIFF ヘッダから Orientation を読んで回転
- **書き出し**：フル解像度 JPEG（品質 92）。iOS Safari の canvas 面積上限 16.7MP を超える場合のみ自動縮小

## 制約

RAW は 8bit の埋め込みプレビューを読んでいるので、RAW 本来の階調余裕は使えない。白飛びを実際に引き戻す作業は DaVinci Resolve や darktable で。現場で露出の当たりを速く確認する用途を想定している。

## 操作

| 操作 | 動作 |
| --- | --- |
| ドラッグ&ドロップ / ペースト | 画像を開く |
| 画像を押している間 | 元画像と比較 |
| スライダーをダブルクリック | その項目だけリセット |
| 自動 | 平均輝度 18% を目標に露出を当て、ハイライト/シャドウを補助的に調整 |
| クリップ表示 | 白飛びを赤、黒つぶれを青で表示 |
| 背景 | グレー / ホワイト / ブラック（露出判断はグレーが基準） |
