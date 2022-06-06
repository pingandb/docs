---
title: TiDB Cloud Release Notes in 2020
summary: Learn about the release notes of TiDB Cloud in 2020.
---

# 2020年のTiDBクラウドリリースノート {#tidb-cloud-release-notes-in-2020}

このページには、2020年の[TiDBクラウド](https://en.pingcap.com/tidb-cloud/)のリリースノートがリストされています。

## 2020年12月30日 {#december-30-2020}

-   デフォルトのTiDBバージョンをv4.0.9にアップグレードします
-   TiDBでのアップグレードとスケーリングを適切にサポートして、クライアント障害をゼロにします
-   バックアップから新しいクラスタを復元した後、クラスタ構成を回復する

## 2020年12月16日 {#december-16-2020}

-   すべてのクラスタ層でTiDBノードの最小数を1つに調整します
-   SQLWebシェルでのシステムコマンドの実行を禁止する
-   デフォルトでTiDBクラスターの編集ログを有効にする

## 2020年11月24日 {#november-24-2020}

-   TiDBクラスターのパブリックエンドポイントのトラフィックフィルターIPリストを空にして、パブリックアクセスを無効にします
-   OutlookまたはHotmailを使用して顧客に送信される招待メールの配信率を向上させる
-   サインアップのためのエラー通知メッセージを磨く
-   新しいクラスターはUbuntuではなくCentOSVMで実行されます
-   対応するバックアップがまだ存在する場合に、クラスタがごみ箱に表示されない問題を修正します

## 2020年11月4日 {#november-4-2020}

-   組織名を変更する機能を実装する
-   データの復元中にユーザーがTiDBにアクセスできないようにする
-   サインアップページで利用規約とプライバシーの場所を更新する
-   フィードバックフォームの入り口ウィジェットを追加する
-   [設定]タブでメンバーが所有者を削除できないようにする
-   TiFlash<sup>ベータ</sup>およびTiKVストレージチャートメトリックを変更します
-   デフォルトのTiDBクラスタバージョンを4.0.8にアップグレードします

## 2020年10月12日 {#october-12-2020}

-   SQLWebshellクライアントをOracleMySQLクライアントから`usql`クライアントに変更します
-   デフォルトのTiDBバージョンを4.0.7にアップグレードします
-   手動バックアップの保存期間を7日から30日に延長します

## 2020年10月2日 {#october-2-2020}

-   TiFlash<sup>ベータ</sup>ディスクストレージ構成を修正

## 2020年9月14日 {#september-14-2020}

-   `region`のラベルを追加して、監視メトリックを修正します
-   非HTAPクラスターをスケーリングできない問題を修正します

## 2020年9月11日 {#september-11-2020}

-   お客様は、トラフィックフィルターを備えたパブリックエンドポイントを使用してTiDBにアクセスできるようになりました
-   自動バックアップ設定ダイアログでタイムゾーンインジケーターを追加します
-   登録が完了していないときに壊れた招待リンクを修正する

## 2020年9月4日 {#september-4-2020}

-   招待メールの誤ったURLを修正する

## 2020年8月6日 {#august-6-2020}

-   メールサポートをTiDBクラウドカスタマーサポートにアクセスするように変更します
-   カスタム電子メールログイン用のシンプルな2FA機能を追加します
-   VPCピアリングを設定する機能を追加します
-   サインアップ/ログイン用のカスタム電子メールサポートを追加します

## 2020年7月17日 {#july-17-2020}

-   自動化された毎日のバックアップのデフォルトの保持を7日間に調整します
-   異常な状態のクラスターのツールチップに理由を追加する
-   初期クレジットが0の場合でも、ユーザーがクラスタを作成できる問題を修正します
-   ダッシュボードの統合を最適化する
-   顧客のクレジットを追加するときにメールを送信する
-   テナント設定ページにテナントIDを追加します
-   ユーザーの割り当て制限に合わせて合理的な通知メッセージを最適化する
-   バックアップ/復元の指標を修正する