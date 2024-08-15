@Library('pipeline-library@v3.0') _

def deploy (def consulToken) {
    getConsul("${consulToken}")
    updateSecret()
    //updateCronImage()
    applyKubernetesManifest()
    //updateDeploymentImage()
}

def updateSecret() {
    try {
        sh "gcloud container clusters get-credentials ${gkeName} --zone ${zoneGke} --project ${projectName}"
        sh "kubectl --context ${gkeContext} -n ${namespace} delete secret ${serviceName}-cold-app-secret || true"
        sh "kubectl --context ${gkeContext} -n ${namespace} delete secret ${serviceName}-hot-app-secret || true"
        sh "sleep 5"
        sh "kubectl --context ${gkeContext} -n ${namespace} create secret generic ${serviceName}-cold-app-secret --from-env-file=${serviceName}-env-cold"
        sh "kubectl --context ${gkeContext} -n ${namespace} create secret generic ${serviceName}-hot-app-secret --from-env-file=${serviceName}-env-hot"
        currentBuild.result = 'SUCCESS'
    } catch(e) {
        notification(
            slackStatus: "FAILURE",
            messageColor: "danger",
            headMessage: "Update Secret to ${resourceEnv} environment"
        )
        currentBuild.result = 'FAILURE'
        throw e
    } finally {
        if (currentBuild.result == "FAILURE") {
            echo "Deployment to ${resourceEnv} environment fail"
        }
    }
}

def getConsul(def consulToken) {
    try {
        sh "getConsul.py ${consul}/backend/cold ${consulToken} > ${serviceName}-env-cold"
        sh "getConsul.py ${consul}/backend/hot ${consulToken} > ${serviceName}-env-hot"
        currentBuild.result = 'SUCCESS'
    } catch(e) {
        notification(
            slackStatus: "FAILURE",
            messageColor: "danger",
            headMessage: "Get Consul ${serviceName} Fail"
        )
        currentBuild.result = 'FAILURE'
        throw e
    } finally {
        if (currentBuild.result == "FAILURE") {
          echo "Get Consul ${serviceName} Fail"
        }
    }
}

def build() {
    try {
        env.imageTag = "${versioningCode}-${shortCommitHash}-${BUILD_NUMBER}"
        withCredentials([
            file(credentialsId: "jenkinsServiceAccount", variable: 'keyJenkinsServiceAccount')
        ]) {
            sh "gcloud auth activate-service-account ${emailJenkinsServiceAccount} --key-file=${keyJenkinsServiceAccount}"
            sh "gcloud auth configure-docker ${garLocation}"
            sh "docker build -t ${crUri}:${imageTag} -t ${crUri}:${resourceEnv}-latest ."
            sh "docker push ${crUri}:${imageTag}"
            sh "docker push ${crUri}:${resourceEnv}-latest"
        }
        currentBuild.result = 'SUCCESS'
    } catch(e) {
        notification(
            slackStatus: "FAILURE",
            messageColor: "danger",
            headMessage: "Build Image ${serviceName}"
        )
        currentBuild.result = 'FAILURE'
        throw e
    } finally {
        if (currentBuild.result == "FAILURE") {
            echo "Build Image ${serviceName} fail"
        }
    }
}

def updateCronImage() {
    try {
        env.cronCheck = sh (
            script: "kubectl --context ${gkeContext} get cronjobs -n ${namespace} | grep -v NAME | awk '{print \$1}'",
            returnStdout: true
        ).trim()
        if (env.cronCheck == "") {
            echo "Cron not found"
        } else {
            for (String cron : cronCheck.split("\\s+")) {
                sh "kubectl --context ${gkeContext} set image cronjob/${cron} ${cron}=${crUri}/${namespace}:${imageTag} -n ${namespace}"
            }
        }
    } catch(e) {
        notification(
            slackStatus: "FAILURE",
            messageColor: "danger",
            headMessage: "Update image cronjobs to ${resourceEnv} environment"
        )
        currentBuild.result = 'FAILURE'
        throw e
    } finally {
        if (currentBuild.result == "FAILURE") {
            echo "Deployment to ${resourceEnv} environment fail"
        }
    }
}

