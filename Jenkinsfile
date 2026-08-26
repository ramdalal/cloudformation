pipeline {
  agent any

  environment {
    // The ARTIFACT bucket's region. This is where every template is archived for audit, and
    // it does not move — it is NOT necessarily the region a template deploys into.
    BUCKET_REGION = 'eu-central-1'
    // Where a template deploys when it does not say. Templates composed by the G.I.A CI/CD
    // console carry their target region in Metadata.GIA.region; see regionFor() below.
    DEFAULT_DEPLOY_REGION = 'eu-central-1'
    // Regions this pipeline is allowed to deploy into — the union of the app's account/region
    // map (Intertrade's APAC set plus Frankfurt). An allowlist rather than free-form, so a
    // typo in a template's metadata fails the build instead of creating a stack somewhere
    // nobody is looking.
    ALLOWED_REGIONS = 'eu-central-1 ap-east-1 ap-east-2 ap-south-1 ap-southeast-1 ap-southeast-2 ap-northeast-1'
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
            def region = regionFor(path)
            stage("deploy ${stack} (${region})") {
              deployTemplate(path, stack, region)
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

// NOTE: mirrored by hand in THREE places — here, `_stack_name_for` in the console's
// lambda_function.py, and `stackNameFor` in its cfnCompose.js. The console states the stack
// name before you commit, which is only useful if it agrees with this. Change all three.
def stackNameFor(String path) {
  def base = path.tokenize('/').last().replaceFirst(/(?i)\.ya?ml$/, '')
  def name = base.replaceAll(/[^a-zA-Z0-9-]/, '-')
                 .replaceAll(/-+/, '-')
                 .replaceAll(/^-+|-+$/, '')
  if (!(name ==~ /^[a-zA-Z].*/)) name = "stack-${name}"
  return name.length() > 128 ? name.substring(0, 128) : name
}

/**
 * Which region a template deploys into.
 *
 * Read from `Metadata.GIA.region`, which the G.I.A CI/CD console writes into every template it
 * composes, alongside the baseline and account it was built from. That makes the target region a
 * property OF THE TEMPLATE rather than of the pipeline: previously this whole job was pinned to
 * eu-central-1, so a template composed from Singapore inventory deployed to Frankfurt and failed
 * on resource ids that do not exist there.
 *
 * A template with no GIA metadata (anything hand-written) keeps the old behaviour via
 * DEFAULT_DEPLOY_REGION, so this change cannot move an existing stack.
 *
 * The value is checked against ALLOWED_REGIONS. A typo would otherwise create a stack in a
 * region nobody monitors, which is worse than a failed build.
 */
def regionFor(String path) {
  // Parsed in GROOVY, deliberately, not by shelling out to awk or sed. The first attempt at
  // this used awk and carried two bugs that a quick read would not catch: `{2}` interval
  // expressions are not portable (gawk honours them, mawk does not), and the character class
  // meant to strip quotes and spaces was written `["'[:space:]]`, which is a bracket list of
  // literal characters — so it stripped the letters a, c and e, quietly turning
  // "eu-central-1" into "eu-ntrl-1". readFile keeps it in one language with no shell quoting.
  def region = null
  def inMeta = false
  def inGia = false
  readFile(path).split('\n').each { line ->
    if (region != null) return
    if (line ==~ /^Metadata:\s*$/) { inMeta = true; return }
    // any other column-0 key ends the Metadata block
    if (line ==~ /^\S.*/) { inMeta = false; inGia = false; return }
    if (inMeta && line ==~ /^  GIA:\s*$/) { inGia = true; return }
    // a sibling of GIA at the same indent ends it
    if (inGia && line ==~ /^  \S.*/) { inGia = false; return }
    if (inGia) {
      def m = (line =~ /^\s+region:\s*(.+?)\s*$/)
      if (m) region = m[0][1].replaceAll(/^["']|["']$/, '').trim()
    }
  }

  if (!region) {
    echo "${path}: no Metadata.GIA.region — deploying to ${env.DEFAULT_DEPLOY_REGION}"
    return env.DEFAULT_DEPLOY_REGION
  }
  def found = region
  if (!(env.ALLOWED_REGIONS.split(' ').contains(found))) {
    error("${path}: Metadata.GIA.region is '${found}', which is not in ALLOWED_REGIONS " +
          "(${env.ALLOWED_REGIONS}). Add it there if that region is genuinely in use.")
  }
  echo "${path}: deploying to ${found} (from Metadata.GIA.region)"
  return found
}

def deployTemplate(String path, String stack, String deployRegion) {
  withEnv(["TPL=${path}", "STACK=${stack}", "DEPLOY_REGION=${deployRegion}"]) {
    sh '''
      set -euo pipefail 

      FILE=$(basename "$TPL")
      KEY="deployed/${STACK}/${GIT_SHA}/${FILE}"

      echo "--- deploying $TPL to $DEPLOY_REGION (artifact bucket is in $BUCKET_REGION)"

      # EVERY CloudFormation call below uses --template-body, not --template-url, and that is
      # what makes multi-region work. The artifact bucket lives in one region only; a
      # --template-url pointing at it is not reliably usable by CloudFormation in another
      # region. Passing the body inline removes that coupling entirely, at the cost of the
      # 51,200-byte inline ceiling — checked explicitly below, because the failure mode
      # otherwise is an opaque ValidationError deep in the deploy.
      SIZE=$(wc -c < "$TPL")
      if [ "$SIZE" -gt 51200 ]; then
        echo "!!! $TPL is ${SIZE} bytes, over the 51,200-byte inline limit."
        echo "    Deploying it needs an artifact bucket in $DEPLOY_REGION and --template-url."
        exit 1
      fi

      echo "--- validating $TPL"
      aws cloudformation validate-template \
        --template-body "file://${TPL}" --region "$DEPLOY_REGION" >/dev/null

      # The archive copy stays in the home-region bucket: it is an audit trail of what was
      # deployed, not the thing CloudFormation reads.
      echo "--- archiving to s3://${CFN_BUCKET}/${KEY}"
      aws s3 cp "$TPL" "s3://${CFN_BUCKET}/${KEY}" \
        --region "$BUCKET_REGION" --only-show-errors

      # capabilities the template actually needs
      CAPS=$(aws cloudformation get-template-summary \
               --template-body "file://${TPL}" --region "$DEPLOY_REGION" \
               --query 'Capabilities' --output text 2>/dev/null || true)
      CAP_ARG=""
      if [ -n "$CAPS" ] && [ "$CAPS" != "None" ]; then
        CAP_ARG="--capabilities $CAPS"
      fi

      # ── PARAMETERS ────────────────────────────────────────────────────────────────────
      #
      # update-stack does NOT inherit a stack's parameter values. Every parameter the template
      # declares must be given a value or carried over explicitly, and one with no Default and
      # no value fails the whole deploy with:
      #
      #   ValidationError: Parameters: [InstanceName] must have values
      #
      # which is what broke the FR0AECLEPIP001-Infra-Templates-Specs deploy. The stack HELD
      # InstanceName=FR0AECLEPIP001 the entire time; nothing asked for it. Every EC2 baseline
      # here declares InstanceName without a Default (deliberately — a default would let an
      # untouched template deploy a server named after the example), so this affects every
      # update of every EC2 stack, not just this one.
      #
      # So: carry over what the stack already has with UsePreviousValue, and pass the stack name
      # explicitly for PARAM_NAME as before. Declared-but-unheld parameters are named in the
      # failure rather than left to surface as CloudFormation's list.
      # `--output text` on a list returns ONE line of TAB-separated values, not spaces. `for W in
      # $WANTS` splits on tabs happily (IFS has one), but a `case " $HELD " in *" $W "*)`
      # membership test does NOT — the separator either side of a name is a tab, so the pattern
      # never matches and every parameter looks unheld. That is precisely how this went out
      # wrong the first time: the logic was checked against hand-written space-separated strings
      # rather than against what the command actually emits, so it passed a test and failed the
      # build. Normalised to spaces at the source, once, so every test below is a space test.
      WANTS=$(aws cloudformation get-template-summary \
                --template-body "file://${TPL}" --region "$DEPLOY_REGION" \
                --query 'Parameters[].ParameterKey' --output text 2>/dev/null | tr '\t' ' ' || true)
      NODEFAULT=$(aws cloudformation get-template-summary \
                --template-body "file://${TPL}" --region "$DEPLOY_REGION" \
                --query 'Parameters[?DefaultValue==null].ParameterKey' --output text 2>/dev/null | tr '\t' ' ' || true)
      HELD=$(aws cloudformation describe-stacks \
               --stack-name "$STACK" --region "$DEPLOY_REGION" \
               --query 'Stacks[0].Parameters[].ParameterKey' --output text 2>/dev/null | tr '\t' ' ' || true)

      # Built separately for create and update: on a create there is no previous value to use,
      # so UsePreviousValue is not merely wrong there, it is rejected.
      CREATE_PARAMS=""
      UPDATE_PARAMS=""
      MISSING=""
      for W in $WANTS; do
        if [ "$W" = "${PARAM_NAME}" ]; then
          CREATE_PARAMS="$CREATE_PARAMS ParameterKey=${W},ParameterValue=${STACK}"
          UPDATE_PARAMS="$UPDATE_PARAMS ParameterKey=${W},ParameterValue=${STACK}"
          continue
        fi
        case " $HELD " in
          *" $W "*) UPDATE_PARAMS="$UPDATE_PARAMS ParameterKey=${W},UsePreviousValue=true" ;;
          *)
            # Not held by the stack. Fine if the template defaults it; fatal if not.
            case " $NODEFAULT " in
              *" $W "*) MISSING="$MISSING $W" ;;
            esac
            ;;
        esac
      done

      PARAM_ARG=""
      if [ -n "$CREATE_PARAMS" ]; then
        PARAM_ARG="--parameters$CREATE_PARAMS"
      fi
      UPDATE_PARAM_ARG=""
      if [ -n "$UPDATE_PARAMS" ]; then
        UPDATE_PARAM_ARG="--parameters$UPDATE_PARAMS"
      fi

      STATUS=$(aws cloudformation describe-stacks \
                 --stack-name "$STACK" --region "$DEPLOY_REGION" \
                 --query 'Stacks[0].StackStatus' --output text 2>/dev/null || echo "NONE")

      # a stack stuck in ROLLBACK_COMPLETE can never be updated, only deleted
      if [ "$STATUS" = "ROLLBACK_COMPLETE" ]; then
        echo "--- $STACK is in ROLLBACK_COMPLETE; deleting before recreate"
        aws cloudformation delete-stack --stack-name "$STACK" --region "$DEPLOY_REGION"
        aws cloudformation wait stack-delete-complete \
          --stack-name "$STACK" --region "$DEPLOY_REGION"
        STATUS="NONE"
      fi

      # Said before either call, naming the parameter, because CloudFormation's own message
      # ("Parameters: [X] must have values") does not say that the fix is a Default or a value.
      if [ "$STATUS" != "NONE" ] && [ -n "$MISSING" ]; then
        echo "!!! $STACK declares parameter(s) with no Default that the stack has no value for:$MISSING" >&2
        echo "!!! An update cannot supply them. Give each a Default in the template, or set the" >&2
        echo "!!! value on the stack first." >&2
        exit 2
      fi

      if [ "$STATUS" = "NONE" ]; then
        echo "--- creating stack $STACK"
        aws cloudformation create-stack \
          --stack-name "$STACK" \
          --template-body "file://${TPL}" \
          --region "$DEPLOY_REGION" \
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
                --template-body "file://${TPL}" \
                --region "$DEPLOY_REGION" \
                $CAP_ARG $UPDATE_PARAM_ARG 2>&1)
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
             --stack-name "$STACK" --region "$DEPLOY_REGION"; then
        echo "!!! $STACK failed — recent events:"
        aws cloudformation describe-stack-events \
          --stack-name "$STACK" --region "$DEPLOY_REGION" \
          --max-items 25 \
          --query 'StackEvents[?ResourceStatusReason!=null].[Timestamp,LogicalResourceId,ResourceStatus,ResourceStatusReason]' \
          --output table
        exit 1
      fi

      echo "--- $STACK deployed"
      aws cloudformation describe-stacks \
        --stack-name "$STACK" --region "$DEPLOY_REGION" \
        --query 'Stacks[0].Outputs' --output table 2>/dev/null || true
    '''
  }
}
