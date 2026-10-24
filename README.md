# PostgreSQL Docker Compose

Docker Compose で PostgreSQL の環境を構築するための定義です。

## 構成

| 項目 | 内容 |
| --- | --- |
| イメージ | `postgres:18-trixie`（PostgreSQL 18 系の最新版 / Debian 13 trixie ベース） |
| コンテナ名 | `postgresql` |
| 公開ポート | `.env` の `POSTGRESQL_PORT`（コンテナ内は 5432） |
| データ保存先 | `./postgresql/data`（ホスト側にバインドマウント） |
| タイムゾーン | `Asia/Tokyo` |

```
.
├── .env.example         # 環境変数のサンプル
├── .env                 # 環境変数（各自作成。Git 管理外）
├── docker-compose.yml
└── postgresql/
    └── data/            # DB のデータファイル（初回起動時に作成。Git 管理外）
```

## 前提条件

- Docker と Docker Compose（`docker compose` コマンド）がインストールされていること

## セットアップ

### 1. 環境変数ファイルを作成する

`.env.example` をコピーして `.env` を作成し、値を環境に合わせて変更します。

```bash
cp .env.example .env
```

| 変数名 | 説明 | 例 |
| --- | --- | --- |
| `POSTGRESQL_PORT` | 外部から接続するポート番号 | `35432` |
| `POSTGRESQL_DB_NAME` | 作成するデータベース名 | `app` |
| `POSTGRESQL_DB_USER` | 接続用ユーザー名 | `app` |
| `POSTGRESQL_DB_PASSWORD` | 接続用ユーザーのパスワード | （任意の強力なパスワード） |

> **注意**
> - パスワードは必ず変更してください。`.env` は `.gitignore` で除外されているため、コミットされません。
> - DB名・ユーザー名・パスワードは **初回起動時（データディレクトリが空のとき）にのみ** 反映されます。初回起動後に `.env` を変更しても既存の DB には反映されません。

### 2. コンテナを起動する

```bash
docker compose up -d
```

初回起動時に `postgresql/data/18/docker/` 配下にデータベースが初期化されます。

### 3. 起動を確認する

```bash
docker compose ps
```

`STATUS` が `Up ... (healthy)` になっていれば起動完了です。

## 接続方法

### psql（コンテナ内）

```bash
docker compose exec postgresql psql -U app -d app
```

### psql（ホストから）

```bash
psql -h localhost -p 35432 -U app -d app
```

### pgAdmin

1. pgAdmin で **Servers** を右クリックし、**Register > Server...** を選択します。
2. **General** タブで、**Name** に任意の名前（例: `postgresql-docker`）を入力します。
3. **Connection** タブに以下を入力します。

   | 項目 | 値 |
   | --- | --- |
   | Host name/address | `localhost`（または `127.0.0.1`） |
   | Port | `.env` の `POSTGRESQL_PORT`（例: `35432`） |
   | Maintenance database | `.env` の `POSTGRESQL_DB_NAME`（例: `app`） |
   | Username | `.env` の `POSTGRESQL_DB_USER`（例: `app`） |
   | Password | `.env` の `POSTGRESQL_DB_PASSWORD` |

4. **Parameters** タブで **+** をクリックし、以下のパラメータを追加します。

   | Name | Value |
   | --- | --- |
   | GSS encryption mode（`gssencmode`） | `disable` |

5. **Save** をクリックして接続します。

> **GSS encryption mode を `disable` にする理由**
>
> Windows 版の pgAdmin では、接続時に以下のエラーが発生して接続できないことがあります。
>
> ```
> exception: access violation writing 0x0000000000000000
> ```
>
> これは pgAdmin が使う接続ライブラリ（libpq）が、接続の最初に GSSAPI（Kerberos）による暗号化接続を試みる処理でクラッシュするためです。
> GSS encryption mode を `disable` にすると GSSAPI の処理が行われなくなり、通常のパスワード認証（scram-sha-256）で接続できるようになります。

## 運用

### 停止

```bash
docker compose down
```

データは `postgresql/data` に残るため、再度 `docker compose up -d` すれば同じデータで起動します。

### ログの確認

```bash
docker compose logs -f postgresql
```

### マイナーバージョンの更新

`postgres:18-trixie` タグは 18 系の最新マイナーバージョンを指します。以下でイメージを更新できます。

```bash
docker compose pull
docker compose up -d
```

### データの初期化

DB を作り直したい場合は、コンテナを停止してデータディレクトリを削除します。**すべてのデータが失われるので注意してください。**

```bash
docker compose down
sudo rm -rf postgresql/data
docker compose up -d
```

> `postgresql/data` 配下のファイルはコンテナ内のユーザー権限で作成されるため、削除には `sudo` が必要です。