def updateDeploymentImage() {
    try {
        env.deploymentCheck = sh (
            script: "kubectl --context ${gkeContext} get deployment -n ${namespace} | grep -Ev 'primary|NAME' | awk '{print \$1}'",
            returnStdout: true
        ).trim()
        if (env.deploymentCheck == "") {
            echo "deployment not found"
        } else {
            for (String deploymentName : deploymentCheck.split("\\s+")) {
                sh "kubectl --context ${gkeContext} -n ${namespace} set image deployment/${deploymentName} ${serviceName}-app=${crUri}:${imageTag}"
                sh "kubectl --context ${gkeContext} -n ${namespace} restart deployment/${deploymentName}"
            }
            notification(
                slackStatus: "SUCCESS",
                messageColor: "good",
                headMessage: "Deployment to ${resourceEnv} environment"
            )
        }
    } catch(e) {
        notification(
            slackStatus: "FAILURE",
            messageColor: "danger",
            headMessage: "Deployment to ${resourceEnv} environment"
        )
        currentBuild.result = 'FAILURE'
        throw e
    } finally {
        if (currentBuild.result == "FAILURE") {
            echo "Deployment to ${resourceEnv} environment fail"
        }
    }
}

def applyKubernetesManifest() {
    try {
        sh "sed -i 's|image:.*|image: ${crUri}:${imageTag}|' ${WORKSPACE}/kubernetes/${resourceEnv}/backend/*.yaml"
        sh "cd ${WORKSPACE}/kubernetes/${resourceEnv}/backend; for each in *; do cat \$each; echo '---'; done"
        sh "kubectl --context ${gkeContext} apply -f ${WORKSPACE}/kubernetes/${resourceEnv}/backend -n ${namespace}"
        notification(
            slackStatus: "SUCCESS",
            messageColor: "good",
            headMessage: "Deployment to ${resourceEnv} environment"
        )
    } catch(e) {
        notification(
            slackStatus: "FAILURE",
            messageColor: "danger",
            headMessage: "Deployment to ${resourceEnv} environment"
        )
        currentBuild.result = 'FAILURE'
        throw e
    } finally {
        if (currentBuild.result == "FAILURE") {
            echo "Deployment to ${resourceEnv} environment fail"
        }
    }
}

pipeline {
    agent {
        node {
            label 'docker-slave-ssh || docker-slave-ssh-01'
        }
    }
    stages {
        stage('Get Env') {
            steps {
                script {
                    getEnv()
                }
            }
        }
        stage('Environment Check') {
            steps {
                script {
                    envFilter()
                }
            }
        }
        stage('Cicd Notification'){
            when {
                expression {
                    currentBuild.result == 'SUCCESS'
                }
            }
            steps {
                script {
                    notification(
                        slackStatus: "STARTED",
                        messageColor: "#439FE0",
                        headMessage: "Deployment Status"
                    )
                }
            }
        }
        stage ('PR Check') {
            parallel {
                stage ("Run Sonarqube") {
                    when {
                        expression {
                            currentBuild.result == "SUCCESS" && env.resourceEnv == "pull_request"
                        }
                    }
                    steps {
                        script {
                            unitTest()
                        }
                    }
                }
            }
        }
        stage ("Build Image") {
            when {
                expression {
                    currentBuild.result == 'SUCCESS' && env.resourceEnv != "pull_request"
                }
            }
            steps {
                script {
                    build()
                }
            }
        }
        stage ("Deployment to Development Environment") {
            when {
                expression {
                    currentBuild.result == "SUCCESS" && env.resourceEnv == "development"
                }
            }
            steps {
                script {
                    withCredentials([
                        string(credentialsId: "consul-dev-token", variable: 'consulDevToken')
                    ]) {
                        deploy ("${consulDevToken}")
                    }
                }
            }
        }
        stage ("Deployment to Staging Environment") {
            when {
                expression {
                    currentBuild.result == "SUCCESS" && env.resourceEnv == "staging"
                }
            }
            steps {
                script {
                    withCredentials([
                        string(credentialsId: "consul-stg-token", variable:'consulStgToken')
                    ]) {
                        deploy ("${consulStgToken}")
                    }
                }
            }
        }
        stage ("Approval") {
            when {
                expression {
                    currentBuild.result == "SUCCESS" && env.resourceEnv == "production"
                }
            }
            steps {
                script {
                    approval()
                }
            }
        }
        stage ("Deployment to Production Environment") {
            when {
                expression {
                    env.doDeploy == "do"
                }
            }
            steps {
                script {
                    withCredentials([
                        string(credentialsId: "consul-prod-token", variable:'consulProdToken')
                    ]) {
                        deploy ("${consulProdToken}")
                    }
                }
            }
        }
    }
    post {
        always {
            cleanWs()
       }
    }
}
