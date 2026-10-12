---
title: "Java 不要の Firestore エミュレータ「hidane」v0.1.0 をリリースしました"
emoji: "🔥"
type: "tech"
topics: ["firebase", "firestore", "rust", "emulator", "test"]
published: true
---

Cloud Firestore のローカル開発とテストで使う公式エミュレータを、Rust の単一バイナリで置き換える **hidane（火種）** の最初のリリース、v0.1.0 を公開しました。firebase-tools からそのまま起動でき、Java は要りません。この記事では、使い方と、どうやって「公式と同じ挙動」を確かめているかを紹介します。

- リポジトリ: https://github.com/hidane-dev/hidane
- サイト: https://hidane.dev/ja/
- リリース: https://github.com/hidane-dev/hidane/releases/tag/v0.1.0

:::message
hidane は非公式のオープンソースプロジェクトで、Google LLC とは関係ありません。Firebase と Cloud Firestore は Google LLC の商標です。
:::

:::message
**追記（2026-10-13）**: v0.2 で Security Rules に対応しました。v0.2.0 から `firestore.rules` で読み取り・書き込み・リスナー・クエリを判定し、拒否メッセージ、`:ruleCoverage`、`debug()` の出力まで公式エミュレータと同じです。v0.2.1 で Emulator UI の Requests タブ（ルールの判定の一覧）に対応し、v0.2.2 ではトランザクションの `verify` 書き込みもルールで判定するようになりました。下の「まだできないこと」のうち、Security Rules とリクエストモニタは解消しています。ルールのテストにもそのまま使えます。v0.2.3 では、Functions エミュレータと併用したときに Firestore トリガーを持つ関数があると起動が止まる不具合を直し、rules-unit-testing が Node とブラウザの両方で公式と同じ結果になることも確かめました。v0.3.0 では、firebase-js-sdk 自身のブラウザ用テスト 1,183 件が公式エミュレータと同じく hidane でも通るようになり、Web 対応が揃いました。公式の応答をどこまで比べているかは [parity 表](https://hidane.dev/ja/#parity) にまとめています。最新版は[リリース一覧](https://github.com/hidane-dev/hidane/releases)にあります。
:::

![hidane v0.1.0](/images/hidane-firestore-emulator/card.png)

## 使い方

インストールして、いつもの firebase-tools のコマンドの前に `hidane exec --` を付けるだけです。

![インストールと hidane exec](/images/hidane-firestore-emulator/usage.png)

```sh
curl -fsSL https://hidane.dev/install.sh | sh
hidane exec -- firebase emulators:start --only firestore
```

`hidane exec` は、そのコマンドの間だけ hidane を `java` と公式 jar の代わりに見せます。firebase-tools の設定やテストコードはそのままで、`emulators:exec` や Emulator UI も動きます。

他のインストール方法もあります。どれも同じリリースのバイナリが入ります（macOS・Linux・Windows）。

```sh
brew install hidane-dev/tap/hidane     # Homebrew
npm install --save-dev hidane          # npm（プロジェクトの devDependencies に）
cargo binstall hidane                  # または cargo install hidane
dart pub global activate hidane        # pub.dev
docker run --rm -p 8080:8080 ghcr.io/hidane-dev/hidane
```

firebase-tools を使わず、単体で起動することもできます。

```sh
hidane --host 127.0.0.1 --port 8080
export FIRESTORE_EMULATOR_HOST=127.0.0.1:8080   # サーバー側の SDK
```

Web SDK ではいつもどおり `connectFirestoreEmulator(db, '127.0.0.1', 8080)` を使います。

## 公式エミュレータで困っていたこと

公式の Firestore エミュレータは Java 製の jar として配布されています。普段の開発と CI で、次の 3 点が気になっていました。

1. **Java 21 が必須になった**。firebase-tools 15.0.0 から、Java 21 未満のサポートが削除されました。ローカルでデータベースを動かすためだけに JDK を入れる必要があります。
2. **起動とメモリ**。ポートが開くまで 0.74 秒（`firebase emulators:start` 経由では 2.5 秒）、待機中に 95 MiB、1,000 件書き込むと 456 MiB でした。
3. **リスナーがあると書き込みが遅くなる**。コレクションに `onSnapshot` を張った状態で 10 万件をバッチ書き込みすると、保存済みの件数に比例して 1 バッチの時間が伸び、全体で 121 秒かかりました。リスナーが無いときの 11 倍です（[firebase-tools#3477](https://github.com/firebase/firebase-tools/issues/3477)）。Emulator UI を開いているとリスナーが張られるので、「UI を開くと遅い」という報告とも一致します。

## hidane で同じことを測ると

![公式エミュレータと hidane の比較](/images/hidane-firestore-emulator/bench.png)

| | 公式 v1.22.0 | hidane |
|---|---|---|
| ポートが開くまで | 0.74 s（CLI 経由 2.5 s） | 5.5 ms |
| メモリ（待機中 / 1,000 件後） | 95 MiB / 456 MiB | 7.4 MiB / 21 MiB |
| 10 万件 + リスナー | 121 s（件数に比例して遅くなる） | 4.9 s（バッチごとに一定） |

:::message alert
同じマシン・同じ条件で並べた比較ではありません。公式の数値は調査段階に負荷のかかったマシンで測ったもので、hidane は後日、静かな状態の M1 Max で測っています。同一マシンでの再計測は予定しています。
:::

書き込みのコストを「変更した件数」だけに比例させるのが設計上の狙いで、リスナーがいても 1 バッチあたり約 10.5 ms のまま変わりません。

## 「公式と同じ挙動」をどう確かめているか

エミュレータは、本番の Firestore ではなく**公式エミュレータと同じ**振る舞いをすることが大事です。テストは公式エミュレータで書かれているので、エラーメッセージの一字一句やスナップショットの届き方が違うと、乗り換えた途端にテストが落ちます。

そこで hidane では、**公式エミュレータをオラクル（正解）**として扱っています。

![公式エミュレータをオラクルにする仕組み](/images/hidane-firestore-emulator/parity.png)

1. 公式エミュレータに、境界ケースを含むリクエストを大量に投げるスクリプトを書く（`tools/oracle/`）
2. 応答をそのままフィクスチャとして記録する（現在 17 個）
3. hidane のテストスイートが、記録をすべて hidane に対して再生し、一致を確かめる

規模感の例を挙げると、Authorization ヘッダーの扱いで 146 件、ベクトル検索（`find_nearest`）で 87 件、REST の応答で 57 件（こちらはバイト単位で比較）です。

SDK レベルでも、Admin SDK、Node で動かす Web SDK、Lite（REST）、ブラウザで動かす Web SDK、`@firebase/rules-unit-testing` を両方のエミュレータに流し、結果を突き合わせています。トランスクリプトは `results/` に公開しています。

調べていくと、公式の挙動には知らないと戸惑う細部が多くありました。

- ページサイズちょうどで結果が尽きても次のページトークンを返す（最後のページは空）
- `explain_options` は受け付けるが無視する。ベクトル検索は実装されている
- REST のリクエストボディの上限は 16 MiB、gRPC は 100 MiB
- ブラウザ向けの WebChannel では、フレームの長さをバイトではなく UTF-16 の単位で数える

ブラウザ向けの WebChannel は仕様が公開されていないため、Chromium と公式エミュレータの間に記録用のプロキシを挟んで通信を記録し、そこから実装しました。

## export / import も公式と同じ形式

![export / import の互換性](/images/hidane-firestore-emulator/export.png)

`firebase emulators:start --import` / `--export-on-exit` で使うエクスポートは、公式エミュレータと同じ形式（LevelDB のログ形式に App Engine のエンティティを詰めたもの）で読み書きします。ファイルの中身はエンティティ単位で公式とバイト一致することをテストで確かめていて、公式が書いたデータを hidane が、hidane が書いたデータを公式が読めることも、firebase-tools 経由の 4 通りの組み合わせで確認しました。乗り換えても、手元のシードデータはそのまま使えます。

## 公式のバグはあえて再現しない

すべてを真似しているわけではありません。公式のバグと判断したものは再現せず、違いを [docs/parity-exceptions.md](https://github.com/hidane-dev/hidane/blob/main/docs/parity-exceptions.md) にすべて書いています。たとえば次のようなものです。

- 先ほどの「リスナーがあると書き込みが遅くなる」
- `POST /reset` のあと、接続中のリスナーにその後の変更も一切届かなくなる
- 10 秒ごとに HTTP/2 の ping を送るクライアントが、30 秒後に `GOAWAY too_many_pings` で切断される。iOS SDK で報告されている「90 秒で切断される」問題と同じ原因とみています
- WebChannel のセッションが解放されない
- ネストしたフィールドのベクトルを `find_nearest` で検索できない

## v0.1.0 でできること、まだできないこと

できること（公式と突き合わせ済み）:

- ドキュメントの読み書き、クエリ（全演算子・カーソル・コレクショングループ）、集計（count / sum / avg）、ベクトル検索
- トランザクション（公式と同じロックの挙動）
- リアルタイムリスナー（差分配信と再開）
- gRPC・REST・WebChannel を 1 つのポートで提供
- firebase-tools からの起動、Emulator UI（閲覧・作成・削除・全データ削除）、export / import
- Admin SDK、Web SDK（Node・ブラウザ・Lite）、rules-unit-testing（ルール無しの範囲）

まだできないこと:

- **Security Rules**（v0.2 で対応予定。いまは全リクエストを許可します）
- Emulator UI のリクエストモニタ、Functions エミュレータへのイベント通知
- Go・Python・Java・iOS・Android・Flutter での動作確認（同じプロトコルを話しますが、まだ試していません）

ルールを使わないテストや開発なら、いまから公式エミュレータの代わりに使えます。

## 配布について

GitHub Release の各アーカイブには SHA-256 とビルドの来歴証明が付いていて、`gh attestation verify` で確かめられます。crates.io・npm・pub.dev へは GitHub Actions の信頼済み公開（OIDC）で公開していて、レジストリのトークンをリポジトリに置いていません。npm はステージ公開にしていて、メンテナが 2FA で承認するまで公開されません。ライセンスは MIT OR Apache-2.0 です。

## これから

次は v0.2 の Security Rules の評価器です。計画は [ROADMAP](https://github.com/hidane-dev/hidane/blob/main/docs/ROADMAP.md) にあります。

## お願い

お手元のテストスイートで試してみて、公式エミュレータと違う応答があれば、[Parity gap のテンプレート](https://github.com/hidane-dev/hidane/issues/new/choose)で issue を立ててもらえるととても助かります。Go・Python・Java・モバイルの SDK で試した報告（動いた・動かなかった）も歓迎です。使い方の相談は [Discussions](https://github.com/hidane-dev/hidane/discussions) へどうぞ。
