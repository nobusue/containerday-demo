[インフラCI]
アプリ変更なし、ライブラリ/ベースイメージの更新を反映
サンプルコード(php)
https://github.com/OpenShiftDemos/os-sample-php
=>s2iビルドでこちらをビルドしておき、ベースイメージをphp:latestに変更する

latestを5.6に変更
$ oc tag php:5.6 php:latest -n openshift
=>image change triggerで自動的にビルドが実行される
=>image change triggerで自動的にデプロイが実行される

latestを7.0に変更
$ oc tag php:7.0 php:latest -n openshift
=>image change triggerで自動的にビルドが実行される
=>image change triggerで自動的にデプロイが実行される

履歴確認
$ oc describe is php -n openshift
=>latestの履歴が見える
