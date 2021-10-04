pipeline {
  agent any
  environment {
    GOPROXY = 'https://goproxy.cn,direct'
  }
  tools {
    go 'go'
  }
  stages {
    stage('Clone traefik ingress') {
      steps {
        git(url: scm.userRemoteConfigs[0].url, branch: '$BRANCH_NAME', changelog: true, credentialsId: 'KK-github-key', poll: true)
      }
    }

    stage('Build traefik') {
      steps {
        sh 'rm .traefik -rf'
        sh 'git clone https://github.com/traefik/traefik.git .traefik; cd .traefik; git checkout v2.5.3'
        sh 'cp Makefile.service .traefik/Makefile'
        sh 'cp build.Dockerfile.service .traefik/build.Dockerfile'
        sh 'cd .traefik; make traefik-binary'
        sh 'mkdir -p .traefik-release'
        sh 'cp .traefik/dist/traefik .traefik-release'
        sh 'cp entrypoint.sh .traefik-release'
        sh 'cp .traefik/script/ca-certificates.crt .traefik-release'
        sh 'cp Dockerfile.service .traefik-release/Dockerfile'
        sh(returnStdout: true, script: '''#!/bin/sh
          sh 'docker images | grep entropypool | grep traefik-service'
          if [ 0 -eq $? ]; then
            sh 'docker rmi entropypool/traefik-service:v2.5.3'
          fi
        '''.stripIndent())
        sh 'cd .traefik-release; docker build -t entropypool/traefik-service:v2.5.3 .'

        nodejs('nodejs') {
          sh 'cd .traefik/webui; npm install'
          sh 'cd .traefik/webui; NODE_ENV=production APP_ENV=production PLATFORM_URL=http://traefik-webui.internal-devops.$TARGET_ENV.npool.top/traefik/dashboard APP_API=http://traefik-api.internal-devops.$TARGET_ENV.npool.top/traefik/api APP_PUBLIC_PATH=traefik/dashboard npm run build-quasar'
          sh 'mkdir -p .webui/static; cp .traefik/webui/dist/spa/* .webui/static -rf'
          sh 'cp Dockerfile.webui .webui/Dockerfile'
          sh 'cp nginx.conf.template .webui/nginx.conf.template'
          sh(returnStdout: true, script: '''#!/bin/sh
            sh 'docker images | grep entropypool | traefik-webui'
            if [ 0 -eq $? ]; then
              sh 'docker rmi entropypool/traefik-webui:v2.5.3'
            fi
          '''.stripIndent())
          sh 'cd .webui; docker build -t entropypool/traefik-webui:v2.5.3 .'
        }
      }
    }

    stage('Push docker image and deploy') {
      when {
        expression { DEPLOY_TARGET == 'true' }
      }
      steps {
        sh 'sed -i "s/internal-devops.development.npool.top/internal-devops.$TARGET_ENV.npool.top/g" k8s/04-traefik-dashboard-ingress.yaml'
        sh 'docker push entropypool/traefik-service:v2.5.3'
        sh 'docker push entropypool/traefik-webui:v2.5.3'
        sh 'cd /etc/kubeasz; ./ezctl checkout $TARGET_ENV'
        sh 'kubectl apply -k k8s/'
      }
    }
  }
  post('Report') {
    fixed {
      script {
        sh(script: 'bash $JENKINS_HOME/wechat-templates/send_wxmsg.sh fixed')
     }
      script {
        // env.ForEmailPlugin = env.WORKSPACE
        emailext attachmentsPattern: 'TestResults\\*.trx',
        body: '${FILE,path="$JENKINS_HOME/email-templates/success_email_tmp.html"}',
        mimeType: 'text/html',
        subject: currentBuild.currentResult + " : " + env.JOB_NAME,
        to: '$DEFAULT_RECIPIENTS'
      }
     }
    success {
      script {
        sh(script: 'bash $JENKINS_HOME/wechat-templates/send_wxmsg.sh successful')
     }
      script {
        // env.ForEmailPlugin = env.WORKSPACE
        emailext attachmentsPattern: 'TestResults\\*.trx',
        body: '${FILE,path="$JENKINS_HOME/email-templates/success_email_tmp.html"}',
        mimeType: 'text/html',
        subject: currentBuild.currentResult + " : " + env.JOB_NAME,
        to: '$DEFAULT_RECIPIENTS'
      }
     }
    failure {
      script {
        sh(script: 'bash $JENKINS_HOME/wechat-templates/send_wxmsg.sh failure')
     }
      script {
        // env.ForEmailPlugin = env.WORKSPACE
        emailext attachmentsPattern: 'TestResults\\*.trx',
        body: '${FILE,path="$JENKINS_HOME/email-templates/fail_email_tmp.html"}',
        mimeType: 'text/html',
        subject: currentBuild.currentResult + " : " + env.JOB_NAME,
        to: '$DEFAULT_RECIPIENTS'
      }
     }
    aborted {
      script {
        sh(script: 'bash $JENKINS_HOME/wechat-templates/send_wxmsg.sh aborted')
     }
      script {
        // env.ForEmailPlugin = env.WORKSPACE
        emailext attachmentsPattern: 'TestResults\\*.trx',
        body: '${FILE,path="$JENKINS_HOME/email-templates/fail_email_tmp.html"}',
        mimeType: 'text/html',
        subject: currentBuild.currentResult + " : " + env.JOB_NAME,
        to: '$DEFAULT_RECIPIENTS'
      }
     }
  }
}
