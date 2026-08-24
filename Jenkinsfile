pipeline {
  agent any

  environment {
    AWS_REGION   = 'eu-central-1'
    CFN_BUCKET   = 'triumph-platform-cloudformation-templates'
    TEMPLATE_DIR = 'templates'
    PARAM_NAME   = 'ServerName'   // parameter in your template for the instance Name tag
  }

  options {
    timestamps()
    buildDiscarder(logRotator(numToKeepStr: '20'))
    disableConcurrentBuilds()
   
  }

  triggers { githubPush() }

  stages {

    stage('Checkout') {
      when { branch 'main' }
      steps {
        checkout scm
        script {
          env.GIT_SHA = sh(script: 'git rev-parse --short HEAD', returnStdout: true).trim()
        }
      }
    }

    stage('Detect changed templates') {
      when { branch 'main' }
      steps {
        script {
          def base = env.GIT_PREVIOUS_SUCCESSFUL_COMMIT
          def range = base ? "${base}..HEAD" : 'HEAD~1..HEAD'

          // first build on a fresh branch has no parent to diff against
          def ok = sh(script: "git rev-parse --verify --quiet ${range.split('\\.\\.')[0]}",
                      returnStatus: true) == 0
          def changed
          if (ok) {
            changed = sh(script: "git diff --name-only --diff-filter=ACMR ${range}",
                         returnStdout: true).trim()
          } else {
            echo 'No usable diff base — treating all templates as changed.'
            changed = sh(script: "git ls-files '${TEMPLATE_DIR}'",
                         returnStdout: true).trim()
          }

          def templates = changed.split('\n')
              .findAll { it && it ==~ /^${env.TEMPLATE_DIR}\/.*\.(ya?ml)$/ }
              .sort()

          if (!templates) {
            echo 'No CloudFormation templates changed. Nothing to deploy.'
            env.TEMPLATES = ''
          } else {
            echo "Templates to deploy:\n  ${templates.join('\n  ')}"
            env.TEMPLATES = templates.join(' ')
          }
        }
      }
    }

    stage('Deploy') {
      when {
        allOf {
          branch 'main'
          expression { env.TEMPLATES?.trim() }
        }
      }
      steps {
        script {
          env.TEMPLATES.split(' ').each { path ->
            def stack = stackNameFor(path)
            stage("deploy ${stack}") {
              deployTemplate(path, stack)
            }
          }
        }
      }
    }
  }

  post {
    failure {
      echo 'Deploy failed — see the stack events dumped above.'
    }
  }
}

// ---- helpers ----

def stackNameFor(String path) {
  def base = path.tokenize('/').last().replaceFirst(/(?i)\.ya?ml$/, '')
  def name = base.replaceAll(/[^a-zA-Z0-9-]/, '-')
                 .replaceAll(/-+/, '-')
                 .replaceAll(/^-+|-+$/, '')
  if (!(name ==~ /^[a-zA-Z].*/)) name = "stack-${name}"
  return name.length() > 128 ? name.substring(0, 128) : name
}

def deployTemplate(String path, String stack) {
  withEnv(["TPL=${path}", "STACK=${stack}"]) {
    sh '''
      set -euo pipefail 

      FILE=$(basename "$TPL")
      KEY="deployed/${STACK}/${GIT_SHA}/${FILE}"
      URL="https://${CFN_BUCKET}.s3.${AWS_REGION}.amazonaws.com/${KEY}"

      echo "--- validating $TPL"
      aws cloudformation validate-template \
        --template-body "file://${TPL}" --region "$AWS_REGION" >/dev/null

      echo "--- uploading to s3://${CFN_BUCKET}/${KEY}"
      aws s3 cp "$TPL" "s3://${CFN_BUCKET}/${KEY}" \
        --region "$AWS_REGION" --only-show-errors

      # capabilities the template actually needs
      CAPS=$(aws cloudformation get-template-summary \
               --template-url "$URL" --region "$AWS_REGION" \
               --query 'Capabilities' --output text 2>/dev/null || true)
      CAP_ARG=""
      if [ -n "$CAPS" ] && [ "$CAPS" != "None" ]; then
        CAP_ARG="--capabilities $CAPS"
      fi

      # pass the stack name in as a parameter, only if the template declares it
      HAS=$(aws cloudformation get-template-summary \
              --template-url "$URL" --region "$AWS_REGION" \
              --query "length(Parameters[?ParameterKey=='${PARAM_NAME}'])" \
              --output text)
      PARAM_ARG=""
      if [ "$HAS" != "0" ]; then
        PARAM_ARG="--parameters ParameterKey=${PARAM_NAME},ParameterValue=${STACK}"
      fi

      STATUS=$(aws cloudformation describe-stacks \
                 --stack-name "$STACK" --region "$AWS_REGION" \
                 --query 'Stacks[0].StackStatus' --output text 2>/dev/null || echo "NONE")

      # a stack stuck in ROLLBACK_COMPLETE can never be updated, only deleted
      if [ "$STATUS" = "ROLLBACK_COMPLETE" ]; then
        echo "--- $STACK is in ROLLBACK_COMPLETE; deleting before recreate"
        aws cloudformation delete-stack --stack-name "$STACK" --region "$AWS_REGION"
        aws cloudformation wait stack-delete-complete \
          --stack-name "$STACK" --region "$AWS_REGION"
        STATUS="NONE"
      fi

      if [ "$STATUS" = "NONE" ]; then
        echo "--- creating stack $STACK"
        aws cloudformation create-stack \
          --stack-name "$STACK" \
          --template-url "$URL" \
          --region "$AWS_REGION" \
          --on-failure ROLLBACK \
          --tags Key=ManagedBy,Value=jenkins \
                 Key=GitSha,Value=${GIT_SHA} \
                 Key=Source,Value=${FILE} \
          $CAP_ARG $PARAM_ARG
        WAITER=stack-create-complete
      else
        echo "--- updating stack $STACK (current: $STATUS)"
        set +e
        OUT=$(aws cloudformation update-stack \
                --stack-name "$STACK" \
                --template-url "$URL" \
                --region "$AWS_REGION" \
                $CAP_ARG $PARAM_ARG 2>&1)
        RC=$?
        set -e
        if [ $RC -ne 0 ]; then
          if echo "$OUT" | grep -q "No updates are to be performed"; then
            echo "--- no changes for $STACK, skipping"
            exit 0
          fi
          echo "$OUT" >&2
          exit $RC
        fi
        WAITER=stack-update-complete
      fi

      echo "--- waiting for $WAITER"
      if ! aws cloudformation wait $WAITER \
             --stack-name "$STACK" --region "$AWS_REGION"; then
        echo "!!! $STACK failed — recent events:"
        aws cloudformation describe-stack-events \
          --stack-name "$STACK" --region "$AWS_REGION" \
          --max-items 25 \
          --query 'StackEvents[?ResourceStatusReason!=null].[Timestamp,LogicalResourceId,ResourceStatus,ResourceStatusReason]' \
          --output table
        exit 1
      fi

      echo "--- $STACK deployed"
      aws cloudformation describe-stacks \
        --stack-name "$STACK" --region "$AWS_REGION" \
        --query 'Stacks[0].Outputs' --output table 2>/dev/null || true
    '''
  }
}
