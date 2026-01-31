pipeline {
  agent any

  environment {
    AWS_DEFAULT_REGION = 'us-east-1'
    SAM_CLI_TELEMETRY  = '0'
  }

  options {
    timestamps()
  }

  stages {
    stage('Checkout') {
      steps { checkout scm }
    }

    stage('Info') {
      steps {
        sh '''
          set -e
          whoami
          uname -a
          aws --version
          sam --version || true
          python3 --version || true
        '''
      }
    }

    stage('Build') {
      steps {
        sh '''
          set -e
          sam build
        '''
      }
    }

    stage('Unit tests') {
      steps {
        sh '''
          set -e
          pytest -q || true
        '''
      }
    }

    stage('Deploy STAGING') {
      when { branch 'develop' }
      steps {
        sh '''
          set -e
          sam deploy \
            --stack-name todo-list-staging \
            --no-confirm-changeset \
            --no-fail-on-empty-changeset \
            --capabilities CAPABILITY_IAM
        '''
      }
    }

    stage('Deploy PRODUCTION') {
      when { branch 'master' }
      steps {
        sh '''
          set -e
          sam deploy \
            --stack-name todo-list-production \
            --no-confirm-changeset \
            --no-fail-on-empty-changeset \
            --capabilities CAPABILITY_IAM
        '''
      }
    }
  }
}

