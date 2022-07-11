---
title: Quick Start Guide for PingCAP Clinic
summary: Learn how to use PingCAP Clinic to collect, upload, and view cluster diagnosis data quickly.
---

# PingCAPクリニックのクイックスタートガイド {#quick-start-guide-for-pingcap-clinic}

このドキュメントでは、 PingCAPクリニック診断サービス（PingCAPクリニック）を使用して、クラスタ診断データをすばやく収集、アップロード、および表示する方法について説明します。

PingCAPクリニックは、Diagクライアント（Diagとして短縮）と[クリニックサーバークラウドサービス](https://clinic.pingcap.com.cn) （Clinic Serverとして短縮）の2つのコンポーネントで構成されています。 2つのコンポーネントの詳細については、 [PingCAPクリニックの概要](/clinic/clinic-introduction.md)を参照してください。

クラスタに問題がある場合、PingCAPテクニカルサポートに連絡する必要がある場合は、リモートトラブルシューティングを容易にするために次の操作を実行できます。Diagで診断データを収集し、収集したデータをClinic Serverにアップロードし、データアクセスリンクをテクニカルサポートスタッフ。

PingCAPクリニックは現在テクニカルプレビュー段階にあります。

> **ノート：**
>
> -   データを収集およびアップロードするための次の方法は、 [TiUPでデプロイされたクラスター](/production-deployment-using-tiup.md)にのみ適用されます。
> -   PingCAPクリニックによって収集された診断データは、クラスタの問題のトラブルシューティングに**のみ**使用されます。

## 前提条件 {#prerequisites}

PingCAPクリニックを使用する前に、Diagをインストールし、データをアップロードするための環境を準備する必要があります。

1.  TiUPがインストールされているコントロールマシンで、次のコマンドを実行してDiagをインストールします。

    {{< copyable "" >}}

    ```bash
    tiup install diag
    ```

2.  [クリニックサーバー](https://clinic.pingcap.com.cn)にログインし、[ **AskTUGでサインイン**]を選択して、AskTUGコミュニティのログインページに入ります。 AskTUGアカウントをお持ちでない場合は、ログインページで登録できます。

3.  クリニックサーバー上に組織を作成します。編成は、TiDBクラスターのコレクションです。作成した組織に診断データをアップロードできます。

4.  データをアップロードするためのアクセストークンを取得します。収集したデータをDiagを介してアップロードする場合、データを安全に分離するために、ユーザー認証用のトークンが必要です。クリニックサーバーからすでにトークンを取得している場合は、トークンを再利用できます。

    トークンを取得するには、[クラスター]ページの右下隅にあるアイコンをクリックし、[**診断ツールのアクセストークンの取得**]を選択します。次に、ポップアップウィンドウで[ <strong>+</strong> ]をクリックし、表示されたトークン情報をコピーして保存します。

    ![An example of a token](/media/clinic-get-token.png)

    > **ノート：**
    >
    > -   データセキュリティのために、TiDBはトークン情報が作成されたときにのみ表示します。情報を紛失した場合は、古いトークンを削除して新しいトークンを作成できます。
    > -   トークンは、データのアップロードにのみ使用されます。

5.  Diagでトークンを設定します。

    例えば：

    {{< copyable "" >}}

    ```bash
    tiup diag config clinic.token ${token-value}
    ```

6.  （オプション）ログの編集を有効にします。

    TiDBが詳細なログ情報を提供する場合、機密情報（ユーザーデータなど）をログに出力する場合があります。ローカルログとクリニックサーバーで機密情報が漏洩するのを防ぎたい場合は、TiDB側でログの編集を有効にすることができます。詳細については、 [ログ編集](/log-redaction.md#log-redaction-in-tidb-side)を参照してください。

## 手順 {#steps}

1.  Diagを実行して診断データを収集します。

    たとえば、現在の時刻に基づいて4時間前から2時間前までの診断データを収集するには、次のコマンドを実行します。

    {{< copyable "" >}}

    ```bash
    tiup diag collect ${cluster-name} -f="-4h" -t="-2h"
    ```

    コマンドを実行した後、Diagはすぐにデータの収集を開始しません。代わりに、Diagは、続行するかどうかを確認するために、出力に推定データサイズとターゲットデータストレージパスを提供します。データの収集を開始することを確認するには、 `Y`を入力します。

    収集が完了すると、Diagは収集されたデータが配置されているフォルダーパスを提供します。

2.  収集したデータをクリニックサーバーにアップロードします。

    > **ノート：**
    >
    > アップロードするデータ（収集されたデータを含むフォルダー）のサイズは、10GB**以下で**ある必要があります。そうしないと、データのアップロードが失敗します。

    -   クラスタが配置されているネットワークがインターネットにアクセスできる場合は、次のコマンドを使用して、収集されたデータを含むフォルダを直接アップロードできます。

        {{< copyable "" >}}

        ```bash
        tiup diag upload ${filepath}
        ```

        アップロードが完了すると、出力に`Download URL`が表示されます。

        > **ノート：**
        >
        > この方法でデータをアップロードする場合は、Diagv0.7.0以降のバージョンを使用する必要があります。実行すると、Diagバージョンを取得できます。 Diagのバージョンが0.7.0より前の場合は、 `tiup update diag`コマンドを使用してDiagを最新バージョンにアップグレードできます。

    -   クラスタが配置されているネットワークがインターネットにアクセスできない場合は、収集したデータをパックして、パッケージをアップロードする必要があります。詳細については、 [方法2.データをパックしてアップロードする](/clinic/clinic-user-guide-for-tiup.md#method-2-pack-and-upload-data)を参照してください。

3.  アップロードが完了したら、コマンド出力の`Download URL`からデータアクセスリンクを取得します。

    デフォルトでは、診断データには、クラスタ名、クラスタトポロジ情報、収集された診断データのログコンテンツ、および収集されたデータのメトリックに基づいて再編成されたGrafanaダッシュボード情報が含まれます。

    データを使用してクラスタの問題を自分でトラブルシューティングすることも、PingCAPテクニカルサポートスタッフにデータアクセスリンクを提供してリモートトラブルシューティングを容易にすることもできます。

## 次は何ですか {#what-s-next}

-   [PingCAPクリニックの概要](/clinic/clinic-introduction.md)
-   [PingCAPクリニックを使用したTiDBクラスターのトラブルシューティング](/clinic/clinic-user-guide-for-tiup.md)
-   [PingCAPクリニックの診断データ](/clinic/clinic-data-instruction-for-tiup.md)