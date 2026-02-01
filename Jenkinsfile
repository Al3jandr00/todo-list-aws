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

          # Obtener BaseUrlApi desde CloudFormation Outputs
          BASE_URL=$(aws cloudformation describe-stacks \
            --stack-name ${STACK_NAME} \
            --region ${AWS_REGION} \
            --query "Stacks[0].Outputs[?OutputKey=='BaseUrlApi'].OutputValue" \
            --output text)

          echo "BaseUrlApi = $BASE_URL"

          # Test sencillo tipo smoke (fallará si API no responde 200)
          curl -sS -o /dev/null -w "%{http_code}" "$BASE_URL/todos" | grep -q "^200$"
        '''
      }
    }

    stage('Promote') {
      steps {
        input message: "¿Promover a master (merge develop -> master)?", ok: "Promote"

        sh '''
          set -e
          echo "Promote stage: aquí harías el merge a master."
          echo "Si quieres automatizarlo de verdad, añadimos credenciales GitHub en Jenkins."
        '''
      }
    }
  }
}



