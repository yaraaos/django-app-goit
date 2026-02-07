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

# Ensure ECR repo exists
aws ecr describe-repositories --region ${AWS_REGION} --repository-names ${ECR_REPO_NAME} >/dev/null 2>&1 \
 || aws ecr create-repository --region ${AWS_REGION} --repository-name ${ECR_REPO_NAME} >/dev/null

# Create/update docker config secret for Kaniko
TOKEN=$(aws ecr get-login-password --region ${AWS_REGION})
kubectl -n ci delete secret ecr-docker-config >/dev/null 2>&1 || true
kubectl -n ci create secret generic ecr-docker-config \
  --from-literal=config.json="{\\"auths\\":{\\"${ECR_REGISTRY}\\":{\\"username\\":\\"AWS\\",\\"password\\":\\"${TOKEN}\\"}}}"

# Unique pod name
POD="kaniko-${BUILD_NUMBER}-${IMAGE_TAG}"

# Launch Kaniko pod (GIT context, no hostPath)
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

# Wait for completion + stream logs
kubectl -n ci wait --for=condition=Ready pod/${POD} --timeout=180s || true
kubectl -n ci logs -f pod/${POD} -c kaniko || true

# Ensure pod succeeded
PHASE=$(kubectl -n ci get pod ${POD} -o jsonpath='{.status.phase}')
echo "Kaniko pod phase: ${PHASE}"
test "${PHASE}" = "Succeeded"

# Save image reference for next stage
echo "${ECR_IMAGE}" > image.txt

# Cleanup
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
git clone https://${GITHUB_TOKEN}@${HELM_REPO_HTTPS#https://} helm-repo
cd helm-repo

# Update values.yaml (image.repository + image.tag)
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
