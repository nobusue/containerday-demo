[CI/CDパイプライン]
https://github.com/StefanoPicozzi/cotd
https://stefanopicozzi.blog/2016/10/18/pipelines-with-openshift/
https://stefanopicozzi.blog/2016/10/07/ab-deployments-made-easy-with-openshift/

forkしてiuncludes/selectors.phpを修正する
https://github.com/nobusue/cotd
=>cats / cities / pets のいずれか (cotd/data/*.phpのファイル名)
修正後、pipeline buildを実行

Jenkins Pipeline (cotd)

サンプル
https://github.com/StefanoPicozzi/cotd

手順
- Project作成
oc login -u developer
GUID=mydemo
oc new-project pipeline-${GUID}-dev --description="Cat of the Day Development Environment" --display-name="Cat Of The Day - Dev"
oc new-project pipeline-${GUID}-test --description="Cat of the Day Testing Environment" --display-name="Cat Of The Day - Test"
oc new-project pipeline-${GUID}-prod --description="Cat of the Day Production Environment" --display-name="Cat Of The Day - Prod"

- Jenkins作成
oc new-app jenkins-persistent -n pipeline-${GUID}-dev

- 権限設定
oc policy add-role-to-user edit system:serviceaccount:pipeline-${GUID}-dev:jenkins -n pipeline-${GUID}-test
oc policy add-role-to-user edit system:serviceaccount:pipeline-${GUID}-dev:jenkins -n pipeline-${GUID}-prod

oc policy add-role-to-group system:image-puller system:serviceaccounts:pipeline-${GUID}-test -n pipeline-${GUID}-dev
oc policy add-role-to-group system:image-puller system:serviceaccounts:pipeline-${GUID}-prod -n pipeline-${GUID}-dev

- アプリケーションデプロイ
oc new-app https://github.com/StefanoPicozzi/cotd.git -n pipeline-${GUID}-dev
sleep 5 # give OpenShift a chance to start the build
oc logs -f build/cotd-1 -n pipeline-${GUID}-dev

- image streamタグ付け
oc tag cotd:latest cotd:testready -n pipeline-${GUID}-dev
oc tag cotd:testready cotd:prodready -n pipeline-${GUID}-dev
oc describe is cotd -n pipeline-${GUID}-dev

- testとprodへアプリケーションをリリース
oc new-app pipeline-${GUID}-dev/cotd:testready --name=cotd -n pipeline-${GUID}-test
oc new-app pipeline-${GUID}-dev/cotd:prodready --name=cotd -n pipeline-${GUID}-prod

- サービス公開
oc expose service cotd -n pipeline-${GUID}-dev
oc expose service cotd -n pipeline-${GUID}-test
oc expose service cotd -n pipeline-${GUID}-prod

- automatic deploymentを無効化
oc get dc cotd -o yaml -n pipeline-${GUID}-dev | sed 's/automatic: true/automatic: false/g' | oc replace -f -
oc get dc cotd -o yaml -n pipeline-${GUID}-test| sed 's/automatic: true/automatic: false/g' | oc replace -f -
oc get dc cotd -o yaml -n pipeline-${GUID}-prod | sed 's/automatic: true/automatic: false/g' | oc replace -f -

- Build Config Pipeline作成
devプロジェクトで Add to Project -> Import YAML/JSON
apiVersion: v1
items:
- kind: "BuildConfig"
  apiVersion: "v1"
  metadata:
    name: "pipeline-demo"
  spec:
    triggers:
          - github:
              secret: 5Mlic4Le
            type: GitHub
          - generic:
              secret: FiArdDBH
            type: Generic
    strategy:
      type: "JenkinsPipeline"
      jenkinsPipelineStrategy:
        jenkinsfile: |
                          node {
                              stage ("Build")
                                    echo '*** Build Starting ***'
                                    openshiftBuild bldCfg: 'cotd', buildName: '', checkForTriggeredDeployments: 'false', commitID: '', namespace: '', showBuildLogs: 'true', verbose: 'true'
                                    openshiftVerifyBuild bldCfg: 'cotd', checkForTriggeredDeployments: 'false', namespace: '', verbose: 'false'
                                    echo '*** Build Complete ***'
                              stage ("Deploy and Verify in Development Env")
                                    echo '*** Deployment Starting ***'
                                    openshiftDeploy depCfg: 'cotd', namespace: '', verbose: 'false', waitTime: ''
                                    openshiftVerifyDeployment authToken: '', depCfg: 'cotd', namespace: '', replicaCount: '1', verbose: 'false', verifyReplicaCount: 'false', waitTime: ''
                                    echo '*** Deployment Complete ***'
                               }

kind: List
metadata: {}

- Pipelineのテスト
Builds->Pipeline->Start Pipeline
完了まで待つ
Jenkinsとの統合
Pipeline定義の解説

- CD Pipeline
build configをedit
  node {
    withEnv(['GUID=mydemo']) {

    stage ("Build")
    echo '*** Build Starting ***'
    openshiftBuild bldCfg: 'cotd', buildName: '', checkForTriggeredDeployments: 'false', commitID: '', namespace: '', showBuildLogs: 'false', verbose: 'false', waitTime: ''
    openshiftVerifyBuild apiURL: 'https://openshift.default.svc.cluster.local', authToken: '', bldCfg: 'cotd', checkForTriggeredDeployments: 'false', namespace: '', verbose: 'false'
    echo '*** Build Complete ***'

    stage ("Deploy and Verify in Development Env")

    echo '*** Deployment Starting ***'
    openshiftDeploy apiURL: 'https://openshift.default.svc.cluster.local', authToken: '', depCfg: 'cotd', namespace: '', verbose: 'false', waitTime: ''
    openshiftVerifyDeployment apiURL: 'https://openshift.default.svc.cluster.local', authToken: '', depCfg: 'cotd', namespace: '', replicaCount: '1', verbose: 'false', verifyReplicaCount: 'false', waitTime: ''
    echo '*** Deployment Complete ***'

    echo '*** Service Verification Starting ***'
    openshiftVerifyService apiURL: 'https://openshift.default.svc.cluster.local', authToken: '', namespace: 'pipeline-${GUID}-dev', svcName: 'cotd', verbose: 'false'
    echo '*** Service Verification Complete ***'
    openshiftTag(srcStream: 'cotd', srcTag: 'latest', destStream: 'cotd', destTag: 'testready')

    stage ('Deploy and Test in Testing Env')
    echo '*** Deploy testready build in pipeline-${GUID}-test project  ***'
    openshiftDeploy apiURL: 'https://openshift.default.svc.cluster.local', authToken: '', depCfg: 'cotd', namespace: 'pipeline-${GUID}-test', verbose: 'false', waitTime: ''

    openshiftVerifyDeployment apiURL: 'https://openshift.default.svc.cluster.local', authToken: '', depCfg: 'cotd', namespace: 'pipeline-${GUID}-test', replicaCount: '1', verbose: 'false', verifyReplicaCount: 'false', waitTime: '10'
    sh 'curl http://cotd-pipeline-${GUID}-test.apps.nobusue.net/data/ | grep cats -q'

    stage ('Promote and Verify in Production Env')
    echo '*** Waiting for Input ***'
    input 'Should we deploy to Production?'
    openshiftTag(srcStream: 'cotd', srcTag: 'testready', destStream: 'cotd', destTag: 'prodready')
    echo '*** Deploying to Production ***'
    openshiftDeploy apiURL: 'https://openshift.default.svc.cluster.local', authToken: '', depCfg: 'cotd', namespace: 'pipeline-${GUID}-prod', verbose: 'false', waitTime: ''
    openshiftVerifyDeployment apiURL: 'https://openshift.default.svc.cluster.local', authToken: '', depCfg: 'cotd', namespace: 'pipeline-${GUID}-prod', replicaCount: '1', verbose: 'false', verifyReplicaCount: 'false', waitTime: '10'
    sleep 10
    sh 'curl http://cotd-pipeline-${GUID}-prod.apps.nobusue.net/data/ | grep cats -q'
                            }

  }
  
