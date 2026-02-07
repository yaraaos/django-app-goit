pipeline {
  agent any

  environment {
    AWS_REGION = "eu-central-1"

    // ECR repo name (will be created if missing)
    ECR_REPO_NAME = "django"

    // Your Helm repo (Repo B)
    HELM_REPO_HTTPS  = "https://github.com/yaraaos/helm-charts-goit.git"

    // Path INSIDE the Helm repo to the values.yaml that has image.repository + image.tag
    HELM_VALUES_PATH = "django-app/values.yaml"

    // Git context of your APP repo (Repo A) for Kaniko
    APP_GIT_CONTEXT  = "git://github.com/yaraaos/django-app-goit.git#refs/heads/main"
  }

  stages {

    stage('Sanity check tools') {
      steps {
        sh '''
set -euo pipefail
echo "Checking tools..."
command -v aws >/dev/null
command -v kubectl >/dev/null
command -v git >/dev/null
command -v python3 >/dev/null
python3 -c "import yaml; print('pyyaml ok')" >/dev/null 2>&1 || true
echo "aws: $(aws --version)"
echo "kubectl: $(kubectl version --client --short 2>/dev/null || true)"
'''
      }
    }

    stage('Prepare CI namespace + RBAC') {
      steps {
        sh '''
set -euo pipefail

kubectl get ns ci >/dev/null 2>&1 || kubectl create ns ci

cat <<'YAML' | kubectl apply -f -
apiVersion: v1
kind: ServiceAccount
metadata:
  name: ci-kaniko
  namespace: ci
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: ci-kaniko-role
  namespace: ci
rules:
- apiGroups: [""]
  resources: ["pods","pods/log","pods/exec","secrets","configmaps"]
  verbs: ["get","list","watch","create","delete","patch","update"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: ci-kaniko-binding
  namespace: ci
subjects:
- kind: ServiceAccount
  name: ci-kaniko
  namespace: ci
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: ci-kaniko-role
YAML
'''
      }
    }

    stage('Build + Push (Kaniko in K8s)') {
      steps {
        withCredentials([
          string(credentialsId: 'aws-access-key-id', variable: 'AWS_ACCESS_KEY_ID'),
          string(credentialsId: 'aws-secret-access-key', variable: 'AWS_SECRET_ACCESS_KEY')
        ]) {
          sh '''
set -euo pipefail

# Use commit SHA from Jenkins checkout if available; otherwise a random short tag
IMAGE_TAG="$(git rev-parse --short HEAD 2>/dev/null || true)"
if [ -z "${IMAGE_TAG}" ]; then
  IMAGE_TAG="$(date +%s)"
fi

AWS_ACCOUNT_ID="$(aws sts get-caller-identity --query Account --output text)"
ECR_REGISTRY="${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com"
ECR_IMAGE="${ECR_REGISTRY}/${ECR_REPO_NAME}:${IMAGE_TAG}"

# Ensure ECR repo exists
aws ecr describe-repositories --region "${AWS_REGION}" --repository-names "${ECR_REPO_NAME}" >/dev/null 2>&1 \
  || aws ecr create-repository --region "${AWS_REGION}" --repository-name "${ECR_REPO_NAME}" >/dev/null

# Create/update docker config secret for Kaniko
TOKEN="$(aws ecr get-login-password --region "${AWS_REGION}")"
kubectl -n ci delete secret ecr-docker-config >/dev/null 2>&1 || true
kubectl -n ci create secret generic ecr-docker-config \
  --from-literal=config.json="{\\"auths\\":{\\"${ECR_REGISTRY}\\":{\\"username\\":\\"AWS\\",\\"password\\":\\"${TOKEN}\\"}}}"

# Clean any leftover pod name collision
POD="kaniko-${BUILD_NUMBER}-${IMAGE_TAG}"
kubectl -n ci delete pod "${POD}" --ignore-not-found=true >/dev/null 2>&1 || true

# Launch Kaniko pod (GIT context — NO hostPath)
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
      - --context=${APP_GIT_CONTEXT}
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

# Stream logs until completion
kubectl -n ci logs -f "pod/${POD}" -c kaniko || true

PHASE="$(kubectl -n ci get pod "${POD}" -o jsonpath='{.status.phase}')"
echo "Kaniko pod phase: ${PHASE}"
test "${PHASE}" = "Succeeded"

echo "${ECR_IMAGE}" > image.txt

kubectl -n ci delete pod "${POD}" --ignore-not-found=true
'''
        }
      }
    }

    stage('Update Helm repo + push') {
      steps {
        withCredentials([string(credentialsId: 'github-token', variable: 'GITHUB_TOKEN')]) {
          sh '''
set -euo pipefail
export GIT_TERMINAL_PROMPT=0

ECR_IMAGE="$(cat image.txt)"
IMAGE_REPO="$(echo "$ECR_IMAGE" | cut -d: -f1)"
IMAGE_TAG="$(echo "$ECR_IMAGE" | cut -d: -f2)"

rm -rf helm-repo

# GitHub HTTPS auth (reliable): Basic header with x-access-token:<PAT>
AUTH="$(printf "x-access-token:%s" "$GITHUB_TOKEN" | base64 -w0)"

echo "Cloning helm repo..."
git -c http.https://github.com/.extraHeader="AUTHORIZATION: Basic ${AUTH}" \
    clone "${HELM_REPO_HTTPS}" helm-repo

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
print("Updated:", path)
PY

git config user.email "jenkins@local"
git config user.name "jenkins"
git add "${HELM_VALUES_PATH}"
git commit -m "chore: bump image tag to ${IMAGE_TAG}" || echo "No changes"

echo "Pushing to main..."
git -c http.https://github.com/.extraHeader="AUTHORIZATION: Basic ${AUTH}" \
    push origin HEAD:main
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
