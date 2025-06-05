pipeline{
  // Inicio do Pipeline
  agent any
  environment {
    // Definições de variáveis para uso na pipeline
    NAME_APP = "dexter"
    ZAP_API_KEY = credentials('api-key-owasp')
  }

  // Estágio de clone do repositório
  stages {
    stage('Repositório de Código'){
      steps{
        git credentialsId: 'gitlab',
            url: 'git@github.com:davidhsv/527-php-app.git',
            branch: 'main'
        stash 'dexter-repositorio'
      }
    }

    stage('Configurando Git Secret'){
      agent { node 'automation' }
      environment {
        // Definições de variáveis para uso na pipeline
        gpg_passphrase = credentials("gpg-pass")
      }
      steps{
        script{
        //             git secret reveal -p '$gpg_passphrase'
          sh """
            cd $WORKSPACE
            git secret reveal -p '$gpg_passphrase'
          """
        }
      }
    }

    stage('SCA - Dependency Check Scan') {
        steps {
            withCredentials([string(credentialsId: 'nvd-api-key',
                                    variable: 'NVD_API_KEY')]) {
                // garantir que o workspace tem arquivo
                // da step anterior
                script {
                    sh "ls -l ${WORKSPACE}"
                }
                dependencyCheck(
                    odcInstallation: 'dependency-check',
                    additionalArguments:
                        "--scan \"${WORKSPACE}\" " +          // o que escanear
                        "--format ALL " +                    // XML + HTML
                        "--nvdApiKey ${NVD_API_KEY}"         // <<< aqui
                )
            }
        }
    }

    stage('SCA - Dependency Check Publish Report'){
      steps{
        //Publicação do relatorio de vulnerabilidades de dependencias no Jenkins
        dependencyCheckPublisher pattern: "dependency-check-report.xml"
      }
    }

    stage('SAST - Escaneamento com Sonarqube'){
      environment {
        // Referencia do Scanner do Sonarqube
        scanner = tool 'sonar-scanner'
      }
      steps{
        // Referencia ao Plugin do Sonarqube
        withSonarQubeEnv('sonarqube') {
          // Execução do scanner com os parametros do Sonarqube
          sh "${scanner}/bin/sonar-scanner -Dsonar.projectKey=$NAME_APP -Dsonar.sources=${WORKSPACE}/ -Dsonar.projectVersion=${BUILD_NUMBER} -Dsonar.dependencyCheck.xmlReportPath=${WORKSPACE}/dependency-check-report.xml -Dsonar.dependencyCheck.htmlReportPath=${WORKSPACE}/dependency-check-report.html"
        }
        // Validação da Qualidade de Código
        waitForQualityGate abortPipeline: true
      }
    }

    stage('Build Image'){

      // Configuração do Agent Automation
      agent { node 'automation' }

      steps {
        // Criando a imagem em container
        script {
          docker.build("$NAME_APP:${BUILD_NUMBER}", "-f dockerfiles/php7.1.dockerfile .")
        }
      }
    }

    stage('Artifact Repository - Trivy'){

      agent { node 'automation' }
      steps {
        unstash 'dexter-repositorio'
        script {
          sh 'trivy --severity CRITICAL image $NAME_APP:${BUILD_NUMBER}'
        }
      }
    }

    stage('Executando Container'){

      // Configuração do Agent Automation
      agent { node 'automation' }

      steps {
        // Executando stack
        script {
          sh "docker-compose up -d"
        }
      }
    }

    stage('DAST-OWASP'){
      environment {
        // variaveis especifcas para o OWASP ZAP
        ZAP_PATH = "/usr/share/owasp-zap/"
        ZAP_LOG_PATH = "$WORKSPACE"
        APP_URL= "http://192.168.56.20"
      }
      steps {
        // execução do OWASP ZAP utilizando ap-cli-v2
        //sh 'zap-cli-v2 start -o "-config api.key=${ZAP_API_KEY} -config ajaxSpider.browserId=htmlunit -config connection.timeoutInSecs=2200 -dir ${WORKSPACE} -addoninstallall"'
        sh """
            java -Xmx746m -jar /usr/share/zap/zap-2.16.1.jar \
            -daemon -port 8090 \
            -config api.key=${ZAP_API_KEY} \
            -config ajaxSpider.browserId=htmlunit \
            -config connection.timeoutInSecs=2200 \
            -dir ${WORKSPACE} -addoninstallall &
        """
            sleep(30)
        // Espera até 5 minutos para o ZAP inicializar
        sh "zap-cli-v2 status -t 3000"
        sh "zap-cli-v2 -v spider $APP_URL"
        sh "zap-cli-v2 -v ajax-spider $APP_URL"
        sh "zap-cli-v2 -v active-scan -r $APP_URL"
        sh "zap-cli-v2 report -o reports/vuln-report-${BUILD_NUMBER}.html -f html"
        sh "zap-cli-v2 report -o reports/vuln-report-${BUILD_NUMBER}.xml -f xml"
        sh "zap-cli-v2 shutdown"

        // Publicacao dos relatorio em formato HTML
        publishHTML([
          allowMissing: false,
          alwaysLinkToLastBuild: true,
          keepAll: true,
          reportDir: 'reports/',
          reportFiles: 'vuln-report-${BUILD_NUMBER}.html',
          reportName: 'OWASP ZAP - VULNERABILITY REPORT'
        ])
      }
    }
  }


  // Execuções de finalização da Pipeline
  post {
    always { chuckNorris() }
    success {
        echo "Pipeline executada com sucesso!"
    }
    failure {
        echo "Pipeline falhou!"
    }
  }
}