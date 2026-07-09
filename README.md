# personacial-log-media

## 目標

個人用サーバとしてスマホ一台でホストでき、P2Pでデバイス間同期ができ、
かつ連合も可能で、Misskeyのように多機能、かつDiscordライクな操作感が
得られるサービス。

### 究極目標

広義のソーシャルメディアを、個人的なログとしても使えるよう一般化した
情報共有・記録基盤を提供することが、このプロジェクトの究極目標である。

具体的には、以下のような多様な用途すべてに対応しうる
カスタム性・拡張性を備えたホスティングソフトウェアの開発を目指す：

- **私的チャット**（LINE 的な閉鎖的メッセージング）
- **半公開チャット**（Discord 的なサーバ・チャンネル型社交空間）
- **私的ログ・文書管理**（Obsidian 的なローカルファーストな知識整理）
- **公開掲示板・マイクロブログ**（Misskey / Bluesky 的な連合型投稿）
- **写真共有**（Instagram 的なメディア中心の公開プロフィール）
- **動画共有**（ニコニコ動画的なコメント付き動画ホスティング）
- **音楽共有**（SoundCloud 的な音声メディアホスティング）
- **その他あらゆる情報整理・共有の形態**

すなわち、「用途を限定しないこと」そのものが設計原則であり、
プラグインやテーマ・スキーマの差し替えによって
任意のメディア形態に変形できるプラットフォームを目指す。
Misskey Flavored Markup Language 的なリッチテキスト修飾構文パース機能を
基盤として備えつつ、各メディア形態に応じた拡張を可能とする。

#### 連合方式の選択制

- **サーバ設立時**：連合機能を有効にするか否かを選択できるようにし、有効にする場合は
  ActivityPub方式・AT Protocol方式のいずれの連合プロトコルを採用するかを選択できるようにする。
