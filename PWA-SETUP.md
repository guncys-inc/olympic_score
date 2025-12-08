# PWA（Progressive Web App）セットアップ手順

## 必要なファイル

1. ✅ `index.html` - メインアプリ（golf-app.htmlをリネーム）
2. ✅ `manifest.json` - PWA設定ファイル
3. ✅ `sw.js` - Service Worker（オフライン対応）
4. ⚠️ `icon-192.png` - アプリアイコン 192x192
5. ⚠️ `icon-512.png` - アプリアイコン 512x512

## アイコン作成方法

### 方法1: オンラインツールを使用
1. https://www.favicon-generator.org/ にアクセス
2. ⛳ のスクリーンショットをアップロード
3. 192x192 と 512x512 のサイズで生成
4. `icon-192.png` と `icon-512.png` として保存

### 方法2: Canvaを使用
1. Canva で 192x192 の新規デザイン作成
2. 背景を緑色（#16a34a）に設定
3. ⛳ 絵文字を中央に配置
4. PNG形式でダウンロード → `icon-192.png`
5. 同じ手順で 512x512 を作成 → `icon-512.png`

### 方法3: 絵文字を直接使用（簡易版）
1. https://www.emojiall.com/ja/emoji/⛳ にアクセス
2. 大きいサイズの画像をダウンロード
3. オンラインリサイザーで 192x192 と 512x512 にリサイズ

## GitHubへのデプロイ

```bash
# リポジトリのルートディレクトリで
cp golf-app.html index.html
git add index.html manifest.json sw.js icon-192.png icon-512.png
git commit -m "Add PWA support"
git push
```

## iPhoneでのインストール方法

1. Safariで https://guncys-inc.github.io/olympic_score/ を開く
2. 画面下部の「共有」ボタン（□に↑）をタップ
3. 「ホーム画面に追加」をタップ
4. 「追加」をタップ
5. ホーム画面に「Golf Olympic」アイコンが追加される

### iPhoneの注意点
- **必ずSafariで開く**こと（Chrome不可）
- アイコンは `apple-touch-icon` で指定したものが使用される
- フルスクリーンで動作（ブラウザのUIが非表示）

## Androidでのインストール方法

1. Chrome または Edge で https://guncys-inc.github.io/olympic_score/ を開く
2. メニュー（⋮）→「アプリをインストール」または「ホーム画面に追加」をタップ
3. 「インストール」をタップ
4. ホーム画面とアプリドロワーに追加される

### Androidの注意点
- Chrome、Edge、Firefox などで動作
- 自動的にPWAプロンプトが表示される
- manifest.jsonの設定がそのまま使用される

## 確認方法

### デスクトップ（開発者ツール）
1. Chrome DevTools → Application タブ
2. 左側の「Manifest」で設定を確認
3. 左側の「Service Workers」で登録状況を確認

### モバイル
1. インストール後、アプリアイコンをタップ
2. フルスクリーンで起動することを確認
3. 機内モードにしてオフラインでも動作するか確認

## トラブルシューティング

### アイコンが表示されない
- ファイル名が正確か確認（icon-192.png, icon-512.png）
- GitHubにアップロードされているか確認
- ブラウザのキャッシュをクリア

### Service Workerが登録されない
- HTTPSで配信されているか確認（GitHubページは自動的にHTTPS）
- Console で "SW registered" が表示されるか確認
- sw.js のパスが正しいか確認

### iPhoneで「ホーム画面に追加」が出ない
- **Safariで開いているか確認**（必須）
- manifest.json が正しく配信されているか確認
- キャッシュをクリアして再度アクセス

### インストールプロンプトが表示されない（Android）
- manifest.json の start_url が正しいか確認
- Service Worker が正常に登録されているか確認
- 一度サイトを離れて再度訪問してみる

## 機能

### PWA化で追加される機能
✅ ホーム画面にアイコン追加
✅ アプリのように起動（フルスクリーン）
✅ オフラインでも動作
✅ ブックマークより簡単にアクセス
✅ ネイティブアプリのような体験

### 今後追加できる機能
- プッシュ通知（スコア更新通知）
- バックグラウンド同期
- デバイス間のデータ同期

## 現在のURL
https://guncys-inc.github.io/olympic_score/

## 更新時の注意
Service Workerのキャッシュバージョン（CACHE_NAME）を変更することで、ユーザーに最新版を配信できます。
sw.js の `const CACHE_NAME = 'golf-olympic-v1';` の v1 を v2, v3... と増やしてください。
