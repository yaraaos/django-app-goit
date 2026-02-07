pipeline {
  agent any

  environment {
    AWS_REGION = "eu-central-1"

    // ECR repo name
    ECR_REPO_NAME = "django"

    // Helm repo + path to values.yaml INSIDE that repo
    HELM_REPO_HTTPS  = "https://github.com/yaraaos/helm-charts-goit.git"
    HELM_VALUES_PATH = "django-app/values.yaml"

    // App repo for Kaniko build context
    APP_REPO_GIT     = "git://github.com/yaraaos/django-app-goit.git"
    APP_REPO_GIT_REF = "refs/heads/main"
  }

  stages {
    stage('Checkout app repo') {
      steps { checkout scm }
    }

    stage('Build + Push (Kaniko in K8s)') {
      steps {
        withCredentials([
          string(credentialsId: 'aws-access-key-id', variable: 'AWS_ACCESS_KEY_ID'),
          string(credentialsId: 'aws-secret-access-key', variable: 'AWS_SECRET_ACCESS_KEY')
        ]) {
          sh '''
set -euo pipefail

kubectl get ns ci >/dev/null 2>&1 || kubectl create ns ci

IMAGE_TAG=$(git rev-parse --short HEAD)
AWS_ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
ECR_REGISTRY="${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com"
ECR_IMAGE="${ECR_REGISTRY}/${ECR_REPO_NAME}:${IMAGE_TAG}"

aws ecr describe-repositories --region ${AWS_REGION} --repository-names ${ECR_REPO_NAME} >/dev/null 2>&1 \
 || aws ecr create-repository --region ${AWS_REGION} --repository-name ${ECR_REPO_NAME} >/dev/null

TOKEN=$(aws ecr get-login-password --region ${AWS_REGION})
kubectl -n ci delete secret ecr-docker-config >/dev/null 2>&1 || true
kubectl -n ci create secret generic ecr-docker-config \
  --from-literal=config.json="{\\"auths\\":{\\"${ECR_REGISTRY}\\":{\\"username\\":\\"AWS\\",\\"password\\":\\"${TOKEN}\\"}}}"

POD="kaniko-${BUILD_NUMBER}-${IMAGE_TAG}"
kubectl -n ci delete pod ${POD} --ignore-not-found=true >/dev/null 2>&1 || true

cat <<YAML | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: ${POD}
  namespace: ci
  labels:
    app: kaniko
spec:
  serviceAccountName: ci-kaniko
  restartPolicy: Never
  containers:
  - name: kaniko
    image: gcr.io/kaniko-project/executor:latest
    args:
      - --context=${APP_REPO_GIT}#${APP_REPO_GIT_REF}
      - --dockerfile=Dockerfile
      - --destination=${ECR_IMAGE}
    volumeMounts:
      - name: docker-config
        mountPath: /kaniko/.docker
  volumes:
    - name: docker-config
      secret:
        secretName: ecr-docker-config
YAML

kubectl -n ci logs -f pod/${POD} -c kaniko || true

PHASE=$(kubectl -n ci get pod ${POD} -o jsonpath='{.status.phase}')
echo "Kaniko pod phase: ${PHASE}"
test "${PHASE}" = "Succeeded"

echo "${ECR_IMAGE}" > image.txt
kubectl -n ci delete pod ${POD} --ignore-not-found=true
'''
        }
      }
    }

    stage('Update Helm repo + push') {
      steps {
        withCredentials([string(credentialsId: 'github-token', variable: 'GITHUB_TOKEN')]) {
          sh '''
set -euo pipefail

# 1) Build image vars
ECR_IMAGE=$(cat image.txt)
IMAGE_REPO=$(echo "$ECR_IMAGE" | cut -d: -f1)
IMAGE_TAG=$(echo "$ECR_IMAGE" | cut -d: -f2)

# 2) URL-encode token to avoid "Malformed URL"
ENC_TOKEN=$(python3 - <<'PY'
import os, urllib.parse
print(urllib.parse.quote(os.environ["GITHUB_TOKEN"], safe=""))
PY
)

# 3) Create an authenticated URL that never prompts
CLONE_URL="https://x-access-token:${ENC_TOKEN}@${HELM_REPO_HTTPS#https://}"

rm -rf helm-repo
echo "Cloning Helm repo..."
git clone "${CLONE_URL}" helm-repo

cd helm-repo

# 4) Update values.yaml (image.repository + image.tag)
python3 - <<PY
import yaml
path = "${HELM_VALUES_PATH}"
with open(path) as f:
    data = yaml.safe_load(f) or {}
data.setdefault("image", {})
data["image"]["repository"] = "${IMAGE_REPO}"
data["image"]["tag"] = "${IMAGE_TAG}"
with open(path, "w") as f:
    yaml.safe_dump(data, f, sort_keys=False)
PY

git config user.email "jenkins@local"
git config user.name "jenkins"
git add "${HELM_VALUES_PATH}"
git commit -m "chore: bump image tag to ${IMAGE_TAG}" || echo "No changes to commit"

# 5) Ensure push uses the same authenticated URL (no prompts)
git remote set-url origin "${CLONE_URL}"

echo "Pushing to main..."
git push origin HEAD:main
'''
        }
      }
    }
  }

  post {
    always {
      sh '''
set +e
kubectl -n ci delete pod -l app=kaniko --ignore-not-found=true >/dev/null 2>&1 || true
'''
    }
  }
}
