# AGENTS.md — nodered ユーザーとして ssh ログインしたエージェントへ

あなたはこの **genpack artifact** に `nodered` ユーザー（uid 1000, groups: `nodered`, `mosquitto`,
shell `/bin/bash`, `HOME=/home/nodered`）として ssh ログインしている。
ここは汎用 Linux ではなく、設計図 `genpack.json5` からビルドされた**不変イメージ**である。

> ⚠ **サービスを外部公開する前に Node-RED の `adminAuth` を設定すること（詳細は §3）。**
> デフォルトではエディタ/管理 API に認証が無く、:80(→1880) に届く誰でもフローをデプロイ＝
> `nodered` 権限でコード実行できてしまう。公開は必ず認証導入後に行うこと。

## 0. 最初に読むこと（最重要）

**`/.genpack/genpack.md` を必ず先に読むこと。** ここでの世界の歩き方
（overlay/SquashFS、永続性の判定、`system.ini` と `/etc` のどちらに書くか、Portage が無いこと、
exec-guard、更新はイメージ差し替え、VM ゲストではセルフアップデート不可 など）はそこに書いてある。
普通の Linux だと思って操作すると失敗するか、成功しても意味を持たない。

素性・来歴は `/.genpack/` にある:

| ファイル | 内容 |
|---|---|
| `/.genpack/genpack.md` | 手引き（まずこれ） |
| `/.genpack/packages` | このイメージに入っているパッケージ一覧（DESCRIPTION / USE / LICENSE 付き） |
| `/.genpack/arch` / `artifact` / `profile` / `banner` / `timestamp.commit` | 素性メタ |

