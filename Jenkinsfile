pipeline{
  // Inicio do Pipeline
  agent any
  environment {
    // Definições de variáveis para uso na pipeline
    NAME_APP = "dexter"
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
      //environment {
        // Definições de variáveis para uso na pipeline
        gpg_passphrase = credentials("gpg-pass")
      //}
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

    stage('SCA - Dependency Check Scan'){
      steps{
        //Execução do escaneamento de dependencias
        dependencyCheck additionalArguments: 'scan="${WORKSPACE}/" --format ALL', odcInstallation: 'dependency-check'
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