# PaaS的な利用(s2iビルド->自動デプロイ)
s2iビルドで、node.jsアプリケーションのサンプルをデプロイする。

サンプルアプリケーションとテンプレートの組み合わせは自由だが、組み合わせによっては待ち時間が長くなるので、事前にリハーサルしておくこと。
特にJavaアプリケーションの場合はビルドに時間がかかり、メモリ消費量も多いため、待ち時間が長くなるので注意が必要。

## 前提条件
MetricsとLoggingが有効になっているOpenShiftクラスタを用意すること。
必要なリソース(メモリ)が多く、CDK/Minishift環境ではセットアップに問題があることが多いため、マルチノードのデモ環境(自前クラスタ or RHPDSなど)を準備するのがおすすめ。

## 環境設定
```
export OCP_MASTER=https://<ocp-master-url>:<port>
```

## 事前準備
プロジェクトを作成しておく。管理コンソールから作成してもよい。

### CLI
```
oc login $OCP_MASTER
oc new-project paas --display-name="PaaS Demo"
```

### GUI
![paas_1](paas_1.png)

## node.jsアプリケーションデプロイ

### CLI
```
oc new-app https://github.com/openshift/nodejs-ex.git

--> Found image 7191d41 (6 weeks old) in image stream "openshift/nodejs" under tag "6" for "nodejs"

    Node.js 6
    ---------
    Node.js 6 available as docker container is a base platform for building and running various Node.js 6 applications and frameworks. Node.js is a platform built on Chrome's JavaScript runtime for easily building fast, scalable network applications. Node.js uses an event-driven, non-blocking I/O model that makes it lightweight and efficient, perfect for data-intensive real-time applications that run across distributed devices.

    Tags: builder, nodejs, nodejs6

    * The source repository appears to match: nodejs
    * A source build using source code from https://github.com/openshift/nodejs-ex.git will be created
      * The resulting image will be pushed to image stream "nodejs-ex:latest"
      * Use 'start-build' to trigger a new build
    * This image will be deployed in deployment config "nodejs-ex"
    * Port 8080/tcp will be load balanced by service "nodejs-ex"
      * Other containers can access this service through the hostname "nodejs-ex"

--> Creating resources ...
    imagestream "nodejs-ex" created
    buildconfig "nodejs-ex" created
    deploymentconfig "nodejs-ex" created
    service "nodejs-ex" created
--> Success
    Build scheduled, use 'oc logs -f bc/nodejs-ex' to track its progress.
    Application is not exposed. You can expose services to the outside world by executing one or more of the commands below:
     'oc expose svc/nodejs-ex'
    Run 'oc status' to view your app.
```

`oc new-app` コマンドは、指定されたGitリポジトリの内容に応じて適切なテンプレートを自動判定するため、対応しているテンプレートがあれば言語やフレームワークの指定は不要。
また、`oc` コマンドを実行する環境で一旦 `git clone` を行うため、 `git` コマンドがインストールされている必要がある。

GUIから操作した場合と異なり、サービスは自動的に公開されていないので、以下でRouteを作成しておく。

```
oc expose svc nodejs-ex
```


### GUI

![paas_2](paas_2.png)


## アプリケーション管理容易性


### アプリケーションログの確認

#### CLI

```
[master]$ oc project paas
Already on project "paas" on server "https://ocp-master.nosue.mobi:8443".
[master]$ oc get po
NAME              READY     STATUS      RESTARTS   AGE
nodeapp-1-8pnpm   1/1       Running     1          15h
nodeapp-1-build   0/1       Completed   0          15h
[master]$ oc logs -f nodeapp-1-8pnpm
Environment:
	DEV_MODE=false
	NODE_ENV=production
	DEBUG_PORT=5858
Launching via npm...
npm info it worked if it ends with ok
npm info using npm@3.10.9
npm info using node@v6.11.3
npm info lifecycle nodejs-ex@0.0.1~prestart: nodejs-ex@0.0.1
npm info lifecycle nodejs-ex@0.0.1~start: nodejs-ex@0.0.1
> nodejs-ex@0.0.1 start /opt/app-root/src
> node server.js
Server running on http://0.0.0.0:8080
10.128.0.1 - - [06/Mar/2018:02:18:42 +0000] "GET / HTTP/1.1" 200 40382 "-" "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_12_6) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/64.0.3282.186 Safari/537.36"
```

#### GUI

### コンテナへのログイン(shellのアタッチ)

#### CLI

```
[master]$ oc project paas
$ oc rsh nodeapp-1-8pnpm
sh-4.2$ ls -l
total 28
-rw-rw-r--.   1 default root 12345 Mar  5 10:59 README.md
drwxrwxr-x.   3 default root    20 Mar  5 10:59 helm
drwxrwxr-x. 100 default root  4096 Mar  5 10:59 node_modules
drwxrwxr-x.   4 default root    39 Mar  5 10:59 openshift
-rw-rw-r--.   1 default root   826 Mar  5 10:59 package.json
-rw-rw-r--.   1 default root  3154 Mar  5 10:59 server.js
drwxrwxr-x.   2 default root    25 Mar  5 10:59 tests
drwxrwxr-x.   2 default root    24 Mar  5 10:59 views
sh-4.2$ exit
exit
```

#### GUI

### リソース利用状況のモニタリング