脆弱性検査は `genpack-glsa-check`。genpack 自体の設計詳細はイメージには同梱されず、
公開リポジトリ [wbrxcorp/genpack](https://github.com/wbrxcorp/genpack) の `docs/*.md` にある。

## 1. この artifact の中身

- **Node-RED**（Node.js 製, `ja` 表示）— systemd `nodered.service`（User=nodered）
- **Mosquitto** MQTT ブローカー — systemd `mosquitto.service`（`allow_anonymous true`, 1883）
- **nftables** — systemd `nftables.service`, `main.nft` は TCP 80 を 1880 へ redirect（下記）

**バージョンはビルド時期で変わるため本文に即値を書かない。** 実測は次の通り得る:

```bash
journalctl -u nodered -b | grep -i バージョン   # 起動バナーに Node-RED / Node.js / Linux
node -v                                         # Node.js
grep -iE 'node-red|net-libs/nodejs|mosquitto' /.genpack/packages
```

### エンドポイント（リバースプロキシ公開前提）

nftables はフィルタではなく **NAT redirect だけ** を行う（policy accept, アウトバウンド制限なし）:

```
tcp dport 80 -> redirect :1880     # 標準HTTPで Node-RED (UI / httpNode / OpenAPI) に届く
1883 (mosquitto) 素通し / 22 (ssh)
```

## 2. できること / 権限の範囲

`nodered` は**低特権のサービスアカウント**（root ではない）で、以下が範囲:

- **Node-RED userDir を完全に支配** — `/home/nodered/.node-red`
  （`settings.js`, `flows.json` / `flows_cred.json`, `node_modules/`, `lib/`, `.config.*.json`）
  を読み書きできる。`HOME=/home/nodered`、サービス WorkingDirectory は `/home/nodered/nodered`
- **サービスの再起動だけ root 権限で実行可能**（NOPASSWD sudo）:
  ```bash
  sudo systemctl restart nodered       # settings.js 編集後に反映
  sudo systemctl restart mosquitto
  ```
  (`sudo -l` で見られる許可は `systemctl * nodered` と `systemctl * mosquitto` の 2 本だけ。他の sudo は不可)
- **フロー経由で nodered 権限のコード実行** — exec / function ノードでシェル実行、
  home 配下のファイル読み書き、外部 TCP/HTTP 通信
- **MQTT** — `mosquitto` グループ所属、ローカル `1883` に**無認証**で publish/subscribe 可
- **Palette Manager でノード導入**（既定で許可, 外部ネット到達可）

## 3. できないこと / 注意

- `emerge` / `eix` は無い。ソフトウェア追加・OS 更新は**ビルド側の話**（→ §5）
- `/usr` は read-only bind mount（exec-guard）。upper に持ち込んだ ELF 実行・ネイティブ拡張の dlopen は `EPERM`。
  Palette Manager で**純 JS** のノードは入るが、**ネイティブモジュール**は exec-guard と衝突しうる
  （→ `/.genpack/genpack.md` §5 を確認）
- home 以外（`/etc` のシステム領域など）は書けない／書いても genpack-init 管理項目は次回起動で再適用される
- **エディタ/管理 API は既定で無認証**（`adminAuth` 未設定）。:80(→1880) に届く人は誰でも
  フロー編集・デプロイ＝nodered 権限のコード実行できてしまう。**外部公開前に `adminAuth` を設定する**
  （設定方法は §4）。手前のリバースプロキシ側の認証・アクセス制御も併用する
- `credentialSecret` 未設定 → クレデンシャルはシステム生成鍵で暗号化（鍵を失うと復元不能）
- Projects 機能は無効（`editorTheme.projects.enabled=false`）→ flows はフラットな `flows.json`
- `runtimeState.enabled=false` → フローランタイムの start/stop API（`/flows/state`）は無効

## 4. 実用レシピ

```bash
# 設定を編集して反映
${EDITOR:-nano} /home/nodered/.node-red/settings.js
sudo systemctl restart nodered

# ログを追う
journalctl -u nodered -f

# userDir 状態
ls -la /home/nodered/.node-red

# 稼働しているノードパッケージ（core のみ / 追加ノードの確認）
cat /home/nodered/.node-red/.config.nodes.json | head
ls /home/nodered/.node-red/node_modules
```

### 外部公開前: `adminAuth` を有効にする

`settings.js` の `adminAuth`（コメントアウトされた例）を有効化し、bcrypt ハッシュで
認証させる。同時に `credentialSecret` も独自鍵に置き換える（既定はシステム生成鍵）。

```js
credentialSecret: "<長いランダム文字列>",
adminAuth: {
    type: "credentials",
    users: [{ username: "admin", password: "$2a$08$<bcryptハッシュ>", permissions: "*" }],
},
```

`<bcryptハッシュ>` は `nodered` 権限で生成できる（bundled `bcryptjs` は純 JS なので
exec-guard に抵触しない。Node v26 では `NODE_PATH` 経由ではなく **node-red の `node_modules`
内で `require`** するのが確実）:

```bash
cd /usr/lib64/node_modules/node-red/node_modules && \
  node -e 'console.log(require("bcryptjs").hashSync("パスワード", 8))'
```

編集後は `sudo systemctl restart nodered`。設定後は **http のみでなく httpNode/httpStatic
経由のエンドポイント**も保護されること、TLS/リバースプロキシの設定も併せて確認すること。
（`settings.js` 冒頭コメントに `adminAuth` / `httpStaticAuth` の雛形がある）

永続性（書き込みが再起動を越えて残るか）は**デプロイ構成で決まる**ので、この環境が
常に残るとは限らない。upper 層がデータディスクか tmpfs かの判定・永続化の扱いには
**`/.genpack/genpack.md` §2 の一般則**に従うこと。

## 5. 範囲を超えるとき

足りないソフトウェアの追加、恒久的なシステム設定変更、OS・パッケージの更新、
Node-RED/Mosquitto 本体のバージョン変更 — これらはすべて **アーティファクトの
`genpack.json5` とビルドを更新する話**であり、稼働中に作り込むものではない。
**迷ったらまずユーザに相談すること**（→ `/.genpack/genpack.md` §9）。
