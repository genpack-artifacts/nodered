# nodered — Node-RED genpackアーティファクト

[Node-RED](https://nodered.org/)（フローベースの連携／イベント編集環境）と
[Mosquitto](https://mosquitto.org/)（MQTT ブローカー）を載せた genpack アーティファクト。

## 概要

- **Node-RED** が `1880/tcp`、**Mosquitto** が `1883/tcp`（既定で `allow_anonymous`）で稼働
- **nftables** により `80/tcp → 1880/tcp` へ **NAT redirect**（TLS終端リバースプロキシからの接続を想定。
  フィルタではなく転送のみ、アウトバウンド制限なし）
- systemd サービス: `nodered`（User=nodered）, `mosquitto`, `nftables`
- パッケージ: `net-libs/nodejs`, `app-misc/mosquitto`, `net-firewall/nftables`, `app-admin/sudo`, `dev-vcs/git`
- Node-RED / Node.js のバージョンはビルド時期で変わるため本文中に即値を書かない（実測は VM 内の §「扱い方」参照）

## 扱い方（重要）

この VM にログインした際の **扱い方** — `nodered` アカウントの権限範囲、設定場所、
そして **外部公開前に必ず Node-RED の `adminAuth` を設定する**こと — は、イメージに同梱した
`AGENTS.md` に書いてある。扱うには **`nodered` アカウントで ssh ログインして `AGENTS.md` を参照**すること。

```
files/home/nodered/AGENTS.md      # アーティファクト上の場所（ビルドで /home/nodered/AGENTS.md として焼き込まれる）
/home/nodered/AGENTS.md           # VM 内（nodered でログインして読む）
/.genpack/genpack.md              # genpack アーティファクトとしての一般則（まずここ）
```

`nodered` は低特権のサービスアカウント（root ではない）。sudo NOPASSWD は
`systemctl * nodered` / `systemctl * mosquitto` の 2 本のみ。Node-RED の userDir
（`/home/nodered/.node-red`）の支配と、その範囲内の設定変更・再起動が可能。

## ビルドと起動

```sh
genpack                                    # genpack.json5 からビルド → nodered-<arch>.squashfs
vm run --detach --wait-ssh nodered-$(uname -m).squashfs
ssh nodered@vsock%$(vm cid nodered)        # 運用は nodered で入り、AGENTS.md を読む
# ... （扱い方は AGENTS.md / /.genpack/genpack.md 参照）
exit
vm stop -w nodered                          # 停止までブロック
```

- `root` でもログイン可（素性調査やビルド成果物の確認用。`/.genpack/` 参照）
- `vm run` は `-d` 無指定ならディスクレス(tmpfs overlay)起動で、ファイルシステムへの変更は揮発する。
  書き込みが再起動を越えて残るかを含む永続性はデプロイ構成で決まる（→ `/.genpack/genpack.md` §2）
- SSH 公開鍵は root と uid 1000 番台ユーザーの `authorized_keys` に配られる（`vm.ini` 経由）

## 構成（`files/`）

```
files/
├── build.d/node-red.sh                    # node-red を npm で導入・nodered ユーザー作成・sudoers 許可
├── etc/mosquitto/mosquitto.conf           # allow_anonymous, 1883 listener (v4/v6)
├── etc/nftables/rules/main.nft            # tcp/80 -> 1880 redirect
├── etc/.../systemd/system/nodered.service # User=nodered で node-red 起動
└── home/nodered/AGENTS.md                 # nodered でログインしたエージェント向けの手引き
```

## 外部公開の前に

既定では Node-RED のエディタ／管理 API に**認証がない**。`80`(→`1880`) に届く誰でもフローを
デプロイ＝`nodered` 権限でコード実行できてしまうため、**公開前に `adminAuth` を設定**する
（方法・bcrypt ハッシュ生成コマンドは `AGENTS.md` §4）。TLS 終端は手前のリバースプロキシ側で行う前提。
