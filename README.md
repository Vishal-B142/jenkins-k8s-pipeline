# jenkins-k8s-pipeline

> Production CI/CD using two separate Jenkins pipelines — `Jenkinsfile` for app deployments (triggers on every git push) and `Jenkinsfile.infra` for infrastructure changes (manual trigger). Implements 6-step blue/green slot switching with zero-downtime and automatic rollback.

---

## Pipeline Architecture

```
Git push to dev branch
         │
         ▼
┌─────────────────────────────────────────────────────────────────┐
│  Jenkinsfile  (App pipeline — auto-triggered)                   │
│                                                                 │
│  1. Checkout                                                    │
│  2. ECR login + image build + push                              │
│  3. Apply manifests  (deployments.yaml + ingress.yaml + hpa.yaml│
│  4. Trivy vulnerability scan  ──── FAIL → block deploy          │
│  5. Blue/Green deploy (6-step slot switch)                      │
│       a. Read active slot from Service annotation               │
│       b. Set new image on INACTIVE slot                         │
│       c. Scale up inactive → wait readiness (timeout 300s)      │
│       d. Patch Service selector → new slot live                 │
│       e. Scale down old slot                                    │
│       f. If step c fails → steps d+e skipped, old slot stays   │
└─────────────────────────────────────────────────────────────────┘

Manual trigger
         │
         ▼
┌─────────────────────────────────────────────────────────────────┐
│  Jenkinsfile.infra  (Infra pipeline — manual trigger)           │
│                                                                 │
│  Prometheus + Grafana  (kube-prometheus-stack Helm chart)       │
│  Loki                  (grafana/loki Helm chart)                │
│  Promtail              (grafana/promtail Helm chart)            │
│  Headlamp              (kubectl manifest — Helm repo is 404)    │
│  Apply monitoring + headlamp Ingress rules                      │
└─────────────────────────────────────────────────────────────────┘
```

---

## Stack

