# ゴルフオリンピック スコア管理アプリ

ゴルフラウンド中のスコアをリアルタイムで共有・管理できるWebアプリケーションです。

## 🎯 主な機能

### スコア管理
- **18ホール対応**: OUT/IN 各9ホールのスコア記録
- **複数プレイヤー**: 2〜4人まで対応
- **ポイント制**: 金メダル、銀メダル、銅メダル、鉄、ダイヤモンド、イーグル、バーディー、ニアピン、ドラコンなど多彩なスコアタイプ
- **リアルタイム同期**: Firebase Realtime Databaseによる即座のデータ共有

### ラウンド管理
- **ラウンドID共有**: 固有のIDを共有して複数人で同じラウンドに参加
- **履歴機能**: 過去に参加したラウンドIDをサジェスト表示
- **キャリーオーバー**: ニアピン・ドラコンの該当者なしを次ホールに繰越
- **ニアピン/ドラコンホール設定**: 任意のホールに設定可能

### ユーザー管理
- **Google認証**: Googleアカウントでログイン
- **承認制**: 管理者による承認後に利用可能
- **管理者機能**: ユーザーの承認・却下、管理者権限の付与

### その他
- **プリセット保存**: コース設定やポイント設定を保存・読込
- **プレイヤーカスタマイズ**: アバター・名前の変更
- **レスポンシブデザイン**: モバイル・デスクトップ両対応

## 🚀 デモ

https://guncys-inc.github.io/olympic-score/

## 📋 必要要件

- モダンブラウザ（Chrome, Firefox, Safari, Edge 最新版）
- インターネット接続（Firebase利用のため）

## 🔧 セットアップ

### Firebaseプロジェクトの作成

1. [Firebase Console](https://console.firebase.google.com/) でプロジェクトを作成
2. Authentication で Google プロバイダを有効化
3. Realtime Database を作成（テストモードで開始可）
4. プロジェクト設定から Firebase SDK 設定情報を取得

### 設定ファイルの編集

`index.html` の Firebase設定部分を編集：

```javascript
const firebaseConfig = {
  apiKey: "YOUR_API_KEY",
  authDomain: "YOUR_PROJECT_ID.firebaseapp.com",
  databaseURL: "https://YOUR_PROJECT_ID.firebaseio.com",
  projectId: "YOUR_PROJECT_ID",
  storageBucket: "YOUR_PROJECT_ID.appspot.com",
  messagingSenderId: "YOUR_SENDER_ID",
  appId: "YOUR_APP_ID"
};
```

### デプロイ

#### GitHub Pagesにデプロイ

1. GitHubリポジトリを作成
2. `index.html` をリポジトリにプッシュ
3. Settings → Pages → Source を `main` ブランチに設定
4. 数分後に `https://USERNAME.github.io/REPOSITORY_NAME/` でアクセス可能

#### ローカルで実行

```bash
# シンプルなHTTPサーバーを起動
python -m http.server 8000
# または
npx http-server
```

ブラウザで `http://localhost:8000` にアクセス

## 📱 使い方

### 初回利用

1. アプリにアクセス
2. Googleアカウントでログイン
3. 管理者に承認をリクエスト（画面に表示されるメールアドレスを管理者に共有）
4. 承認後、アプリが利用可能に

### ラウンド作成

1. 「設定」タブを開く
2. 「🆕 新規作成」ボタンをタップ
3. ゴルフ場名、日付、プレイヤー名を設定
4. ニアピン・ドラコンホールを設定
5. 表示されるラウンドIDを他のプレイヤーと共有

### ラウンド参加

1. 「設定」タブを開く
2. 「🔗 参加」ボタンをタップ
3. 共有されたラウンドIDを入力（過去の履歴から選択も可能）
4. 「参加」ボタンをタップ

### スコア入力

1. 「入力」タブでホール番号を選択
2. プレイヤーを選択
3. 該当するスコアボタンをタップ
4. 獲得ポイントが自動計算され表示

### スコア確認

1. 「一覧」タブで全プレイヤーのスコアを確認
2. OUT、IN、TOTALのポイントを一目で把握

## 🔐 セキュリティ

### Firebase Realtime Database ルール例

```json
{
  "rules": {
    "rounds": {
      "$roundId": {
        ".read": "auth != null",
        ".write": "auth != null && (
          !data.exists() ||
          root.child('authorized_users').child(auth.uid).child('isAuthorized').val() === true
        )"
      }
    },
    "authorized_users": {
      ".read": "auth != null",
      "$uid": {
        ".write": "auth != null && (
          !data.exists() ||
          root.child('authorized_users').child(auth.uid).child('isAdmin').val() === true
        )"
      }
    }
  }
}
```

## 🛠 技術スタック

- **フロントエンド**: Vanilla JavaScript, HTML5, CSS3
- **認証**: Firebase Authentication (Google)
- **データベース**: Firebase Realtime Database
- **ホスティング**: GitHub Pages
- **デザイン**: レスポンシブデザイン、モバイルファースト

## 📝 ライセンス

MIT License - 詳細は [LICENSE](LICENSE) を参照

## 🤝 コントリビューション

プルリクエストを歓迎します。大きな変更の場合は、まずissueを開いて変更内容を議論してください。

## 📧 お問い合わせ

問題や提案がある場合は、GitHubのIssueを作成してください。

## 🙏 謝辞

- Firebase による素晴らしいバックエンドサービス
- すべてのコントリビューター

---

**注意**: 本アプリケーションは個人利用・小規模グループでの利用を想定しています。大規模な利用の場合は、Firebaseのプランや料金に注意してください。
