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
- [ ] MFMテキストのパース・レンダリング（`@misskey-dev/mfm-js` を流用）
- [ ] IndexedDBへのAutomergeドキュメント永続化（Automerge-Repo + IndexedDB adapter）
- [ ] カスタム絵文字の登録・表示（絵文字一覧をAutomergeのMapで管理）

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
  "content": "Text",                 // AutomergeのText型 -> MFMパーサを通してレンダリング
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
- MFMの意味的整合性はアプリ層での確認が将来的に必要

---

## 技術スタック

| 層 | 技術 | 備考 |
| --- | --- | --- |
| フロントエンド | React + Vite + TypeScript | Automergeの公式サンプルがReact中心 |
| CRDTライブラリ | `@automerge/automerge` + `@automerge/automerge-repo` | StorageとNetworkを抽象化 |
| 永続化 | IndexedDB（Automerge-Repo標準アダプタ） | ブラウザ・Electron・Capacitorで共通利用可 |
| MFMパーサ | `@misskey-dev/mfm-js` | Misskey公式パーサをnpmから直接利用 |
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