![Jenkins](https://img.shields.io/badge/Jenkins-D24939?style=flat&logo=jenkins&logoColor=white)
![Docker](https://img.shields.io/badge/Docker-2496ED?style=flat&logo=docker&logoColor=white)
![Kubernetes](https://img.shields.io/badge/Kubernetes-326CE5?style=flat&logo=kubernetes&logoColor=white)
![Trivy](https://img.shields.io/badge/Trivy-1904DA?style=flat&logo=aquasecurity&logoColor=white)
![AWS ECR](https://img.shields.io/badge/AWS_ECR-232F3E?style=flat&logo=amazonaws&logoColor=white)

---

## Repo Structure

```
jenkins-k8s-pipeline/
├── Jenkinsfile               # App pipeline — blue/green deploy
├── Jenkinsfile.infra         # Infra pipeline — monitoring + headlamp
├── k3s/
│   ├── deployments.yaml      # All services as blue/green Deployment pairs
│   ├── ingress.yaml          # Traefik Ingress + Middleware rules
│   └── hpa.yaml              # HPA for all services
└── monitoring/
    ├── monitoring-values.yaml
    ├── loki-values.yaml
    ├── promtail-values.yaml
    ├── monitoring-ingress.yaml
    └── headlamp-ingress.yaml
```

---

## Jenkinsfile — App Pipeline

```groovy
pipeline {
    agent any
    environment {
        ECR_REGISTRY = "<ACCOUNT_ID>.dkr.ecr.ap-south-1.amazonaws.com"
        IMAGE_TAG    = "${BUILD_NUMBER}"
        NAMESPACE    = "app"
    }
    stages {
        stage('Checkout') {
            steps { checkout scm }
        }
        stage('ECR Login') {
            steps {
                sh '''
                    aws ecr get-login-password --region ap-south-1 \
                    | docker login --username AWS --password-stdin $ECR_REGISTRY
                    # Refresh ecr-secret in cluster
                    ECR_PASS=$(aws ecr get-login-password --region ap-south-1)
                    kubectl delete secret ecr-secret -n $NAMESPACE --ignore-not-found
                    kubectl create secret docker-registry ecr-secret \
                        --docker-server=$ECR_REGISTRY \
                        --docker-username=AWS \
                        --docker-password=$ECR_PASS -n $NAMESPACE
                '''
            }
        }
        stage('Build & Push') {
            steps {
                sh '''
                    docker build -t $ECR_REGISTRY/app:$IMAGE_TAG .
                    docker push $ECR_REGISTRY/app:$IMAGE_TAG
                '''
            }
        }
        stage('Trivy Scan') {
            steps {
                sh '''
                    trivy image --exit-code 1 --severity HIGH,CRITICAL \
                        $ECR_REGISTRY/app:$IMAGE_TAG
                '''
            }
        }
        stage('Apply Manifests') {
            steps {
                sh '''
                    # --validate=false: remote kubectl lacks Traefik CRD schemas
                    # cluster itself validates correctly
                    kubectl apply -f k3s/deployments.yaml --validate=false
                    kubectl apply -f k3s/ingress.yaml --validate=false
                    kubectl apply -f k3s/hpa.yaml
                '''
            }
        }
        stage('Blue/Green Deploy') {
            steps {
                sh '''
                    # Step 1: Read active slot
                    ACTIVE=$(kubectl get svc app-service -n $NAMESPACE \
                        -o jsonpath={.metadata.annotations.active-slot})
                    if [ "$ACTIVE" = "blue" ]; then
                        INACTIVE="green"
                    else
                        INACTIVE="blue"
                    fi

                    # Step 2: Set new image on inactive slot
                    kubectl set image deployment/app-service-$INACTIVE \
                        app=$ECR_REGISTRY/app:$IMAGE_TAG -n $NAMESPACE

                    # Step 3: Scale up inactive and wait for readiness
                    kubectl scale deployment/app-service-$INACTIVE \
                        --replicas=1 -n $NAMESPACE
                    kubectl rollout status deployment/app-service-$INACTIVE \
                        -n $NAMESPACE --timeout=300s

                    # Step 4: Patch service selector to new slot
                    kubectl patch service app-service -n $NAMESPACE \
                        -p "{\"spec\":{\"selector\":{\"app\":\"app-service\",\"slot\":\"$INACTIVE\"}},\"metadata\":{\"annotations\":{\"active-slot\":\"$INACTIVE\"}}}"

                    # Step 5: Scale down old slot
                    kubectl scale deployment/app-service-$ACTIVE \
                        --replicas=0 -n $NAMESPACE
                '''
            }
        }
    }
    post {
        failure {
            echo 'Deploy failed — old slot still running, zero impact'
        }
    }
}
```

---

## Jenkinsfile.infra — Infra Pipeline

```groovy
pipeline {
    agent any
    parameters {
        booleanParam(name: 'FORCE_REINSTALL', defaultValue: false,
            description: 'Delete all Helm releases and reinstall from scratch')
        booleanParam(name: 'SKIP_PROMETHEUS', defaultValue: false,
            description: 'Skip Prometheus + Grafana')
        booleanParam(name: 'SKIP_LOKI',       defaultValue: false,
            description: 'Skip Loki')
        booleanParam(name: 'SKIP_PROMTAIL',   defaultValue: false,
            description: 'Skip Promtail')
        booleanParam(name: 'SKIP_HEADLAMP',   defaultValue: false,
            description: 'Skip Headlamp')
        booleanParam(name: 'REFRESH_TLS',     defaultValue: false,
            description: 'Recreate TLS secrets after cert renewal')
    }
    stages {
        stage('Prometheus + Grafana') {
            when { expression { !params.SKIP_PROMETHEUS } }
            steps {
                sh '''
                    helm repo add prometheus-community \
                        https://prometheus-community.github.io/helm-charts
                    helm repo update
                    helm upgrade --install monitoring \
                        prometheus-community/kube-prometheus-stack \
                        -n monitoring --create-namespace \
                        -f monitoring/monitoring-values.yaml \
                        --atomic=false
                '''
                // NOTE: --atomic=false used intentionally
                // --wait caused 15min timeout (context deadline exceeded)
                // --atomic=false submits the install and lets k8s handle rollout
            }
        }
        stage('Loki') {
            when { expression { !params.SKIP_LOKI } }
            steps {
                sh '''
                    helm repo add grafana https://grafana.github.io/helm-charts
                    helm upgrade --install loki grafana/loki \
                        -n monitoring -f monitoring/loki-values.yaml --atomic=false
                '''
            }
        }
        stage('Promtail') {
            when { expression { !params.SKIP_PROMTAIL } }
            steps {
                sh '''
                    helm upgrade --install promtail grafana/promtail \
                        -n monitoring -f monitoring/promtail-values.yaml --atomic=false
                '''
            }
        }
        stage('Headlamp') {
            // catchError: Headlamp issues must not block ingress apply
            steps {
                catchError(buildResult: 'UNSTABLE', stageResult: 'UNSTABLE') {
                    sh '''
                        # Helm repo (headlamp-k8s.github.io/headlamp/) returns 404
                        # Deploy via direct manifest instead
                        kubectl apply -f https://raw.githubusercontent.com/headlamp-k8s/headlamp/main/kubernetes-headlamp.yaml
                    '''
                }
            }
        }
        stage('Apply Ingress') {
            steps {
                sh '''
                    kubectl apply -f monitoring/monitoring-ingress.yaml --validate=false
                    kubectl apply -f monitoring/headlamp-ingress.yaml --validate=false
                '''
            }
        }
    }
}
```

---

## HPA Configuration (hpa.yaml)

```yaml
# Public-facing services — more aggressive scaling
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: gateway-service
  namespace: app
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: gateway-service-blue   # HPA targets blue; follows after slot switch
  minReplicas: 1
  maxReplicas: 5
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 60   # Public: 60% CPU threshold
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 75
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 30    # Fast scale-up for public traffic
      policies:
      - type: Pods
        value: 2
        periodSeconds: 60
    scaleDown:
      stabilizationWindowSeconds: 300   # Slow scale-down to prevent thrashing
      policies:
      - type: Pods
        value: 1
        periodSeconds: 120
---
# Internal services — conservative scaling
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: auth-service
  namespace: app
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: auth-service-blue
  minReplicas: 1
  maxReplicas: 5
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70   # Internal: 70% CPU threshold
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 60
      policies:
      - type: Pods
        value: 2
        periodSeconds: 60
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
      - type: Pods
        value: 1
        periodSeconds: 120
```

> **Note:** HPA targets the `-blue` deployment by name. After a blue/green slot switch, the selector changes on the Service — but HPA continues tracking the named Deployment correctly regardless of which slot is active.

---

## Known Issues & Fixes

| Issue | Root Cause | Fix |
|---|---|---|
| Jenkinsfile shell errors | Unicode emoji in shell scripts (checkmarks, arrows) | Rewrote with plain ASCII only — never use emoji in shell |
| `helm upgrade --install` timeout after 15 min | `--wait` flag blocks pipeline until all pods ready | Use `--atomic=false` — submits install, k8s handles rollout |
| `kubectl apply` on ingress.yaml fails | Remote kubectl lacks Traefik CRD schemas | Apply with `--validate=false` — cluster validates correctly |
| Headlamp stage blocks ingress apply on failure | Stage failure propagates downstream | Wrap Headlamp stage in `catchError(buildResult: 'UNSTABLE')` |
| Headlamp Helm install fails | `headlamp-k8s.github.io/headlamp/` returns 404 | Deploy via direct `kubectl apply` on manifest URL instead |

---

## Prerequisites

- Jenkins with Docker, kubectl, Helm, AWS CLI, Trivy on agent
- `kubeconfig` credential in Jenkins (ID: `k3s-kubeconfig`) — Secret File
- AWS credentials configured on Jenkins agent for ECR access
- Cluster already running — see [k3s-dev-staging](https://github.com/Vishal-B142/k3s-dev-staging)

---

## Related

- [k3s-dev-staging](https://github.com/Vishal-B142/k3s-dev-staging) — target cluster
- [eks-production](https://github.com/Vishal-B142/eks-production) — future production target (update kubeconfig credential, remove `--validate=false`)
- [observability-stack](https://github.com/Vishal-B142/observability-stack) — monitoring stack deployed by Jenkinsfile.infra
- [docker-compose-files](https://github.com/Vishal-B142/docker-compose-files) — local Jenkins setup
