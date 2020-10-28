[![FIWARE Banner](https://fiware.github.io/tutorials.IoT-Agent/img/fiware.png)](https://www.fiware.org/developers)
[![NGSI v2](https://img.shields.io/badge/NGSI-v2-5dc0cf.svg)](https://fiware-ges.github.io/orion/api/v2/stable/)

[![FIWARE IoT Agents](https://nexus.lab.fiware.org/repository/raw/public/badges/chapters/iot-agents.svg)](https://github.com/FIWARE/catalogue/blob/master/iot-agents/README.md)
[![License: MIT](https://img.shields.io/github/license/fiware/tutorials.Iot-Agent.svg)](https://opensource.org/licenses/MIT)
[![Support badge](https://img.shields.io/badge/tag-fiware-orange.svg?logo=stackoverflow)](https://stackoverflow.com/questions/tagged/fiware)
![XML](https://img.shields.io/badge/Payload-XML-e8ce27.svg)<br/>
[![Documentation](https://img.shields.io/readthedocs/fiware-tutorials.svg)](https://fiware-tutorials.rtfd.io)

このチュートリアルでは、カスタム [XML](https://www.w3.org/TR/xml11/) メッセージ形式を使用して応答する ダミー IoT
デバイスを接続します。**Custom IoT Agent** は、[Orion Context Broker](https://fiware-orion.readthedocs.io/en/latest/)
に送信された [NGSI-v2](https://fiware.github.io/specifications/OpenAPI/ngsiv2) リクエストを使用して測定値を読み取り、
コマンドを送信できるように、IoT Agent Node.js [ライブラリ](https://iotagent-node-lib.readthedocs.io/en/latest/) と
[IoT Agent for Ultralight](https://fiware-iotagent-ul.readthedocs.io/en/latest/usermanual/index.html#user-programmers-manual)
デバイスにあるフレームワークに基づいて作成されます。

チュートリアルでは、全体を通して [cUrl](https://ec.haxx.se/) コマンドを使用していますが、
[Postman ドキュメント](https://fiware.github.io/tutorials.Custom-IoT-Agent/)としても利用できます。

[![Run in Postman](https://run.pstmn.io/button.svg)](https://app.getpostman.com/run-collection/f554d8e76cd7af1fe796)

## コンテンツ

<details>
<summary><strong>詳細</strong></summary>

-   [カスタム・メッセージ形式を渡す](#passing-custom-message-formats)
    -   [このチュートリアルの目標](#the-teaching-goal-of-this-tutorial)
    -   [共通機能の再利用](#reusing-common-functionality)
-   [アーキテクチャ](#architecture)
    -   [ダミー IoT デバイスの構成](#dummy-iot-devices-configuration)
    -   [カスタム XML IoT Agent の構成](#custom-xml-iot-agent-configuration)
-   [前提条件](#prerequisites)
    -   [Docker](#docker)
    -   [Cygwin](#cygwin)
-   [起動](#start-up)
-   [カスタム IoT Agent の作成](#creating-a-custom-iot-agent)
    -   [IoT Agent サービス の健全性の確認](#checking-the-iot-agent-service-health)
    -   [サービス・グル​​ープのプロビジョニング](#provisioning-a-service-group)
    -   [センサのプロビジョニング](#provisioning-a-sensor)
    -   [アクチュエータのプロビジョニング](#provisioning-an-actuator)

</details>

<a name="passing-custom-message-formats"/>

# カスタム・メッセージ形式を渡す

> "And the whole earth was of one language, and of one speech."
>
>— Genesis 11:1

以前に定義したように、IoT Agent は、デバイスのグループが独自のネイティブ・プロトコルを使用して Context Broker に
データを送信し、Context Broker から管理できるようにするコンポーネントです。すべての IoT Agent は単一のペイロード形式に
対して定義されていますが、そのペイロードには複数の異なるトランスポートを使用できる場合があります。

多くの標準ペイロード用の IoT Agent が存在しますが、コンテキスト・データの多くの潜在的なソースには、システムの周りに
データを渡すための明確なデファクトまたはデジュール標準が既にあるため、追加のペイロードが必要になる場合があることを
想定することは可能です。例として、**ISOXML** 標準
[iso:11783](https://www.iso.org/obp/ui/#iso:std:iso:11783:-10:ed-2:v1:en) は農業機械で頻繁に使用されます。

独自の IoT Agent を作成するプロセスは比較的簡単です。これは、必要なデータ転送を使用する IoT Agent を選択し、
ペイロード処理コードを書き換え/修正して、問題のペイロードを処理することにより、最もよく達成されます。

このチュートリアルでは、既存の Ultralight IoT Agent のコードを修正して、同様のカスタム XML 形式を処理します。
2つの IoT Agent の直接比較を以下に示します:

| IoT Agent for Ultralight                                             | IoT Agent for JSON                                                                                  | プロトコルの関心領域       |
| -------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------- | -------------------------- |
| 測定値のサンプル `c\|1`                                              | 測定値のサンプル `<measure device="lamp002" key="xxx">`<br/>&nbsp;`<c value="1"/>`<br/>`</measure>` | メッセージ・ペイロード     |
| コマンドのサンプル `Robot1@turn\|left=30`                            | コマンドのサンプル `<turn device="Robot1">`<br/>&nbsp;`<left>30</left>`<br/>`</turn>`               | メッセージ・ペイロード     |
| Content Type は `text/plain`                                         | Content Type は `application/xml`                                                                   | メッセージ・ペイロード     |
| 3つのトランスポートを提供 - HTTP, MQTT and AMPQ                      | 3つのトランスポートを提供 - HTTP, MQTT and AMPQ                                                     | トランスポート・メカニズム |
| HTTP はデフォルトで `iot/d` の測定値をリッスン                       | HTTP はデフォルトで `iot/xml` の測定値をリッスン                                                    | トランスポート・メカニズム |
| HTTP デバイスはパラメータ `?i=XXX&k=YYY` によって識別                | HTTP デバイスはペイロード `<measure device="XXX" key="YYY">` によって識別                           | デバイスの識別             |
| 既知の URL にポストされた HTTP コマンド - レスポンスはリプライにある | 既知の URL にポストされた HTTP コマンド - レスポンスはリプライにある                                | 通信ハンドシェイク         |
| MQTT デバイスはトピック `/XXX/YYY` のパスによって識別                | MQTT デバイスはトピック `/XXX/YYY` のパスによって識別されます                                       | デバイスの識別             |
| `cmd` トピックに投稿された MQTT コマンド                             | `cmd`トピックに投稿された MQTT コマンド                                                             | 通信ハンドシェイク         |
| `cmdexe` トピックに投稿された MQTT コマンド・レスポンス              | `cmdexe` トピックに投稿された MQTT コマンド                                                         | 通信ハンドシェイク         |

ご覧のとおり、サポートされている通信トランスポート (HTTP, MQTT, AMPQ) は同じままです。これは、XML デバイスが IoT Agent
と通信できるように適応させる必要があるカスタム・ペイロードの処理です。

ユースケースによっては、通信用に追加のミドルウェアを作成する必要がある場合もあります。この例では、_devices_ は、
2つの別個の通信チャネルで直接測定値を送信し、コマンドをリッスンおよびレスポンスすることができます。
[LoRaWAN](https://fiware-lorawan.readthedocs.io) および [OPC-UA](https://iotagent-opcua.readthedocs.io) IoT Agent
内では別のパラダイムが使用され、HTTP ミドルウェアが IoT Agent にレスポンスし、デバイスが使用する低レベルの CoAP
トランスポートに通信を変換します。

<a name="the-teaching-goal-of-this-tutorial"/>

## このチュートリアルの目標

このチュートリアルの目的は、独自のカスタム IoT Agent を作成する方法について開発者の理解を深めることです。一連の簡単な
変更が Ultralight IoT Agent のコードに加えられ、変更方法が示されています。このチュートリアルは、関連するコードの
ウォークスルーと、新しい IoT Agent に接続するための一連の HTTP リクエストで構成されています。コードは、現在、
[GitHub リポジトリ](https://github.com/FIWARE/tutorials.Custom-IoT-Agent/tree/master/iot-agent) にあります。

<a name="reusing-common-functionality"/>

## 共通機能の再利用

既存の IoT Agent を変更する利点は、開発者がすべての IoT Agent にある共通の機能を再利用できることです。これには、
次のような機能が含まれます:

-   デバイスの更新をリッスンするための標準的な場所を提供
-   コンテキスト・データの更新をリッスンする標準の場所を提供
-   デバイスのリストを保持し、コンテキスト・データ属性をデバイス構文にマッピング
-   セキュリティ認証

この基本機能は、共通の [IoT Agent framework library](https://iotagent-node-lib.readthedocs.io/) に抽象化されています。

#### デバイス・モニタ

このチュートリアルのために、一連のダミー IoT デバイスが作成され、Context Broker に接続されます。使用されている
アーキテクチャとプロトコルの詳細は、[IoT センサのチュートリアル](https://github.com/FIWARE/tutorials.IoT-Sensors/tree/NGSI-v2)
をご覧ください。各デバイスの状態は、JSON デバイス・モニタの 次の Web ページで確認できます:
`http://localhost:3000/device/monitor`

![FIWARE Monitor](https://fiware.github.io/tutorials.Custom-IoT-Agent/img/device-monitor.png)

<a name="architecture"/>

# アーキテクチャ

このアプリケーションは、[以前のチュートリアル](https://github.com/FIWARE/tutorials.Subscriptions/) で作成された
コンポーネントに基づいて構築されています。これは、1つの FIWAREコンポーネント
[Orion Context Broker](https://fiware-orion.readthedocs.io/en/latest/) と **Custom IoT Agent for XML** を使用します。
アプリケーションが _“Powered by FIWARE”_ として認定されるには、Orion Context Broker の使用で十分です。Orion Context
Broker と IoT Agent の両方が、オープンソースの [MongoDB](https://www.mongodb.com/) テクノロジを利用して、
保持している情報の永続性を維持しています。[以前のチュートリアル](https://github.com/FIWARE/tutorials.IoT-Sensors/)
で作成されたダミー IoT デバイスも使用しますが、カスタム XML メッセージング形式に応答するようにすでに調整されています。

したがって、全体的なアーキテクチャは次の要素で構成されます:

-   FIWARE [Orion Context Broker](https://fiware-orion.readthedocs.io/en/latest/) は
    [NGSI-v2](https://fiware.github.io/specifications/OpenAPI/ngsiv2) を使用してリクエストを受信します
-   **Custom IoT Agent for XML** は [NGSI-v2](https://fiware.github.io/specifications/OpenAPI/ngsiv2)
    を使用してサウスバウンド・リクエストを受信し、デバイスの XML コマンドに変換します
-   基礎となる [MongoDB](https://www.mongodb.com/) データベース:
    -   **Orion Context Broker** がデータ・エンティティ、サブスクリプション、レジストレーションなどの
        コンテキスト・データ情報を保持するために使用
    -   **IoT Agent** がデバイスの URL やキーなどのデバイス情報を保持するために使用
-   HTTP で実行されているカスタム XML メッセージング・プロトコルを使用する
    [ダミー IoT デバイス](https://github.com/FIWARE/tutorials.IoT-Sensors/tree/NGSI-v2)のセットとして機能する Web サーバ

要素間のすべての対話は HTTP リクエストによって開始されるため、エンティティをコンテナ化し、
公開されたポートから実行できます。

![](https://fiware.github.io/tutorials.Custom-IoT-Agent/img/architecture.png)

IoT デバイスと IoT Agent を接続するために必要な構成情報は、関連する `docker-compose.yml` ファイルの
services セクションで確認できます。

<a name="dummy-iot-devices-configuration"/>

## ダミー IoT デバイスの構成

```yaml
tutorial:
    image: fiware/tutorials.context-provider
    hostname: iot-sensors
    container_name: fiware-tutorial
    networks:
        - default
    expose:
        - "3000"
        - "3001"
    ports:
        - "3000:3000"
        - "3001:3001"
    environment:
        - "DEBUG=tutorial:*"
        - "PORT=3000"
        - "IOTA_HTTP_HOST=iot-agent"
        - "IOTA_HTTP_PORT=7896"
        - "DUMMY_DEVICES_PORT=3001"
        - "DUMMY_DEVICES_API_KEY=4jggokgpepnvsb2uv4s40d59ov"
        - "DUMMY_DEVICES_TRANSPORT=HTTP"
        - "DUMMY_DEVICES_PAYLOAD=XML"
```

`tutorial` コンテナは2つのポートでリッスンしています:

-  ポート `3000` が公開されているので、ダミー IoTデ バイスを表示する Web ページを見ることができます
-  ポート `3001` は、純粋にチュートリアル・アクセスのために公開されています。そのため、cUrl または Postman は、
   同じネットワークに属さなくても JSON コマンドを実行できます

`tutorial` コンテナは、次のように環境変数によって駆動されます:

| キー                    | 値                           | 説明                                                                                                                                         |
| ----------------------- | ---------------------------- | ---------------------------------------------------------------------------------------------------------------------------------- |
| DEBUG                   | `tutorial:*`                 | ロギングに使用されるデバッグ・フラグ                                                                                                         |
| WEB_APP_PORT            | `3000`                       | ダミー IoT デバイスのデータを表示する Web アプリが使用するポート                                                                             |
| IOTA_HTTP_HOST          | `iot-agent`                  | JSON の IoT Agent のホスト名。以下を参照してください                                                                                         |
| IOTA_HTTP_PORT          | `7896`                       | JSON 用の IoT Agent がリッスンするポート。`7896` は JSON over HTTP の一般的なデフォルトです                                                  |
| DUMMY_DEVICES_PORT      | `3001`                       | コマンドを受信するために ダミー IoT デバイスが使用するポート                                                                                 |
| DUMMY_DEVICES_API_KEY   | `4jggokgpepnvsb2uv4s40d59ov` | IoT インタラクションに使用される ランダム・セキュリティ・キー。デバイスと IoT Agent 間のインタラクションの完全性を保証するために使用されます |
| DUMMY_DEVICES_TRANSPORT | `HTTP`                       | ダミー IoT デバイスで使用されるトランスポート・プロトコル                                                                                    |
| DUMMY_DEVICES_PAYLOAD   | `XML`                        | ダミー IoT デバイスによるメッセージ・ペイロード・プロトコル                                                                                  |

YAML ファイルに記述されている他の `tutorial` コンテナ設定値は、このチュートリアルでは使用されません。

<a name="custom-xml-iot-agent-configuration"/>

## カスタム XML IoT Agent の構成

カスタム XML IoT Agent のコードは、このチュートリアルに関連付けられている
[GitHub リポジトリ](https://github.com/FIWARE/tutorials.Custom-IoT-Agent/tree/master/iot-agent)にあります。これは、
IoT Agent for Ultralight の 1.12.0 バージョンのコピーであり、以下に説明するように少し変更されています。関連する
[Dockerfile](https://github.com/FIWARE/tutorials.Custom-IoT-Agent/blob/master/iot-agent/Dockerfile) は、Node.js
を実行している Docker コンテナ内の適切な場所にコードをコピーするだけです。これにより、`docker-compose.yaml`
ファイルを使用してコンポーネントをインスタンス化できます。必要な設定は次のとおりです:

```yaml
iot-agent:
    image: fiware/iotagent-xml
    build:
        context: iot-agent
        dockerfile: Dockerfile
    hostname: iot-agent
    container_name: fiware-iot-agent
    depends_on:
        - mongo-db
    networks:
        - default
    expose:
        - "4041"
        - "7896"
    ports:
        - "4041:4041"
        - "7896:7896"
    environment:
        - IOTA_CB_HOST=orion
        - IOTA_CB_PORT=1026
        - IOTA_NORTH_PORT=4041
        - IOTA_REGISTRY_TYPE=mongodb
        - IOTA_LOG_LEVEL=DEBUG
        - IOTA_TIMESTAMP=true
        - IOTA_CB_NGSI_VERSION=v2
        - IOTA_AUTOCAST=true
        - IOTA_MONGO_HOST=mongo-db
        - IOTA_MONGO_PORT=27017
        - IOTA_MONGO_DB=iotagentjson
        - IOTA_HTTP_PORT=7896
        - IOTA_PROVIDER_URL=http://iot-agent:4041
        - IOTA_DEFAULT_RESOURCE=/iot/xml
```

`iot-agent` コンテナは Orion Context Broker の存在に依存し、MongoDB データベースを使用してデバイスの URL やキーなどの
デバイス情報を保持します。コンテナは2つのポートでリッスンしています:

-   ポート `7896` が公開され、ダミー IoT デバイスから HTTP 経由で JSON 測定値を受信します
-   ポート `4041`は、純粋にチュートリアル・アクセス用に公開されています。そのため、cUrl または Postman は、
    同じネットワークに属さなくてもプロビジョニング・コマンドを実行できます

`iot-agent` コンテナは、次のように環境変数によって駆動されます:

| キー                  | 値                      | 説明                                                                                                                                              |
| --------------------- | ----------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------- |
| IOTA_CB_HOST          | `orion`                 | コンテキストを更新する Context Broker のホスト名                                                                                                  |
| IOTA_CB_PORT          | `1026`                  | Context Broker がコンテキストを更新するためにリッスンするポート                                                                                   |
| IOTA_NORTH_PORT       | `4041`                  | IoT Agent の構成と Context Broker からのコンテキスト更新の受信に使用されるポート                                                                  |
| IOTA_REGISTRY_TYPE    | `mongodb`               | IoT デバイス情報をメモリまたはデータベースのどちらに保持するか                                                                                    |
| IOTA_LOG_LEVEL        | `DEBUG`                 | IoT Agent のログレベル                                                                                                                            |
| IOTA_TIMESTAMP        | `true`                  | 接続されたデバイスから受信した各測定でタイムスタンプ情報を提供するかどうか                                                                        |
| IOTA_CB_NGSI_VERSION  | `v2`                    | アクティブな属性の更新を送信するときに NGSI v2 を使用するかどうか                                                                                 |
| IOTA_AUTOCAST         | `true`                  | JSON の数値が文字列ではなく数値として読み取られるようにする                                                                                       |
| IOTA_MONGO_HOST       | `context-db`            | mongo DB のホスト名。デバイス情報を保持するために使用されます                                                                                     |
| IOTA_MONGO_PORT       | `27017`                 | mongoDB がリッスンしているポート                                                                                                                  |
| IOTA_MONGO_DB         | `iotagentjson`          | mongoDB で使用されるデータベースの名前                                                                                                            |
| IOTA_HTTP_PORT        | `7896`                  | IoT Agent がHTTP 経由で IoT デバイスのトラフィックをリッスンするポート                                                                            |
| IOTA_PROVIDER_URL     | `http://iot-agent:4041` | コマンドの登録時に Context Broker に渡される URL。ContextBroker がデバイスにコマンドを発行するときにフォワーディング URL の場所として使用されます |
| IOTA_DEFAULT_RESOURCE | `/iot/xml`              | IoT Agent がカスタム XML 測定値のリッスンに使用するデフォルトパス                                                                                 |

<a name="prerequisites"/>

# 前提条件

<a name="docker"/>

## Docker

シンプルにするために、すべてのコンポーネントは [Docker](https://www.docker.com) を使用して実行されます。**Docker**
は、それぞれの環境に分離されたさまざまなコンポーネントを可能にするコンテナ・テクノロジです

-   Windows に Docker をインストールするには、[こちら](https://docs.docker.com/docker-for-windows/)の指示に
従ってください
-   Mac に Docker をインストールするには、[こちら](https://docs.docker.com/docker-for-mac/)の指示に従ってください
-   Linux に Docker をインストールするには、[こちら](https://docs.docker.com/install/)の指示に従ってください

**Docker Compose** は、マルチ・コンテナの Docker アプリケーションを定義して実行するためのツールです。
[YAMLファイル](https://raw.githubusercontent.com/Fiware/tutorials.Entity-Relationships/master/docker-compose.yml)
を使用して、アプリケーションに必要なサービスを構成します。つまり、すべてのコンテナ・サービスを1つのコマンドで
起動できます。Docker Compose は、デフォルトで Docker for Windows および Docker for Mac の一部としてインストール
されますが、Linux ユーザは[こちら](https://docs.docker.com/compose/install/) にある指示に従う必要があります。

次のコマンドを使用して、現在の **Docker** および **Docker Compose** のバージョンを確認できます:

```console
docker-compose -v
docker version
```

Docker version 18.03 以降と Docker Compose 1.21 以降を使用していることを確認し、
必要に応じてアップグレードしてください。

<a name="cygwin"/>

## Cygwin

簡単な bash スクリプトを使用してサービスを起動します。Windows ユーザは、[cygwin](http://www.cygwin.com/) を
ダウンロードして、Windows 上の Linux ディストリビューションと同様のコマンドライン機能を提供する必要があります。

<a name="start-up"/>

# 起動

開始する前に、必要な Docker イメージをローカルで取得またはビルドしたことを確認する必要があります。
以下のコマンドを実行して、リポジトリのクローンを作成し、必要なイメージを作成してください:

```console
git clone https://github.com/FIWARE/tutorials.Custom-IoT-Agent.git
cd tutorials.Custom-IoT-Agent
git checkout NGSI-v2

./services create
```

その後、リポジトリ内で提供される [services](https://github.com/FIWARE/tutorials.IoT-Agent/blob/NGSI-v2/services) Bash
スクリプトを実行することにより、コマンドラインからすべてのサービスを初期化できます:

```console
./services start
```

> :information_source: **注:** クリーンアップしてやり直したい場合は、次のコマンドで実行できます:
>
> ```console
> ./services stop
> ```

<a name="creating-a-custom-iot-agent"/>

# カスタム IoT Agent の作成

次のセクションは、IoT Agent をプロビジョニングし、測定値を受信してコマンドを送信するために使用される一連の HTTP
コマンドで構成されています。Custom IoT Agent 内の関連する修正されたコードは、各アクションが処理されるときに説明します。

チュートリアルを正しく実行するには、ブラウザーでデバイス・モニタのページを利用できることを確認し、ページをクリックして
オーディオを有効にしてから、cUrl コマンドを入力してください。デバイス・モニタは、XML 構文を使用してダミー IoT
デバイスの配列の現在の状態を表示します。

#### デバイス・モニタ (Device Monitor)

デバイス・モニタは次の場所にあります: `http://localhost:3000/device/monitor`

<a name="checking-the-iot-agent-service-health"/>

### IoT Agent サービス の健全性の確認

公開されたポートに HTTP リクエストを送信することで、IoT Agent が実行されているかどうかを確認できます:

#### :one: リクエスト:

```console
curl -X GET \
  'http://localhost:4041/iot/about'
```

レスポンスは次のようになります:

```json
{
    "libVersion": "2.6.0-next",
    "port": "4041",
    "baseRoot": "/",
    "version": "1.12.0-next"
}
```

これは、IoT Agent Node.js ライブラリから直接提供される標準機能であり、コードの変更は必要ありません。

<a name="provisioning-a-service-group"/>

### サービス・グループのプロビジョニング

各プロビジョニングで常に認証キーを提供する必要があり、IoT Agent は最初に Context Broker がレスポンスしている URL
を認識しないため、グループ・プロビジョニングの呼び出しは常にデバイス接続の最初のステップです。

すべての匿名デバイスに対してデフォルトのコマンドと属性を設定することもできますが、このチュートリアルでは、
各デバイスを個別にプロビジョニングするため、これは行いません。

この例では、デバイスの匿名グループをプロビジョニングします。一連のデバイスがメッセージを `IOTA_HTTP_PORT`
に送信することを IoT Agent に通知します (ここで、IoT Agent は **Northbound** 通信をリッスンします)

#### :two: リクエスト:

```console
curl -iX POST \
  'http://localhost:4041/iot/services' \
  -H 'Content-Type: application/json' \
  -H 'fiware-service: openiot' \
  -H 'fiware-servicepath: /' \
  -d '{
 "services": [
   {
     "apikey":      "4jggokgpepnvsb2uv4s40d59ov",
     "cbroker":     "http://orion:1026",
     "entity_type": "Thing",
     "resource":    "/iot/xml"
   }
 ]
}'
```

この例では、IoT Agent に、`/iot/xml` エンドポイントが使用され、デバイスがトークン `4jggokgpepnvsb2uv4s40d59ov`
を含めることで自身を認証することが通知されます。Custom XML IoT Agent の場合、これはデバイスが POST
リクエストを以下に送信することを意味します:

```
http://iot-agent:7896/iot/xml
```

ここで `<measure>` は関連するデバイス ID と API キーを保持します。

```
<measure device="motion001" key="4jggokgpepnvsb2uv4s40d59ov">
    <c value="3"/>
</measure>
```

この構文は、デバイス ID と API キーが URL パラメータとして送信される Ultralight IoT Agent とは異なります。

<h3>Reading XML - Analysing the Code</h3>

関連する変更は、XML パーサーがインスタンス化される `HTTPBindings.js` ファイルにあります。

```javascript
const xmlBodyParser = require("express-xml-bodyparser");
```

```javascript
httpBindingServer.router.post(
    config.getConfig().iota.defaultResource || constants.HTTP_MEASURE_PATH,
    ...xmlBodyParser({ trim: false, explicitArray: false }),
    checkMandatoryParams(false),
    ...etc
);
```

つまり、角括弧構文を使用して、XML リクエストの属性にアクセスできます。`apiKey` と `deviceId` は両方とも
必須パラメータであるため、受信した `<measure>` 内で見つけることができます。

```javascript
function checkMandatoryParams(queryPayload) {
    return function(req, res, next) {
        var notFoundParams = [],
            error;

        req.apiKey = req.body["measure"]["$"]["key"];
        req.deviceId = req.body["measure"]["$"]["device"];

        if (!req.apiKey) {
            notFoundParams.push("API Key");
        }

        if (!req.deviceId) {
            notFoundParams.push("Device Id");
        }

        // CHeck if retrievingParam
        if (queryPayload && !req.query.d && req.query.getCmd !== "1") {
            notFoundParams.push("Payload");
        }

        if (req.method === "POST" && !req.is("application/xml")) {
            error = new errors.UnsupportedType("application/xml");
        }

        if (notFoundParams.length !== 0) {
            next(new errors.MandatoryParamsNotFound(notFoundParams));
        } else {
            next(error);
        }
    };
}
```

この関数は、適切な MIME タイプが受信されたことを確認し、着信メッセージに十分な情報が含まれていない場合はすぐに
失敗します。

<a name="provisioning-an-actuator"/>

### センサのプロビジョニング

エンティティを作成するときは、NGSI-LD
[仕様](https://www.etsi.org/deliver/etsi_gs/CIM/001_099/009/01.03.01_60/gs_cim009v010301p.pdf)
に従って URN を使用するのが一般的です。さらに、データ属性を定義するときに、意味のある名前を理解しやすくなります。
これらのマッピングは、デバイスを個別にプロビジョニングすることで定義できます。

3種類の測定属性をプロビジョニングできます:

-   `attributes` はデバイスからのアクティブな読み取り値です
-   `lazy` 属性はリクエスト時にのみ送信されます。IoT Agent は測定値を返すようにデバイスに通知します
-   `static_attributes` は、その名前が Context Broker に渡されるデバイスに関する静的データ (リレーションシップなど)
    を示唆しているものです

> **注**: 個々の `id` が必要ない場合、または集約されたデータで十分な場合、`attributes`
> は個別ではなくプロビジョニング・サービス内で定義できます。

#### :three: リクエスト:

```console
curl -iX POST \
  'http://localhost:4041/iot/devices' \
  -H 'Content-Type: application/json' \
  -H 'fiware-service: openiot' \
  -H 'fiware-servicepath: /' \
  -d '{
 "devices": [
   {
     "device_id":   "motion001",
     "entity_name": "urn:ngsi-ld:Motion:001",
     "entity_type": "Motion",
     "timezone":    "Europe/Berlin",
     "attributes": [
       { "object_id": "c", "name": "count", "type": "Integer" }
     ],
     "static_attributes": [
       { "name":"refStore", "type": "Relationship", "value": "urn:ngsi-ld:Store:001"}
     ]
   }
 ]
}
'
```

予想通り、元の Ultralight IoT Agent と同じ HTTP トランスポートを使用しているため、**デバイスをプロビジョニングする**
HTTP コマンドは、基になるペイロードまたはトランスポート・プロトコルに基づいて変更されません。`internal_atttributes`
を使用して、必要に応じて Custom IoT Agent の追加情報を提供できます。リクエストでは、デバイス `motion001` を URN
`urn:ngsi-ld:Motion:001` に関連付け、デバイスの `c` をコンテキスト属性 `count` (`Integer` として定義されている)
にマッピングしています。`refStore` は `static_attribute` として定義され、デバイスを **Store**
`urn:ngsi-ld:Store:001` 内に配置します。

次の XML リクエストを行うことで、**Motion Sensor** デバイス `motion001` からのダミー IoT デバイスの測定を
シミュレートできます。

#### :four: リクエスト:

```console
curl -L -X POST 'http://localhost:7896/iot/xml' \
-H 'Content-Type: application/xml' \
--data-raw '<measure device="motion001" key="4jggokgpepnvsb2uv4s40d59ov">
    <c value="3"/>
</measure>'
```

<h3>測定値の読み取り - コードの分析</h3>

ペイロードと `Content-Type` の両方が更新されました。ダミー IoT デバイスは、ドアがロックされていないときに、
以前のチュートリアルで同様の Ultralight リクエストを行いました。各 Motion sensor の状態が変化し、
ノースバウンド・リクエストがデバイス・モニタに記録されます。

これで IoT Agent が接続され、サービス・グループは IoT Agent がリッスンするリソース (`iot/xml`) を定義し、
リクエストの認証に使用される API キー (`4jggokgpepnvsb2uv4s40d59ov`) が本文に見つかりました。
これらは両方とも認識されているため、測定値は有効です。

次のステップは、ペイロードを解析して属性を抽出することです。これは `xmlparser.js` ファイルの修正された
`parse` メソッドにあります。

```javascript
function parse(payload) {
    let result = [];
    const keys = Object.keys(payload["measure"]);
    for (let i = 0; i < keys.length; i++) {
        if (keys[i] !== "$") {
            let obj = {};
            obj[keys[i]] = payload["measure"][keys[i]]["$"].value;
            result.push(obj);
        }
    }
    return result;
}
```

`parse()` は、キーと値のペアの JSON 配列を返します。これは、デバイスの属性名 (`c`など) からエンティティの属性名 (`count`
など) にマッピングできます。明らかに、マッピングは元のプロビジョニングで送信された値に基づいています。

Context Broker からエンティティ・データを取得すると、測定値が記録されていることがわかります。`fiware-service` および
`fiware-service-path` ヘッダを追加することを忘れないでください。

#### :five: リクエスト:

```console
curl -X GET \
  'http://localhost:1026/v2/entities/urn:ngsi-ld:Motion:001?type=Motion' \
  -H 'fiware-service: openiot' \
  -H 'fiware-servicepath: /'
```

#### レスポンス:

```json
{
    "id": "urn:ngsi-ld:Motion:001",
    "type": "Motion",
    "TimeInstant": {
        "type": "ISO8601",
        "value": "2018-05-25T10:51:32.00Z",
        "metadata": {}
    },
    "count": {
        "type": "Integer",
        "value": "3",
        "metadata": {
            "TimeInstant": {
                "type": "ISO8601",
                "value": "2018-05-25T10:51:32.646Z"
            }
        }
    },
    "refStore": {
        "type": "Relationship",
        "value": "urn:ngsi-ld:Store:001",
        "metadata": {
            "TimeInstant": {
                "type": "ISO8601",
                "value": "2018-05-25T10:51:32.646Z"
            }
        }
    }
}
```

レスポンスは `id=motion001` の **Motion Sensor** デバイスが IoT Agent によって正常に識別され、エンティティ
`id=urn:ngsi-ld:Motion:001` にマップされたことを示しています。この新しいエンティティは、コンテキスト・データ内に
作成されています。ダミー IoT デバイスの測定リクエストの `c` 属性は、コンテキスト・データ内のより意味のある `count`
属性にマッピングされています。

### アクチュエータのプロビジョニング

アクチュエータのプロビジョニングは、センサのプロビジョニングに似ています。今回、 `endpoint` 属性は、IoT Agent が
JSON コマンドを送信する必要がある場所を保持し、`commands` 配列には、呼び出すことができる各コマンドのリストが
含まれています。以下の例では、 `deviceId=bell001` でベルをプロビジョニングします。エンドポイントは
`http://iot-sensors:3001/iot/bell001` で、`ring` コマンドを受け入れることができます。`transport=HTTP`
属性は、使用する通信プロトコルを定義します。

#### :six: リクエスト:

```console
curl -iX POST \
  'http://localhost:4041/iot/devices' \
  -H 'Content-Type: application/json' \
  -H 'fiware-service: openiot' \
  -H 'fiware-servicepath: /' \
  -d '{
  "devices": [
    {
      "device_id": "bell001",
      "entity_name": "urn:ngsi-ld:Bell:001",
      "entity_type": "Bell",
      "transport": "HTTP",
      "endpoint": "http://iot-sensors:3001/iot/bell001",
      "commands": [
        { "name": "ring", "type": "command" }
       ],
       "static_attributes": [
         {"name":"refStore", "type": "Relationship","value": "urn:ngsi-ld:Store:001"}
      ]
    }
  ]
}
'
```

センサのプロビジョニングの場合と同様に、このリクエストは変更されません。IoT Agent の構造内で暗黙的に、
コマンドのプロビジョニングは次の暗黙の契約を満たします:

1.  Custom IoT Agent が属性を登録します
2.  Custom IoT Agent が `/v2/op/update` エンドポイントで、コンテキストを更新するための各リクエストをします
3. リクエストの処理方法が決定されます。Custom IoT Agent と Ultralight Agent の両方で、これは `<command>State`
   属性を設定し、`/cmd` エンドポイントでリクエストを修正して、デバイス (またはデバイスの責任を負うミドルウェア)
   にフォワーディングするというパラダイムに従います

最初の2つの項目は Context Broker からのコンテキスト変更のリスニングは、明確に定義された NGSI 構文に従うため、
すべての IoT Agent に共通です。ただし、3番目の項目は、継続的に消費するメッセージを準備するために何をすべきかは、
抽象化されているプロトコルによって異なります。

Context Broker を接続する前に、`/v2/op/update` エンドポイントを使用して IoT Agent のノースポートに直接 REST
リクエストを送信することにより、コマンドがデバイスに送信できることをテストできます。接続すると、最終的に
Context Broker によって呼び出されるのは、このエンドポイントです。
構成をテストするには、次のようにコマンドを直接実行します:

#### :seven: リクエスト:

```console
curl -iX POST \
  http://localhost:4041/v2/op/update \
  -H 'Content-Type: application/json' \
  -H 'fiware-service: openiot' \
  -H 'fiware-servicepath: /' \
  -d '{
    "actionType": "update",
    "entities": [
        {
            "type": "Bell",
            "id": "urn:ngsi-ld:Bell:001",
            "ring" : {
                "type": "command",
                "value": ""
            }
        }
    ]
}'
```

デバイス・モニタのページを表示している場合は、ベルの変化の状態も確認できます。

![](https://fiware.github.io/tutorials.Custom-IoT-Agent/img/bell-ring.gif)

ベルを鳴らすコマンドの結果は、Orion Context Broker 内のエンティティをクエリすることで読み取ることができます。

<h3>コマンドの読み取り - コードの分析</h3>

Custom IoT Agent 内で、`start()` 関数は、リクエストが Context Broker から到着したときに起動する一連のハンドラ関数を
設定します。

```javascript
iotAgentLib.setProvisioningHandler(deviceProvisioningHandler);
iotAgentLib.setConfigurationHandler(configurationHandler);
iotAgentLib.setCommandHandler(commandHandler);
iotAgentLib.setDataUpdateHandler(updateHandler);
```

これは適切なトランスポート・バインディングに渡され、この場合、`HTTPBindings.js` 内の `commandHandler()`
メソッドが呼び出されます。HTTP エラー・ハンドラを提供しますが、コマンドを作成してデバイスに送信する実際の作業を
`generateCommandExecution()` に委任します。

```javascript
function generateCommandExecution(apiKey, device, attribute) {
...
    const options = {
        url: device.endpoint,
        method: 'POST',
        body: xmlParser.createCommandPayload(device, cmdName, cmdAttributes),
        headers: {
            'fiware-service': device.service,
            'fiware-servicepath': device.subservice
        }
    };
... etc
```

ペイロード自体、つまりデバイスで解釈できるようにコマンドを作成する方法は、カスタム XML
メッセージング・プロトコルに固有であり、 `xmlParser.js` の `createCommandPayload()` メソッドで生成されます。

```javascript
function createCommandPayload(device, command, attributes) {
    if (typeof attributes === "object") {
        let payload = "<" + command + '  device="' + device.id + '">';

        Object.keys(attributes).forEach(function(key, value) {
            payload = payload + "<" + key + ">" + value + "</" + key + ">";
        });
        payload = payload + "</" + command + ">";
        return payload;
    } else {
        return "<" + command + '  device="' + device.id + '"/>';
    }
}
```

これは Ultralight プロトコルの修正であり、 `@` および `|` 記号が Ultralight デバイス用に生成されます。

ただし、ペイロードの作成はジョブの半分にすぎず、デバイスに送信して理解する必要があるため、明確に定義された
通信ハンドシェイクを使用して通信を完了する必要があります。そのため、ペイロードを生成した後、 `HTTPBindings.js`
の `sendXMLCommandHTTP() ` メソッドがメッセージを送信し、レスポンスを `xmlParser.js` の `result()`
メソッドに渡して、デバイスからのコマンド・レスポンスを解釈します。

```javascript
function result(payload) {
    const xmlToJson = require("xml-parser");
    const data = xmlToJson(payload);
    const result = {};
    result.deviceId = data.root.attributes.device;
    result.command = data.root.attributes.command;
    result.result = data.root.name;

    return result;
}
```

最後に、コマンドの成功または失敗は、IoT Agent node ライブラリの共通コードを使用して Context Broker に更新されます。

IoT Agents の典型であるように、ペイロードの作成と通信のハンドシェイクは、メンテナンスを容易にするために2つの個別の
問題に分割されています。したがって、今回のケースではペイロードのみが変更されているため、カスタムのユースケースを
満たすために変更が必要なのは、コードの XML ペイロード側のみです。

アクチュエータ・コマンドの結果は、標準の NGSI コマンドを使用して Context Broker で読み取ることができます。

#### :eight: リクエスト:

```console
curl -X GET \
  'http://localhost:1026/v2/entities/urn:ngsi-ld:Bell:001?type=Bell&options=keyValues' \
  -H 'fiware-service: openiot' \
  -H 'fiware-servicepath: /'
```

#### レスポンス:

```json
{
    "id": "urn:ngsi-ld:Bell:001",
    "type": "Bell",
    "TimeInstant": "2018-05-25T20:06:28.00Z",
    "refStore": "urn:ngsi-ld:Store:001",
    "ring_info": " ring OK",
    "ring_status": "OK",
    "ring": ""
}
```

`ring` コマンドの結果は `ring_info` 属性の値で確認できます。

Custom IoT Agent を開発すると、ユーザは標準の NGSI リクエストを Context Broker に送信するだけでデバイスを
作動させることができますが、低レベルの基礎となるプロトコルは不明であり、IoT Agent に正常に抽象化されます。

# 次のステップ

高度な機能を追加することで、アプリケーションに複雑さを加える方法を知りたいですか
？このシリーズ
の[他のチュートリアル](https://www.letsfiware.jp/fiware-tutorials)を読むことで見
つけることができます :

---

## License

[MIT](LICENSE) © 2020 FIWARE Foundation e.V.
