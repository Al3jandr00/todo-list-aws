pipeline {
  agent any

  environment {
    AWS_DEFAULT_REGION = 'us-east-1'
    AWS_REGION         = 'us-east-1'
    SAM_CLI_TELEMETRY  = '0'
    STACK_NAME         = 'todo-list-staging'
    SAM_CONFIG_ENV     = 'staging'
  }

  options {
    timestamps()
    skipDefaultCheckout(true)   // para que "Get Code" sea la stage que hace el checkout
  }

  stages {

    stage('Get Code') {
      steps {
        checkout scm
        sh '''
          set -e
          echo "BRANCH_NAME=$BRANCH_NAME"
          echo "GIT_BRANCH=$GIT_BRANCH"
          git log -1 --oneline
        '''
      }
    }

    stage('Static Test') {
      steps {
        sh '''
          set -e
          python3 -m pip install --user --upgrade pip >/dev/null
          python3 -m pip install --user flake8 bandit >/dev/null

          mkdir -p reports

          # Flake8 (solo src) -> informe
          python3 -m flake8 src --statistics --count --tee --output-file reports/flake8.txt || true

          # Bandit (solo src) -> informe
          python3 -m bandit -r src -f txt -o reports/bandit.txt || true

          echo "== Flake8 report (tail) =="
          tail -n 50 reports/flake8.txt || true
          echo "== Bandit report (tail) =="
          tail -n 50 reports/bandit.txt || true
        '''
      }
      post {
        always {
          archiveArtifacts artifacts: 'reports/*.txt', fingerprint: true
        }
      }
    }

   stage('Deploy') {
  steps {
    sh '''
      set -e
      sam build
      sam validate

      # Crear config vacío para CI
      cat > samconfig-ci.toml <<EOF
version = 0.1
EOF

      sam deploy \
        --stack-name todo-list-staging \
        --resolve-s3 \
        --capabilities CAPABILITY_IAM \
        --no-confirm-changeset \
        --no-fail-on-empty-changeset \
        --config-file samconfig-ci.toml
    '''
  }
}


 stage('Rest Test') {
  steps {
    sh '''
      set -e

      # 1) Obtener la URL del API desde CloudFormation
      BASE_URL=$(aws cloudformation describe-stacks \
        --stack-name todo-list-staging \
        --region us-east-1 \
        --query "Stacks[0].Outputs[?OutputKey=='BaseUrlApi'].OutputValue" \
        --output text)

      echo "BASE_URL=$BASE_URL"
      export BASE_URL

      # 2) Instalar dependencias para tests de integración
      python3 -m pip install --user -U pytest requests

      # 3) Ejecutar pruebas de integración (si falla una, falla la stage)
      pytest -q test/integration/todoApiTest.py -m api
    '''
  }
}


    stage('Promote') {
  when {
    branch 'develop'
  }
  steps {
    withCredentials([usernamePassword(
      credentialsId: 'github-pat',
      usernameVariable: 'GIT_USER',
      passwordVariable: 'GIT_TOKEN'
    )]) {
      sh '''
        set -e
        git config user.email "jenkins@ci.local"
        git config user.name "Jenkins CI"

        git checkout master
        git pull https://$GIT_USER:$GIT_TOKEN@github.com/Al3jandr00/todo-list-aws.git master

        git merge develop

        git push https://$GIT_USER:$GIT_TOKEN@github.com/Al3jandr00/todo-list-aws.git master
      '''
    }
  }
}

  }
}



