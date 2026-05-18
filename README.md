# my-market-msa-manifest

Troica Market Service MSA의 **공통 Helm 차트**. 모든 마이크로서비스가 이 차트를 공유하고, 서비스별 설정은 [my-msa-manifest-values](https://github.com/kjylab/my-msa-manifest-values)에서 주입한다.

## 개념

### Helm 차트란?
Kubernetes 리소스(Deployment, Service, ConfigMap 등) 템플릿의 묶음. `{{ .Values.xxx }}` 문법으로 값을 외부에서 주입받는다.

### 왜 공통 차트를 쓰나?
6개 마이크로서비스가 구조적으로 동일하기 때문에 (Deployment + Service + ConfigMap 패턴) 차트 하나로 모두 처리하고, 차이점만 values로 분리한다.

```
공통 차트 (이 레포)          서비스별 values (별도 레포)
    ↓                              ↓
  템플릿          +          실제 값 (이미지 태그, DB URL 등)
    └──────────── helm install ────────────→ K8s 리소스 생성
```

## 템플릿 구성

| 파일 | 생성되는 K8s 리소스 |
|------|-------------------|
| `templates/deployment.yaml` | Deployment — 파드 실행 정의 |
| `templates/service.yaml` | Service — 클러스터 내부 통신 (HTTP:8080, gRPC:9090) |
| `templates/configmap.yaml` | ConfigMap — Spring Boot `application.yaml` 설정 주입 |
| `templates/ingress.yaml` | Ingress — 외부 트래픽 라우팅 (선택) |
| `templates/servicemonitor.yaml` | ServiceMonitor — Prometheus 메트릭 수집 대상 등록 |

## values 구조

```yaml
image:
  repository: jyupk/my-msa-product-service
  tag: abc123def456          # CI가 커밋 SHA로 자동 업데이트

service:
  type: ClusterIP
  port: 9090                 # gRPC 포트

serviceMonitor:
  enabled: true              # Prometheus 스크래핑 활성화

appConfig:                   # → ConfigMap으로 마운트 → Spring Boot 설정
  spring:
    datasource:
      url: jdbc:postgresql://...
  grpc:
    server:
      port: 9090
  management:
    endpoints:
      web:
        exposure:
          include: health,prometheus
```

## ServiceMonitor 동작 원리

```
ServiceMonitor (monitoring 네임스페이스)
  └─ "default 네임스페이스의 app=product-service 서비스를
      http 포트의 /actuator/prometheus 를 15초마다 스크래핑해라"
         ↓
Prometheus Operator가 감지
         ↓
Prometheus 설정 자동 업데이트
         ↓
Grafana에서 메트릭 조회 가능
```

## 관련 레포

| 레포 | 역할 |
|------|------|
| [my-msa-manifest-values](https://github.com/kjylab/my-msa-manifest-values) | 서비스별 실제 values 파일 |
| [ktcloud-k8s-argocd-manifest](https://github.com/kjylab/ktcloud-k8s-argocd-manifest) | ArgoCD App of Apps (이 차트를 사용하는 주체) |
