pipeline {
  agent any

  environment {
    AWS_REGION      = "eu-central-1"
    ECR_REPO_NAME   = "django"

    // Helm repo that contains your chart
    HELM_REPO_HTTPS  = "https://github.com/yaraaos/helm-charts-goit.git"
    HELM_VALUES_PATH = "django-app/values.yaml"

    // Jenkins Credentials IDs (must exist in Jenkins)
    AWS_KEY_CRED     = "aws-access-key-id"
    AWS_SECRET_CRED  = "aws-secret-access-key"
    GITHUB_TOKEN_CRED = "github-token" // Secret Text credential (PAT)
  }

  stages {
    stage('Checkout app repo') {
      steps { checkout scm }
    }

    stage('Build + Push (Kaniko in K8s)') {
      steps {
        withCredentials([
          string(credentialsId: "${AWS_KEY_CRED}",    variable: 'AWS_ACCESS_KEY_ID'),
          string(credentialsId: "${AWS_SECRET_CRED}", variable: 'AWS_SECRET_ACCESS_KEY')
        ]) {
          sh '''#!/bin/bash
set -euo pipefail

# Ensure ci namespace + SA/RBAC exist (safe to re-apply)
kubectl get ns ci >/dev/null 2>&1 || kubectl create ns ci >/dev/null

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

IMAGE_TAG="$(git rev-parse --short HEAD)"
AWS_ACCOUNT_ID="$(aws sts get-caller-identity --query Account --output text)"
ECR_REGISTRY="${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com"
ECR_IMAGE="${ECR_REGISTRY}/${ECR_REPO_NAME}:${IMAGE_TAG}"

aws ecr describe-repositories --region "${AWS_REGION}" --repository-names "${ECR_REPO_NAME}" >/dev/null 2>&1 \
  || aws ecr create-repository --region "${AWS_REGION}" --repository-name "${ECR_REPO_NAME}" >/dev/null

TOKEN="$(aws ecr get-login-password --region "${AWS_REGION}")"
kubectl -n ci delete secret ecr-docker-config >/dev/null 2>&1 || true
kubectl -n ci create secret generic ecr-docker-config \
  --from-literal=config.json="{\\"auths\\":{\\"${ECR_REGISTRY}\\":{\\"username\\":\\"AWS\\",\\"password\\":\\"${TOKEN}\\"}}}"

POD="kaniko-${BUILD_NUMBER}-${IMAGE_TAG}"
kubectl -n ci delete pod "${POD}" --ignore-not-found=true >/dev/null 2>&1 || true

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

# Wait until Pod is Succeeded or Failed
echo "Waiting for Kaniko pod ${POD} to finish..."
for i in $(seq 1 180); do
  PHASE="$(kubectl -n ci get pod "${POD}" -o jsonpath='{.status.phase}' 2>/dev/null || true)"
  echo "phase=${PHASE}"
  if [[ "${PHASE}" == "Succeeded" || "${PHASE}" == "Failed" ]]; then
    break
  fi
  sleep 2
done

echo "---- Kaniko logs ----"
kubectl -n ci logs pod/"${POD}" -c kaniko || true

PHASE="$(kubectl -n ci get pod "${POD}" -o jsonpath='{.status.phase}')"
echo "Kaniko pod final phase: ${PHASE}"
[[ "${PHASE}" == "Succeeded" ]]

echo "${ECR_IMAGE}" > image.txt
kubectl -n ci delete pod "${POD}" --ignore-not-found=true >/dev/null 2>&1 || true
'''
        }
      }
    }

    stage('Update Helm repo + push') {
      steps {
        withCredentials([string(credentialsId: "${GITHUB_TOKEN_CRED}", variable: 'GITHUB_TOKEN')]) {
          sh '''#!/bin/bash
set -euo pipefail

ECR_IMAGE="$(cat image.txt)"
IMAGE_REPO="${ECR_IMAGE%:*}"
IMAGE_TAG="${ECR_IMAGE##*:}"

rm -rf helm-repo

# Clone using Authorization header (works with GitHub PAT, avoids malformed URL issues)
AUTH_B64="$(printf "x-access-token:%s" "$GITHUB_TOKEN" | base64 -w0)"
git -c http.https://github.com/.extraheader="AUTHORIZATION: basic ${AUTH_B64}" \
  clone "${HELM_REPO_HTTPS}" helm-repo

cd helm-repo

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
print("Updated", path, "->", "${IMAGE_REPO}", "${IMAGE_TAG}")
PY

git config user.email "jenkins@local"
git config user.name "jenkins"
git add "${HELM_VALUES_PATH}"
git commit -m "chore: bump image tag to ${IMAGE_TAG}" || echo "No changes to commit"

# Push using same auth header
git -c http.https://github.com/.extraheader="AUTHORIZATION: basic ${AUTH_B64}" \
  push origin HEAD:main
'''
        }
      }
    }
  }

  post {
    always {
      sh '''#!/bin/bash
set +e
kubectl -n ci delete pod -l app=kaniko --ignore-not-found=true >/dev/null 2>&1
'''
    }
  }
}
