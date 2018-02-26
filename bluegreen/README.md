[Blue/Green Deployment]
プロジェクト作成
oc login https://ocp-master.nobusue.net:8443
oc new-project blue-green-test --display-name="Blue/Green Deployment" --description="Blue/Green Deployment Example"

Blueバージョン(v1)デプロイ
oc new-app openshift/deployment-example:v1 --name=example-blue

Greenバージョン(v2)デプロイ
oc new-app openshift/deployment-example:v2 --name=example-green

Blueバージョン(v1)を公開
oc expose svc/example-blue --name=bluegreen-example

あとはGUIから振り分け調整が可能

テスト
COUNTER=0; while [ $COUNTER -lt 10 ]; do curl -s "http://$(oc get route bluegreen-example --template='{{.spec.host}}')" | grep '"box"'; let COUNTER=COUNTER+1 ; done

COUNTER=0; while [ $COUNTER -lt 10 ]; do curl -s "http://bluegreen-example-blue-green-test.apps.nosue.mobi/" | grep '"box"'; let COUNTER=COUNTER+1 ; done
