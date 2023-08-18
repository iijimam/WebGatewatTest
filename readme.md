# WebGateway用コンテナのHTTPS化＋IRISへの接続サンプル

- ※1 サンプルでは製品版 iris:2023.1.1.380.0 を利用しています。（製品版利用時は [iris](/iris/)以下に iris.keyを配置する必要があります。ライセンスキーはコンテナビルド時に有効化されます。）
- ※2 コミュニティエディションに変更する場合は、[docker-compose.yml](docker-compose.yml) 19行目をコメント化してください。また、IRISの[Dockerfile](/iris/Dockerfile)のFROMを適切なイメージに変更してください。

## コンテナの開始

```
docker-compose up -d --build
```
Apacheコンテナ内の80ポートはホストの**8080**に、443は**8443**を使用するように記述しています。

サンプルの [docker-compose.yml](/docker-compose.yml) を修正せずにそのままコンテナを開始した場合は、管理ポータルへは以下のURLでアクセスできます。

通信方式|URL
--|--
http|http://localhost:8080/csp/sys/UtilHome.csp
https|https://localhost:8443/csp/sys/UtilHome.csp

ユーザ名：SuperUser、パスワード：SYS　でアクセスできます。

## コンテナ停止

```
docker-compose stop
```

## コンテナ削除
```
docker-compose down
```
___

**【WebGateway用コンテナについて補足】**

サンプルでは、下記コンテナを利用。コンテナ実行で80ポートのApacheが立ち上がる。

`containers.intersystems.com/intersystems/webgateway:2023.1.1.380.0`


## 1) WebGateway用コンテナのconfファイルを変更する

WebGateway用コンテナ起動時、IRIS用の設定が必要になるので、/ のパスが来たときApacheからWebゲートウェイに転送されるように設定を追加する。

追加内容は[CSP.conf](/webgateway/CSP.conf)の中身

> ※参考にしたページ：[https://github.com/intersystems-community/webgateway-examples](https://github.com/intersystems-community/webgateway-examples)


## 2) WebゲートウェイからIRISへ接続するための設定追加

（メモ：Apacheコンテナ立ち上げ後に手動でWebゲートウェイ管理画面でも設定できる　👉　[http://webサーバ名/csp/bin/Systems/Module.cxw](http://localhost:8080/csp/bin/Systems/Module.cxw)）

> このコンテナは80ポートをホストの**8080**に割り当てています。

CSP.iniファイルの中身を書き換えればよいので、書き換えの中身のみを [CSP.ini](/webgateway/CSP.ini)に用意

コンテナではCSP.iniファイルはWebゲートウェイのインストールディレクトリ以下に配置される。

ディレクトリは環境変数 **${ISC_PACKAGE_INSTALLDIR}** で確認できる。

```
root@318a6f20db77:/opt/webgateway# ls ${ISC_PACKAGE_INSTALLDIR}/bin/CSP.ini
/opt/webgateway/bin/CSP.ini
```

ここまでで、80（ホスト側は8080）でIRISへ接続を行う設定は完了。

＜メモ1＞
サンプルでは、接続先もコンテナのIRISを指している。

- 接続先IRISのホスト名：**iris4h**
- スーパーサーバーポート：**1972**
- アクセスユーザ名：**CSPSystem**
- パスワード：**SYS**
- CSP.iniに記載するIRISへの接続名は **[LOCAL]** を利用。サーバ名として設定した[LOCAL]に対して、**LOCAL=Enabled** の記述も必要
- **接続先IRISのCSPSystemのパスワードがサンプルと異なる場合は、 [CSP.ini](/webgateway/CSP.ini)20行目の変更が必要。**

＜メモ2＞
IRISコンテナ側では、コンテナビルド時事前定義ユーザに対するパスワードをSYSに固定としています。適宜変更お願いします。
>IRISコンテナビルド時に実行される[Dockerfile](/iris/Dockerfile) で実行している [iris.script](/iris/iris.script)内で、初回アクセス時の事前定義ユーザデフォルトパスワードの変更手続きを無効にしています。

___
以下、HTTPSで管理ポータルに接続したい場合の設定


## 3) HTTPS通信用に証明書ファイルを準備

サンプルではオレオレ証明書ファイルを準備
- [server.crt](/webgateway/server.crt)
- [server.key](/webgateway/server.key)

key はパスフレーズが聞かれないように変更したほうが良い。
> openssl rsa -in キーファイル -out キーファイル

## 4) ApacheのSSL化

[3) HTTPS通信用に証明書ファイルを準備](#3-https通信用に証明書ファイルを準備) のファイル適用のため、[docker-compose.yml](/docker-compose.yml) と [CSP.conf](/webgateway/CSP.conf) に専用の記述を追加

    ※docker-compose.ymlのvolumesの設定で、[webgateway](/webgateway/)以下に配置したキーやini、confファイルがコンテナ内の「webgateway-shared」に配置され、SSL化も[CSP.conf](/webgateway/CSP.conf)の通りに設定されます。

- [docker-compose.yml](/docker-compose.yml) のwebgwコンテナに2種の環境変数とvolumeの設定を行う

    ```
    environment:
    - ISC_CSP_CONF_FILE=/webgateway-shared/CSP.conf
    - ISC_CSP_INI_FILE=/webgateway-shared/CSP.ini
    volumes:
    - ./webgateway:/webgateway-shared
    ```

- [CSP.conf](/webgateway/CSP.conf)

    ```
    SSLCertificateFile "/webgateway-shared/server.crt"
    SSLCertificateKeyFile "/webgateway-shared/server.key"
    ```

あとはコンテナを開始するだけでhttpsでIRISの管理ポータルにアクセスできます。