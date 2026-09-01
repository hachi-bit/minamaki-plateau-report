# 希望が丘 木になるマップ

PLATEAUの3D都市モデル（建物・地形）の上に、住民から寄せられた「気になること」の投稿をピンとして表示するWebアプリ。投稿はGoogleフォームで受け付け、GAS（Google Apps Script）がGitHubにデータをpushする形で更新される。

## ファイル構成

- `index.html` — フロントエンド一式（CesiumJSで3D地図を表示、投稿ピンの描画、Googleフォームへの導線）。ビルド不要、このファイル単体で動く静的ページ。
- `data/town-config.json` — 町ごとに変わる設定（サイトタイトル・初期カメラ位置・建物3D TilesのURL・Googleフォームの情報）。**GAS側の`pushTownConfig()`が生成してこのリポジトリにpushする**。
- `data/concerns.json` — 投稿データ本体。GASがGoogleフォームの回答を受け取ってここにpushする。

`index.html`内にも同じ形の値がフォールバックとして埋め込まれているが、通常は`town-config.json`の値が優先される（`town-config.json`が読み込めなかった時の保険）。

## 別の町に切り替える手順

1. GAS側のスクリプトプロパティ（`SITE_TITLE` / `CAMERA_COORDS` / `BUILDING_URL_1` / `BUILDING_URL_2`）を編集し、`pushTownConfig()`を実行する → `data/town-config.json`が自動更新される。
2. 新しい町用のGoogleフォームを用意する（既存フォームのタイトル変更・複製、または新規作成）。フォームの質問を作り直した場合はentry IDが変わるので、下記の方法で確認して`town-config.json`の`form`セクションを更新する。
3. `data/concerns.json`の`responses`を空配列にして、以前の町の投稿データをクリアする。
4. （任意）`index.html`内のフォールバック値、`REPORTER_KEY`のlocalStorageキー名（投稿者IDの保存キー）も新しい町に合わせておくと、コードの一貫性が保たれる。

### Googleフォームのentry IDの確認方法

フォームの編集画面で、対象の質問（緯度・経度・報告者ID）の右上「⋮」→「回答を事前入力したリンクを取得」を選び、適当な値を入れてリンクを生成する。生成されたURLに含まれる`entry.XXXXXXXXX=値`の`entry.XXXXXXXXX`部分がentry ID。フォームのタイトル変更や質問文の編集だけではentry IDは変わらないが、質問を削除して作り直すと変わる。

## 地形データについて

- 以前はCesium ionの共有アクセストークン（PLATEAU公式チュートリアルに掲載されていたもの）経由で「PLATEAU-Terrain」を取得していたが、このトークンが失効（401エラー）して地図が表示されなくなったことがあった。
- 現在は**Cesium ion不要・認証不要**のPLATEAU公式配信URL（`https://tile.plateauview.mlit.go.jp/terrain`、quantized-mesh形式）を`CesiumTerrainProvider.fromUrl()`で直接読み込んでいる。トークン管理が不要になり、失効リスクがなくなった。
- 万一この配信URLが使えなくなったら、[PLATEAU配信サービス ドキュメントサイト](https://docs.plateauview.mlit.go.jp/datasets/terrain/)で最新の配信方法を確認する。
- 地形データの読み込みに失敗しても地図自体は止まらず、平坦な地形（起伏なし）にフォールバックして続行する仕様になっている。

## よくあるトラブルと確認方法

すべて画面右下の🐞ボタンからデバッグログが見られる。何か表示がおかしい時はまずここを確認する。

| 症状 | 確認・対処 |
|---|---|
| ロード画面のクルクルが終わらない | 🐞ログにエラー内容が表示される（起動処理は全体をtry/catchで囲んであり、失敗時はここにメッセージが出る） |
| 建物と地面の間に隙間がある | 地形データの読み込みに失敗している可能性。🐞ログの「地形データの読み込みに失敗」を確認 |
| 地表が暗い | 右下の🌗ボタンで太陽光の陰影表示をON/OFFできる（デフォルトOFF＝常に明るい） |
| Googleフォームに緯度経度・報告者IDが自動入力されない | `town-config.json`のentry IDがフォームの実際の質問とズレている可能性。上記の確認方法で正しいentry IDを取得して更新する |
| 投稿した内容がすぐ地図に反映されない | GAS側がGoogleフォームの回答を受け取って`concerns.json`にpushする仕組みのため、GASのトリガー実行タイミング次第でタイムラグがある |
