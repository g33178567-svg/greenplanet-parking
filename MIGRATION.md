# 移植手順

このプロジェクトを別の Google アカウント、別の Git リポジトリ、別の Vercel プロジェクトへ移すための手順です。

## 引き継ぐ対象

- このリポジトリ一式
- Google Spreadsheet 本体
- Spreadsheet に紐づく Apps Script
- Apps Script の Web アプリデプロイ
- Vercel プロジェクト設定
- `GAS_WEB_APP_URL`
- `Settings` シートの設定値

## 先に決めること

- 既存の予約データを引き継ぐか
- 新 Google アカウント側で Spreadsheet を新規作成するか、既存 Spreadsheet をコピーするか
- Git の履歴をそのまま残すか

この構成では Spreadsheet にバインドした Apps Script が最も安全です。ただしスタンドアロン Apps Script でも `SPREADSHEET_ID` を設定すれば動かせます。

## 1. Git リポジトリ移行

新しい Git リポジトリを作成したあと、この作業ディレクトリから新しい `origin` を向けます。

```powershell
git remote rename origin old-origin
git remote add origin <NEW_REPO_URL>
git push -u origin main
```

履歴ごと移したくない場合は、新規リポジトリを作って必要ファイルだけコピーしても構いません。ただし通常は履歴ごと移す方が安全です。

## 2. Google アカウント移行

### Spreadsheet

新 Google アカウントで Spreadsheet を用意し、最低でも次の 4 シートを持たせます。

- `Slots`
- `Reservations`
- `Settings`
- `Logs`

既存データを引き継ぐ場合は、現在の Spreadsheet をコピーしてから新アカウント側で管理するのが簡単です。

### Apps Script

新 Google アカウントで対象 Spreadsheet を開き、`拡張機能 > Apps Script` からコンテナバインドされた Apps Script を作成する方法が最も安全です。

基本的には `gas/Code.gs` を反映すれば動かせます。

`gas/appsscript.json` は clasp で管理する場合や、Apps Script の manifest を明示的に編集したい場合だけ使います。Apps Script エディタ上で移植するだけなら無理に触る必要はありません。

`gas/Code.gs` を反映したあと、次のどちらかを 1 回実行してください。

- Spreadsheet バインド型: `setupProject()`
- スタンドアロン型: `setupProjectBySpreadsheetId('スプレッドシートID')`

これで `Slots`、`Reservations`、`Settings`、`Logs` の各シートと初期ヘッダが自動作成されます。スタンドアロン型では Script Properties に `SPREADSHEET_ID` も保存され、その後の API 実行でも同じ Spreadsheet を参照します。

### Settings シート

`Settings` シートに少なくとも次の値を設定します。

- `SLOT_COUNT`
- `RESERVABLE_DAYS_AHEAD`
- `MIN_DURATION_MIN`
- `MAX_DURATION_MIN`
- `CANCEL_DEADLINE_MIN`
- `ADMIN_KEY`

管理者キーは `Settings` シートの `ADMIN_KEY` を編集すれば変更できます。`Script Properties` を使う運用でも動きますが、今の構成では `Settings` シートに寄せた方が引き継ぎしやすいです。

### Web アプリ再デプロイ

Apps Script で Web アプリを新規デプロイし、新しい `/exec` URL を取得します。

この URL は旧環境と別物になるので、Next.js / Vercel 側の `GAS_WEB_APP_URL` も必ず更新してください。

## 3. Vercel 移行

新しい Vercel アカウントまたはチームで、移行先の Git リポジトリを Import して新しいプロジェクトを作成します。

設定する環境変数:

- `GAS_WEB_APP_URL`

推奨:

- Production
- Preview
- Development

環境変数の変更は既存デプロイには反映されないため、設定後に再デプロイが必要です。

## 4. ローカル開発環境

ローカルでは `.env.local` を作成し、次を設定します。

```env
GAS_WEB_APP_URL=https://script.google.com/macros/s/REPLACE_ME/exec
```

テンプレートは `.env.example` を使ってください。

## 5. 移行後チェック

### フロント

- `/` で空き状況が表示できる
- `/reserve` で予約作成できる
- `/cancel` で取消できる
- `/admin` で管理者キー認証が通る

### GAS / Spreadsheet

- `Reservations` に行が追加される
- `Logs` に操作ログが入る
- `Settings.ADMIN_KEY` が意図した値になっている

### Vercel

- 最新デプロイでビルドが成功する
- `GAS_WEB_APP_URL` が新しい `/exec` URL を向いている

## 移植時の注意

- `.env.local` は Git 管理外です
- Apps Script のデプロイ状態と URL はリポジトリに含まれません
- Spreadsheet の共有設定、所有権、データ本体はリポジトリに含まれません
- Script Properties を使っている場合、その値は手動移行が必要です
