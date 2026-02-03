pipeline {
  agent any

  environment {
    AWS_DEFAULT_REGION = 'us-east-1'
    AWS_REGION         = 'us-east-1'
    SAM_CLI_TELEMETRY  = '0'
  }

  options {
    timestamps()
    skipDefaultCheckout(true)
  }

  stages {

    stage('Get Code') {
      steps {
        checkout scm
        sh '''
          set -e
          echo "BRANCH_NAME=$BRANCH_NAME"
          git log -1 --oneline
        '''
      }
    }

    // =========================
    // CI (develop) - Reto 1
    // =========================
    stage('Static Test') {
      when { branch 'develop' }
      steps {
        sh '''
          set -e
          python3 -m pip install --user --upgrade pip >/dev/null
          python3 -m pip install --user flake8 bandit >/dev/null

          mkdir -p reports

          # Flake8 solo /src -> informe (NO falla el stage si hay issues)
          python3 -m flake8 src --statistics --count --tee --output-file reports/flake8.txt || true
          test -s reports/flake8.txt || echo "No flake8 issues" > reports/flake8.txt

          # Bandit solo /src -> informe (NO falla el stage si hay issues)
          python3 -m bandit -r src -f txt -o reports/bandit.txt || true
          test -s reports/bandit.txt || echo "No bandit issues" > reports/bandit.txt

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
      set -eu

      # Elegir entorno por rama
      if [ "$BRANCH_NAME" = "develop" ]; then
        STACK_NAME="todo-list-staging"
        STAGE_PARAM="staging"
      elif [ "$BRANCH_NAME" = "master" ]; then
        STACK_NAME="todo-list-production"
        STAGE_PARAM="production"
      else
        echo "INFO: Rama $BRANCH_NAME sin despliegue (solo develop/master)."
        exit 0
      fi

      echo "=== Deploy stack: $STACK_NAME (Stage=$STAGE_PARAM) ==="

      sam build
      sam validate

      # Config mínimo para CI (evitar modo guiado)
      cat > samconfig-ci.toml <<EOF
version = 0.1
EOF

      # --- FIX: evitar AlreadyExistsException por changeset samcli-deploy repetido ---
      echo "Cleaning old CloudFormation changesets for $STACK_NAME..."
      CHANGESETS=$(aws cloudformation list-change-sets \
        --stack-name "$STACK_NAME" \
        --region us-east-1 \
        --query "Summaries[?starts_with(ChangeSetName, 'samcli-deploy')].[ChangeSetName,Status]" \
        --output text 2>/dev/null || true)

      if [ -n "$CHANGESETS" ]; then
        echo "$CHANGESETS" | while read -r CS_NAME CS_STATUS; do
          case "$CS_STATUS" in
            CREATE_IN_PROGRESS|DELETE_IN_PROGRESS)
              echo " - Skip (in progress): $CS_NAME ($CS_STATUS)"
              ;;
            *)
              echo " - Deleting: $CS_NAME ($CS_STATUS)"
              aws cloudformation delete-change-set \
                --stack-name "$STACK_NAME" \
                --change-set-name "$CS_NAME" \
                --region us-east-1 || true
              ;;
          esac
        done
      else
        echo " - No samcli-deploy changesets found."
      fi

      # pequeño delay por si hay builds muy seguidos
      sleep 2
      # ---------------------------------------------------------------------------

      sam deploy \
        --stack-name "$STACK_NAME" \
        --resolve-s3 \
        --capabilities CAPABILITY_IAM \
        --no-confirm-changeset \
        --no-fail-on-empty-changeset \
        --config-file samconfig-ci.toml \
        --parameter-overrides Stage="$STAGE_PARAM"
    '''
  }
}




    // =========================
    // CI Rest Test (develop) - Pytest completo (5 pruebas)
    // =========================
    stage('Rest Test') {
      when { branch 'develop' }
      steps {
        sh '''
          set -e

          STACK_NAME="todo-list-staging"

          # 1) Obtener la URL del API desde CloudFormation
          BASE_URL=$(aws cloudformation describe-stacks \
            --stack-name "$STACK_NAME" \
            --region us-east-1 \
            --query "Stacks[0].Outputs[?OutputKey=='BaseUrlApi'].OutputValue" \
            --output text)

          echo "BASE_URL=$BASE_URL"
          export BASE_URL

          # 2) Instalar deps de tests
          python3 -m pip install --user -U pytest requests >/dev/null

          # 3) Ejecutar pruebas de integración (si falla una, falla el pipeline)
          pytest -q test/integration/todoApiTest.py -m api
        '''
      }
    }

    // =========================
    // CD Rest Test (master) - SOLO LECTURA con curl (opción 3)
    // =========================
 stage('Rest Test (Read Only)') {
  when { expression { return env.BRANCH_NAME == 'master' } }
  steps {
    sh '''
      set -eu

      STACK_NAME="todo-list-production"
      echo "=== Rest Test (READ-ONLY) ==="

      BASE_URL=$(aws cloudformation describe-stacks \
        --stack-name "$STACK_NAME" \
        --region us-east-1 \
        --query "Stacks[0].Outputs[?OutputKey=='BaseUrlApi'].OutputValue" \
        --output text)

      if [ -z "$BASE_URL" ] || [ "$BASE_URL" = "None" ]; then
        echo "ERROR: BASE_URL no obtenido desde CloudFormation"
        exit 1
      fi

      echo "BASE_URL=$BASE_URL"

      echo "---- curl: GET /todos ----"
      LIST_RESP=$(curl -sS -w "\\nHTTP_STATUS:%{http_code}\\n" "$BASE_URL/todos")
      echo "$LIST_RESP"

      # Separar body y status
      LIST_STATUS=$(echo "$LIST_RESP" | sed -n 's/^HTTP_STATUS://p')
      LIST_BODY=$(echo "$LIST_RESP" | sed '/^HTTP_STATUS:/d')

      if [ "$LIST_STATUS" != "200" ]; then
        echo "ERROR: GET /todos devolvió status $LIST_STATUS"
        exit 1
      fi

      # Obtener un id si hay items (sin jq, usando python)
      TODO_ID=$(python3 - <<'PY'
import json, sys
body = sys.stdin.read().strip()
try:
    data = json.loads(body)
except Exception:
    print("")
    sys.exit(0)

# esperamos lista de todos
if isinstance(data, list) and len(data) > 0 and isinstance(data[0], dict) and 'id' in data[0]:
    print(data[0]['id'])
else:
    print("")
PY
      <<EOF
$LIST_BODY
EOF
)

      if [ -z "$TODO_ID" ]; then
        echo "INFO: No hay todos en producción. No se puede ejecutar GET /todos/{id} sin modificar datos."
        echo "INFO: La prueba de 'listar' se considera OK (HTTP 200)."
        exit 0
      fi

      echo "Usando TODO_ID=$TODO_ID"

      echo "---- curl: GET /todos/$TODO_ID ----"
      GET_RESP=$(curl -sS -w "\\nHTTP_STATUS:%{http_code}\\n" "$BASE_URL/todos/$TODO_ID")
      echo "$GET_RESP"

      GET_STATUS=$(echo "$GET_RESP" | sed -n 's/^HTTP_STATUS://p')
      GET_BODY=$(echo "$GET_RESP" | sed '/^HTTP_STATUS:/d')

      if [ "$GET_STATUS" != "200" ]; then
        echo "ERROR: GET /todos/$TODO_ID devolvió status $GET_STATUS"
        exit 1
      fi

      # Validación mínima: que el JSON devuelto tenga id = TODO_ID
      python3 - <<PY
import json, sys
todo_id = "$TODO_ID"
body = sys.stdin.read().strip()
data = json.loads(body)
assert str(data.get("id")) == str(todo_id), f"id devuelto {data.get('id')} != esperado {todo_id}"
print("OK: GET /todos/{id} devuelve el id esperado")
PY
      <<EOF
$GET_BODY
EOF

      echo "=== Rest Test (READ-ONLY) OK ==="
    '''
  }
}

    // =========================
    // Promote solo en develop
    // =========================
    stage('Promote') {
      when { branch 'develop' }
      steps {
        withCredentials([
          usernamePassword(
            credentialsId: 'github-pat',
            usernameVariable: 'GIT_USER',
            passwordVariable: 'GIT_TOKEN'
          )
        ]) {
          sh '''
            set -e
            echo "=== Promote: merge develop -> master ==="

            git config user.email "jenkins@local"
            git config user.name "jenkins"

            # Asegurar refs
            git fetch origin --prune

            # Traer origin/master explícitamente (en multibranch a veces no existe en el workspace)
            git fetch origin master:refs/remotes/origin/master

            # Dejar master igual al remoto
            git checkout -B master refs/remotes/origin/master

            # Merge desde develop (remota)
            git merge --no-ff origin/develop -m "Promote: merge develop into master"

            # Push a master
            git push https://$GIT_USER:$GIT_TOKEN@github.com/Al3jandr00/todo-list-aws.git master

            echo "=== Promote completed successfully ==="
          '''
        }
      }
    }
  }
}
