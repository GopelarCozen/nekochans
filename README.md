# ai-cat-api

ねこの人格を持った AI とお話できるサービスのバックエンド API

## Getting Started

MacOS を利用する前提の手順になります。

### 環境変数の設定

環境変数の設定を行います。

[direnv](https://github.com/direnv/direnv) を利用すると既存の環境変数に影響を与えないので便利です。

```bash
export OPENAI_API_KEY=OpenAIのAPIキーを指定
export OPEN_WEATHER_API_KEY=https://openweathermap.org/ のAPIキーを指定
export BASIC_AUTH_USERNAME="ベーシック認証のユーザー名（任意の値）"
export BASIC_AUTH_PASSWORD="ベーシック認証のパスワード（任意の値）"
export DB_HOST=PlanetScaleのデータベースホスト名を指定
export DB_NAME=PlanetScaleのデータベース名を指定
export DB_USERNAME=PlanetScaleのデータベースユーザー名を指定
export DB_PASSWORD=PlanetScaleのデータベースパスワードを指定
export SSL_CERT_PATH=`SSL_CERT_PATH` についてを参照
export PLANET_SCALE_SERVICE_TOKEN_ID=`PLANET_SCALE_` から始まる環境変数についてを参照
export PLANET_SCALE_SERVICE_TOKEN_SECRET=`PLANET_SCALE_` から始まる環境変数についてを参照
export PLANET_SCALE_ORG=PlanetScaleの組織名を指定
export PLANET_SCALE_TEST_DB_NAME=参照するPlanetScaleのデータベース名を指定
export PLANET_SCALE_TEST_DB_BRANCH=参照するPlanetScaleのデータベースBranch名を指定
```

#### `SSL_CERT_PATH` について

`SSL_CERT_PATH` は SSL 証明書のパスを指定します。

以下のコマンドで証明書の場所を特定出来ます。

### `PLANET_SCALE_` から始まる環境変数について

データベースのテストの速度低下を回避する為に PlanetScale の以下の API を利用して取得した DBSchema を使って MySQL のコンテナにテスト用のテーブルを作成しています。

https://api-docs.planetscale.com/reference/get-a-branch-schema

`PLANET_SCALE_SERVICE_TOKEN_ID` と `PLANET_SCALE_SERVICE_TOKEN_SECRET` は API の認証情報となります。

必要なスコープは https://api-docs.planetscale.com/reference/get-a-branch-schema を参照してください。

Service tokens の発行方法は下記のドキュメントに記載されています。

https://api-docs.planetscale.com/reference/service-tokens

```bash
openssl version -d
```

筆者の環境だと以下のような結果が表示されました。

```
OPENSSLDIR: "/private/etc/ssl"
```

`/private/etc/ssl/cert.pem` というファイルがあったので以下の通りに指定しています。

```bash
export SSL_CERT_PATH=/private/etc/ssl/cert.pem
```

### uv のインストール

Homebrew でインストールを実施します。

```bash
brew install uv
```

### 依存 package のインストール & 仮想環境の作成

以下を実行します。

```bash
uv sync --frozen
```

### Docker による環境構築

[Docker Desktop](https://www.docker.com/products/docker-desktop/) もしくは [OrbStack](https://orbstack.dev/) がインストールされている場合は Docker による環境構築も可能です。

以下のコマンドでコンテナを起動します。

```bash
docker compose up --build -d
```

※ 2 回目以降は `docker compose up -d` だけで大丈夫です。

以下のコマンドを実行して SSE のレスポンスが返ってくれば正常動作しています。

```bash
API_CREDENTIAL=`echo -n "$BASIC_AUTH_USERNAME:$BASIC_AUTH_PASSWORD" | base64`
curl -v -N \
-X POST \
-H "Content-Type: application/json" \
-H "Authorization: Basic $API_CREDENTIAL" \
-H "Accept: text/event-stream" \
-d '
{
  "userId": "6a17f37c-996e-7782-fefd-d71eb7eaaa37",
  "message": "こんにちはもこちゃん🐱"
}' \
http://localhost:5002/cats/moko/messages-for-guest-users
```

コンテナの中に入る場合は以下を実行します。

```bash
docker compose exec ai-cat-api bash
```

これでコンテナ内でテスト等のコマンドを実行可能です。

### コンテナ内での `make` コマンド利用の注意点

一点注意点があります。

コンテナ内のカレントディレクトリは `/src` になっています。

その為、ここで `make lint` 等を実行しても上手く行きません。

以下のコマンドで `/` に移動します。

```bash
cd /
```

その上で `make lint` 等を実行する必要があります。

これは結構面倒だと思うのでコンテナ内で各タスクを実行する為のタスクを用意しました。

```bash
# コンテナ内でLinterを実行
make lint-container

# コンテナ内でFormatterを実行
make format-container

# コンテナ内でtypecheckを実行
make run-token-creator-container
```

### コンテナ内の MySQL に接続する

以下で接続が可能です。

```bash
mysql -u root -h 127.0.0.1 -p -P 33060
```

パスワードは `DB_PASSWORD` に設定してある値です。

### コンテナの停止

以下でコンテナを停止します。

```bash
docker compose down
```

もしも `Dockerfile` や `docker-compose.yml` に変更があった場合は以下のコマンドでコンテナを完全に廃棄してから再度 `docker compose up --build -d` を実行するようにお願いします。

```bash
docker compose down --rmi all --volumes --remove-orphans
```

## 各コマンドの説明

`Makefile` に定義されているコマンドについて説明します。

`make` が利用出来ない場合は以下のコマンドでインストールします。

```bash
brew install make
```

### Linter を実行

```bash
make lint
```

### Formatter を実行

```bash
make format
```

### テストコードの実行

テストコードの実行は Docker コンテナを立ち上げる必要があります。

この README 内の「Docker による環境構築」を参考にコンテナを起動してから下記を実行します。

```bash
make test-container
```

### typecheck の実行

```bash
make typecheck
```

## `uv` でアプリケーションサーバーを起動する

以下を実行するとアプリケーションサーバーが起動します。

ただし MySQL のコンテナが立ち上がっている必要があるので結局は Docker を利用する事になります。

ただしデバッガーでブレイクポイント等を利用する場合はこちらのほうがやりやすいと思います。

```bash
make run
```

以下で動作確認が可能です。

```bash
API_CREDENTIAL=`echo -n "$BASIC_AUTH_USERNAME:$BASIC_AUTH_PASSWORD" | base64`
curl -v -N \
-X POST \
-H "Content-Type: application/json" \
-H "Authorization: Basic $API_CREDENTIAL" \
-H "Accept: text/event-stream" \
-d '
{
  "userId": "6a17f37c-996e-7782-fefd-d71eb7eaaa37",
  "message": "こんにちはもこちゃん🐱"
}' \
http://0.0.0.0:8000/cats/moko/messages-for-guest-users
```

## テストコードの作成について

GitHubActions 上のテストコードは並列実行されています。

その為、固定の DB 名を用いたテストコードだとテストが失敗する可能性があります。

既存テストコードを参考にテストケース毎にユニークな DB 名を生成するようにお願いします。（`tests/db/setup_test_database.py` の `create_test_db_name` を利用します）

ローカルの MySQL コンテナにはどんどんテスト用のデータベースが作成されてしまうので、定期的に以下を実行してコンテナを作り直す事を推奨します。

```bash
# 一度コンテナを削除（MySQLのコンテナ内のデータも含めて削除）
docker compose down --rmi all --volumes --remove-orphans

# 再度コンテナを立ち上げる
docker compose up --build -d
```

## LLM の精度評価を行う

以下のテストコードを実行するとねこの人格を持った AI のレスポンス評価を LLM を使って評価します。

```bash
rye run pytest -vv -s tests/infrastructure/repository/openai/openai_cat_message_repository/test_generate_message_for_guest_user.py
```

ただし普段はテスト実行時間があまりにも長い事や、API の利用料金が高くなってしまうのでテストをスキップするようにしています。

実行する際は `tests/infrastructure/repository/openai/openai_cat_message_repository/test_generate_message_for_guest_user.py` の以下の部分をコメントアウトしてください。

```python
import pytest

@pytest.mark.skip(reason="APIの課金が発生する、実行時間が非常に長いため普段はスキップ")
```
