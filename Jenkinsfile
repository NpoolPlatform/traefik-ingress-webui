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
      when {
        expression { BUILD_TARGET == 'true' }
      }
      
      steps {
        nodejs('nodejs') {
          sh 'cd .traefik/webui; npm install'
          sh 'cd .traefik/webui; NODE_ENV=production APP_ENV=production PLATFORM_URL=http://traefik-webui.internal-devops.$TARGET_ENV.npool.top/traefik/dashboard APP_API=http://traefik-api.internal-devops.$TARGET_ENV.npool.top/traefik/api APP_PUBLIC_PATH=traefik/dashboard npm run build-quasar'
          sh 'mkdir -p .webui/static; cp .traefik/webui/dist/spa/* .webui/static -rf'
          sh 'cp Dockerfile.webui .webui/Dockerfile'
          sh 'cp nginx.conf.template .webui/nginx.conf.template'
          sh(returnStdout: true, script: '''
            images=`docker images | grep entropypool | grep traefik-webui | grep $TARGET_ENV | awk '{ print $3 }'`
            for image in $images; do
              docker rmi $image
            done
          '''.stripIndent())
          sh 'cd .webui; docker build -t entropypool/traefik-webui-$TARGET_ENV:v2.5.3 .'
        }
      }
    }

    stage('Push docker image') {
      when {
        expression { RELEASE_TARGET == 'true' }
      }
      steps {
        sh(returnStdout: true, script: '''
          while true; do
            docker push entropypool/traefik-service:v2.5.3
            if [ $? -eq 0 ]; then
              break
            fi
          done
        
          while true; do
            docker push entropypool/traefik-webui-$TARGET_ENV:v2.5.3
            if [ $? -eq 0 ]; then
              break
            fi
          done
        '''.stripIndent())
      }
    }

    stage('Deploy traefik') {
      when {
        expression { DEPLOY_TARGET == 'true' }
      }
      steps {
        sh 'sed -i "s/internal-devops.development.npool.top/internal-devops.$TARGET_ENV.npool.top/g" k8s/04-traefik-dashboard-ingress.yaml'
        sh 'sed -i "s/traefik-webui:v2.5.3/traefik-webui-$TARGET_ENV:v2.5.3/g" k8s/03-deployments.yaml'
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
        sh(returnStdout: true, script: '''
          remoteurl=`git remote -v | grep fetch | awk '{ print $2 }'`
          projectname=`basename $remoteurl`
          bash $JENKINS_HOME/wechat-templates/send_wxmsg.sh failure $TARGET_ENV $projectname
        '''.stripIndent())
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
