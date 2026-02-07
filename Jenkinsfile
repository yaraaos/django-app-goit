pipeline {
  agent any

  environment {
    AWS_REGION = "eu-central-1"

    ECR_REPO_NAME    = "django"
    HELM_REPO_HTTPS  = "https://github.com/yaraaos/helm-charts-goit.git"
    HELM_VALUES_PATH = "django-app/values.yaml"
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

cat <<YAML | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: ${POD}
  namespace: ci
spec:
  serviceAccountName: ci-kaniko
  restartPolicy: Never
  containers:
  - name: kaniko
    image: gcr.io/kaniko-project/executor:latest
    args:
      - --context=git://github.com/yaraaos/django-app-goit.git#refs/heads/main
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

kubectl -n ci wait --for=condition=Ready pod/${POD} --timeout=180s || true
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

ECR_IMAGE=$(cat image.txt)
IMAGE_REPO=$(echo "$ECR_IMAGE" | cut -d: -f1)
IMAGE_TAG=$(echo "$ECR_IMAGE" | cut -d: -f2)

rm -rf helm-repo
echo "Cloning helm repo..."
git -c http.extraHeader="AUTHORIZATION: bearer ${GITHUB_TOKEN}" \
  clone "${HELM_REPO_HTTPS}" helm-repo
cd helm-repo

test -f "${HELM_VALUES_PATH}" || (echo "❌ values file not found: ${HELM_VALUES_PATH}" && find . -maxdepth 4 -name values.yaml && exit 1)

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
git commit -m "chore: bump image tag to ${IMAGE_TAG}" || echo "No changes"
git push origin main
'''
        }
      }
    }
  }
}
