[サービスの公開]
LB運用自動化
root実行を許可
$ oc login https://ocp-master.nosue.mobi:8443 -u admin -p admin123
$ oc new-project scaleout
$ oc adm policy add-scc-to-user anyuid -z default -n scaleout
Dockerイメージデプロイ
https://hub.docker.com/r/stenote/nginx-hostname/
=>サービス公開
stenote/nginx-hostname

[スケールアウト]
=>コンソールから実行
COUNTER=0; while [ $COUNTER -lt 10 ]; do curl -s http://$(oc get route nginx-hostname --template='{{.spec.host}}' -n scaleout) ; let COUNTER=COUNTER+1 ; done

COUNTER=0; while [ $COUNTER -lt 10 ]; do curl -s http://nginx-hostname-scaleout.apps.nosue.mobi/ ; let COUNTER=COUNTER+1 ; done

[自律回復]
わざとPodを止めてみる
$ oc get po
$ oc delete po PODNAME
=>コンソールを見せておく

参考) Podのロケーションを確認
$ oc get pods -o wide