- **アカウント登録時**（連合可能なサーバへの登録時）：[Bridgy Fed](https://fed.brid.gy/)（brid.gy）を介して
  他方の連合圏（ActivityPub⇔AT Protocol）とブリッジ接続するか否かを選択する画面を表示する。

---

## フェーズ別ロードマップ

### Phase 1（現在）: ローカルファースト個人メモ（MVP）

- サーバ不要。アプリをインストールして起動すれば即利用可能
- CRDTを用いたローカルファーストなデータ層（Automerge）
- 1端末で完結する手帳として動作することをゴールとする

**実装スコープ**

- [ ] 投稿の作成・表示・削除（ローカルのみ）
- [ ] MFMライクなテキストのパース・レンダリング
  （CommonMark準拠パーサ（`markdown-it`等）をベースに、絵文字コード・メンション・
  ハッシュタグ・MFM関数`$[...]`等のMFM由来記法を独自プラグインとして追加する方式。
  `@misskey-dev/mfm-js`はPEG.jsによる独自文法でCommonMark拡張として実装されておらず
  "not up to spec" と評されているため採用しない（[marked-mfm](https://www.npmjs.com/package/marked-mfm)）。
  Fedify/Holloの`@fedify/markdown-it-hashtag`・`@fedify/markdown-it-mention`
  （[fedify-dev/hollo](https://github.com/fedify-dev/hollo)）を参考実装とする）
- [ ] IndexedDBへのAutomergeドキュメント永続化（Automerge-Repo + IndexedDB adapter）
- [ ] カスタム絵文字の登録・表示（絵文字一覧をAutomergeのMapで管理）
- [ ] 投稿作成時のレスポンシブプレビュー表示（スマートフォン・タブレット・PC等、主要な画面幅での見え方を並べて確認できるようにする）
  （注：これは自クライアントのCSSブレークポイントに基づく再現であり、連合先の別実装クライアント（Mastodon系・Bluesky系アプリ等）での実際の見え方までは保証しない。MFM等独自装飾構文は他実装では正しく解釈されない場合がある点に留意）

### Phase 2: 複数端末P2P同期

- 複数端末間をP2Pで同期（Automerge-Repoのネットワーク層を活用）
- サーバの有無を選択できる
  - P2Pで同期を回すか、ひとつの中継サーバに依存させるかを選べる

### Phase 3: 小規模共有・連合

- AT Protocol的な署名付きリポジトリを用いた連合機能を検討
- ActivityPubとのブリッジは将来的な課題

---

## アーキテクチャ

```
┌───────────────────────────────────┐
│           UI 層                   │  <- フロントエンド（React + Vite）
├───────────────────────────────────┤
│        ビジネスロジック層           │  <- 投稿操作・検索・タイムライン生成
├───────────────────────────────────┤
│         同期・ストレージ層          │  <- Automerge（CRDT）+ IndexedDB
├───────────────────────────────────┤
│     トランスポート層（P2P）         │  <- WebRTC / LAN（Phase 2以降）
├───────────────────────────────────┤
│     連合層（Phase 3以降）          │  <- AT Protocol / ActivityPubブリッジ
└───────────────────────────────────┘
```

---

## データモデル

### Automergeドキュメントの粒度方針

「**一投稿一ドキュメント + 全体インデックスドキュメント**」を採用する。

- 増分同期が軽量になる
- AT Protocolの「各レコードを独立したエントリとして扱う」設計と親和性が高く、Phase 3への移行が容易

### 投稿ドキュメント（Post）

```jsonc
{
  "id": "ImmutableString",           // UUIDv7（時刻順ソートに有利）
  "authorId": "ImmutableString",     // 作者識別子（Phase 1はデバイスローカルなUUID）
  "content": "Text",                 // AutomergeのText型 -> CommonMark準拠パーサ+MFM拡張プラグインを通してレンダリング
  "contentType": "ImmutableString",  // "mfm" | "plaintext"
  "attachments": [                   // List型
    {
      "type": "ImmutableString",     // "image" | "video" | "file"
      "blobId": "ImmutableString"    // コンテンツアドレス（SHA-256ハッシュ）
    }
  ],
  "replyTo": "ImmutableString | null",     // 返信先Post ID
  "threadRoot": "ImmutableString | null",  // スレッド根Post ID
  "createdAt": "Timestamp",
  "editedAt": "Timestamp | null",
  "visibility": "ImmutableString"    // "private" | "local" | "public"
}
```

### リアクションドキュメント（Reaction）

リアクションは投稿ドキュメントに埋め込まず、独立したドキュメントとして管理する。
（複数端末からの同時リアクションによる競合を避けるため）

```jsonc
{
  "targetPostId": "ImmutableString",
  "reactions": {
    "[emoji_code]": { "[userId]": true }
    // キーがemojiCode、値がauthorId -> trueのMap
  }
}
```

### インデックスドキュメント（Timeline）

全投稿のIDと作成時刻を保持するドキュメント。タイムライン表示に使用。

```jsonc
{
  "posts": [
    { "id": "ImmutableString", "createdAt": "Timestamp" }
    // append-only
  ]
}
```

### 同期競合の解決方針

- テキスト本文：AutomergeのText型（Peritext CRDT）が自動マージ
- その他のフィールド：Phase 1ではLast-Write-Wins（最後に編集したデバイスを優先）
- MFM拡張記法（絵文字コード・メンション・ハッシュタグ・MFM関数等）の意味的整合性はアプリ層での確認が将来的に必要

---

## 技術スタック

| 層 | 技術 | 備考 |
| --- | --- | --- |
| フロントエンド | React + Vite + TypeScript | Automergeの公式サンプルがReact中心 |
| CRDTライブラリ | `@automerge/automerge` + `@automerge/automerge-repo` | StorageとNetworkを抽象化 |
| 永続化 | IndexedDB（Automerge-Repo標準アダプタ） | ブラウザ・Electron・Capacitorで共通利用可 |
| Markdownパーサ | `markdown-it`（CommonMark準拠）+ 独自MFM拡張プラグイン群 | 絵文字コード・メンション・ハッシュタグ・MFM関数`$[...]`等をプラグインとして追加。Fedify/Holloの`markdown-it-hashtag`/`markdown-it-mention`を参考実装とする |
| モバイル対応 | Capacitor（将来） | WebアプリをそのままAndroid/iOSアプリ化 |
| 連合層 | AT Protocol（Phase 3以降） | |

---

## 未解決の設計課題

1. **ユーザ識別子の設計**
   - Phase 1はデバイスローカルなUUIDで代替
   - Phase 2以降で複数デバイスをまたいだ同一人物の識別が必要になる
   - AT ProtocolのDID（分散識別子）が将来的な答えの候補

2. **添付ファイルのストレージ**
   - テキストはAutomergeで管理するが、画像・動画の実体はCRDTではなく**コンテンツアドレス型ストレージ**（SHA-256ハッシュでファイルを参照する方式）で別管理するのが定番

3. **スレッドとチャンネルの共存**
   - Misskeyライクな「タイムライン投稿」とDiscordライクな「チャンネル内チャット」をどう統一的に表現するか
   - Roomyの `Space -> Channel -> Thread -> Message` 階層構造が参考になる

4. **投稿編集の表現方式（sed的自己リプライ）**
   - 自身の投稿に対し `s/hoge/fuga/g` のようなsed風の置換コマンドを自己リプライとして送信すると、
     本文が履歴付きで改変される機能を検討する
   - 連合時（Phase 3以降）は、当該コマンドをクライアント側で解釈した結果をActivityPubの
     Updateアクティビティ相当に変換して連合先へ伝える方針とする（Mastodon等が用いる
     編集連合の仕組みに準ずる。Pleroma/Akkomaの`formerRepresentations`拡張のように
     編集履歴自体を連合する方式も参考になる）
   - 未定事項：対応するsedコマンドの範囲（置換のみか、削除・追記等も含めるか）、
     正規表現エンジンの方言（POSIX ERE / PCRE / JS RegExp）、誤置換防止のための確認UI、
     AT Protocol側の編集表現との対応関係

5. **リモートカスタム絵文字のワンタッチ取込とライセンス表記**
   - 自鯖に存在しないカスタム絵文字（他鯖に存在するもの）を、サーバ管理者アカウントが
     クライアントアプリから一操作で自鯖に追加できるようにする
   - 取込に際しては必ずライセンスを併記する。原著作者の許諾意思が確認できない
     （ライセンス不明の）絵文字についても取込自体は妨げず、「ライセンス不明」である旨を
     明示した状態で許可する（表記の義務化は無許諾利用のリスクを可視化するものであり、
     合法性を担保するものではない点に注意）
   - 未定事項：ライセンス情報の取得元（連合元メタデータの引き継ぎか、管理者による自己申告か）、
     再連合時のライセンス情報の伝播方法、Misskeyにおける同種の未解決課題
     （[絵文字ライセンス表記 #10091](https://github.com/misskey-dev/misskey/issues/10091)、
     [インポート元情報の記録 #14021](https://github.com/misskey-dev/misskey/issues/14021)、
     [絵文字ライセンスの連合 #10859](https://github.com/misskey-dev/misskey/issues/10859)）との整合

6. **用語体系の統一**
   - 現状、ActivityPub系用語（「サーバ」「クライアント」等）とAT Protocol系用語
     （「AppView」「PDS」「Relay」等）が併存しており、将来的には独自かつ両者と整合する用語体系を確立する
   - 注意点：ActivityPubの「サーバ」はデータ保存・配信・アプリロジックが一体化した単一インスタンスを
     指すのに対し、AT ProtocolはPDS（データ保存）・AppView（集約・アプリロジック）・Relay（配信最適化）
     に機能を分離したアーキテクチャを採る（参照：[AT Protocol Glossary](https://atproto.com/guides/glossary)）。
     したがって用語統一は単なる呼び方の言い換えでは済まず、機能分割の違いを踏まえたマッピングが必要になる
   - 前述「連合方式の選択制」により両プロトコルを選択的に採用する設計であるため、両陣営の用語対応表（グロッサリ）を
     用意し、採用プロトコルに応じて適切な用語を出し分けられるようにする方針が現実的

7. **マイクロブログ以外のメディア形態への連合対応**
   - 究極目標に掲げる動画共有・音楽共有等について、PeerTube等の先行実装を参考に連合方式を検討する
   - 補足：PeerTubeは独自の別プロトコルを用いているわけではなく、ActivityPubにVideo/Group（チャンネル）等の拡張語彙を
     追加する形で連合を実現しており、実際の動画配信自体はActivityPubの外側でWebTorrent/HLSベースの
     P2P配信機構が担っている（参照：[PeerTube公式ドキュメント](https://docs.joinpeertube.org/api/activitypub)）。
     同様に、当プロジェクトでも「メタデータ連合はActivityPub/AT Protocolの拡張語彙、実体配信は別レイヤ」
     という分離構成を踏襲するのが妥当と考えられる
   - AT Protocol側には同種の確立された動画/音楽連合実装が現状存在しないため、対応する場合はLexicon拡張など独自設計が必要になる

8. **再生キューの階層化・高機能キュー管理**（音楽・動画共有向け）
   - 連続再生用の再生キューを、単純な線形リストではなく、ファイルシステム的な階層構造（グループ／サブグループを持つツリー）
     として設計することを検討する
   - 想定機能：
     - 割り込みキュー：再生中のキューに別のコンテンツを差し込み、優先的に再生する
     - 位置指定スリープタイマー：時刻ではなく「キュー中の特定のコンテンツまで」を指定して自動停止する
     - 階層シャッフル：グループ内の順序を保持したままグループ単位でシャッフルする機能。Apple Music/iTunesの
       「アルバム別・グループ別シャッフル」やSony Walkmanの「シャッフル：アルバム／フォルダ」（いずれも
       グループ内の曲順を保持したままグループの再生順のみをランダム化する）に相当する（参照：
       [Appleサポート](https://support.apple.com/ja-jp/guide/music/mus2989/mac)、
       [Sonyヘルプガイド](https://helpguide.sony.net/ha/ar/v1/ja/contents/TP0000067787.html)）。
       同様の階層的プレイリスト管理はfoobar2000の「Playlist Tree」コンポーネントにも先例がある
       （参照：[foobar2000 Wiki](https://wikiwiki.jp/foobar2000/%E3%82%B3%E3%83%B3%E3%83%9D%E3%83%BC%E3%83%8D%E3%83%B3%E3%83%88%E3%81%AE%E8%A8%AD%E5%AE%9A/Playlist%20Tree)）
   - 未定事項：
     - 割り込みキューの意味論（差し込んだコンテンツ終了後に元のキューへ復帰するスタック型か、
       キュー先頭への恒久的な挿入か）
     - 位置指定スリープタイマーの停止境界（指定コンテンツの再生前で止めるか、再生後まで続けて止めるか）
     - 階層シャッフルにおいて、どの階層をシャッフル対象としどの階層を順序保持対象とするかをユーザが指定できる粒度
       （Sony Walkmanはアルバム階層とフォルダ階層を別モードとして提供しており参考になる）
     - このキュー構造をAutomerge等の同期対象データとして扱い複数端末間で再生キューを共有・引き継げるようにするか、
       各クライアントのローカルな一時状態に留めるか
     - 「キュー」は本来線形構造を指す語であるため、階層化した場合の呼称の見直し（例：再生プレイリストツリー等）

9. **MFM関数記法`$[...]`とMarkdownインライン数式記法の記号衝突**
   - MFM関数は`$[tada text]`のように`$`をプレフィックスとして用いるが、`$`はPandoc・GitHub（2022年以降）・
     GitLab・Obsidian等の多くのMarkdown処理系で「インライン数式（LaTeX記法）」の区切り文字として
     広く使われている（参照：[GitHub Docs: 数式の記述](https://docs.github.com/en/get-started/writing-on-github/working-with-advanced-formatting/writing-mathematical-expressions)、
     [GitHub Changelog 2023-05-08](https://github.blog/changelog/2023-05-08-new-delimiter-syntax-for-inline-mathematical-expressions/)）
   - CommonMark準拠パーサ上にMFM拡張プラグインとして`$[...]`構文を追加する場合、将来的に数式記法
     （`$...$`によるインライン数式サポート）を同一パーサに追加しようとすると、両者の記号が衝突する
     おそれがある
   - 未定事項：MFM関数記法の区切り文字を`$[...]`以外に変更するか、数式記法側の区切り文字を
     `\(...\)`等の代替記法に限定するか、あるいは両者が共存可能な曖昧性解消規則
     （前後の空白・後続文字によるヒューリスティック等、Pandocの数式区切り判定方式を参考にする）
     を設けるか
