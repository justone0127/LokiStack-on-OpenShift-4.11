## LokiStack on OpenShift 4.11

### 1. About the LokiStack

로깅 하위 시스템 설명서에서 **LokiStack**은 Loki가 지원되는 로깅 하위 시스템과 OpenShift Container Platform 인증 통합과 웹 프록시를 나타냅니다. LokiStack의 프록시는 OpenShift Container Platform 인증을 사용하여 멀티 테넌시를 적용합니다. **Loki**는 개별 구성 요소 또는 저장소로 로그 저장소를 나타냅니다.



Loki는 수평적으로 확장 가능하고 가용성이 높은 멀티 테넌트 로그 집계 시스템으로 현재 로깅 하위 시스템의 로그 저장소로 Elasticsearch에 대한 대안으로 제공됩니다.



### 1.1 Deployment Sizing

Loki의 크기 조정은 `N<x>.<size>` 형식을 띠릅니다. 여기서 값 `<N>`은 인스턴스 수이고 `<size>`는 성능 기능을 지정합니다.

- *Table 1. Loki Sizing*

  |                              | 1x.extra-small | 1x.small           | 1x.medium          |
  | ---------------------------- | -------------- | ------------------ | ------------------ |
  | **Data transfer**            | Demo use only. | 500GB/day          | 2TB/day            |
  | **Queries per second (QPS)** | Demo use only. | 25-50 QPS at 200ms | 25-75 QPS at 200ms |
  | **Replication factor**       | None           | 2                  | 3                  |
  | **Total CPU requests**       | 5 vCPUs        | 36 vCPUs           | 54 vCPUs           |
  | **Total Memory requests**    | 7.5Gi          | 63Gi               | 139Gi              |
  | **Total Disk requests**      | 150Gi          | 300Gi              | 450Gi              |



### 1.2 Supported API Custom Resource Definitions

LokiStack 개발이 진행 중이며 현재 모든 API가 지원되는 것은 아닙니다.

| CustomResourceDefinition(CRD) | ApiVersion                         | Support state      |
| ----------------------------- | ---------------------------------- | ------------------ |
| LokiStack                     | lokistack.loki.grafana.com/v1      | Supported in 5.5   |
| RulerConfig                   | rulerconfig.loki.grafana/v1beta1   | Technology Preview |
| AlertingRule                  | alertingrule.loki.grafana/v1beta1  | Technology Preview |
| RecordingRule                 | recordingrule.loki.grafana/v1beta1 | Technology Preview |

> `RulerConfig`, `AlertingRule`, `RecordingRule` 사용자 지정 리소스 정의(CRD) 사용. 기술 프리뷰 기능일 뿐 입니다. 기술 프리뷰 기능은 Red Hat 프로덕션 SLA(서비스 수준 계약)에서 지원되지 않으며 기능적으로 완전하지 않을 수 있습니다. Red Hat은 이를 프로덕션 환경에서 사용하는 것을 권장하지 않습니다. 이러한 기능은 곧 출시될 제품 기능에 대한 조기 엑세스를 제공하여 고객이 개발 프로세스 중에 기능을 테스트하고 피드백을 제공할 수 있도록 합니다.
>
> Red Hat Technology Preview 기능의 지원 범위에 대한 자세한 내용은 다음을 참조하십시오.
>
>  https://access.redhat.com/support/offerings/techpreview/



### 2. Deploying the LokiStack

OpenShift Container Platform 웹 콘솔을 사용하여 LokiStack을 배포할 수 있습니다.

- Prerequisites
  - Red Hat OpenShift Logging Operator 5.5+ 이상 설치 
  - 지원되는 로그 저장소 (AWS S3, Google Cloud Storage, Azure, Swift, Minio, OpenShift Data Foundation)

### 2.1 Creating AWS S3 Bucket 

현재 랩은 AWS 환경에서 테스트 및 진행되고 있습니다.

로그 저장소로 사용하기 위해 AWS 환경에서 S3 버킷을 생성합니다.

```bash
$ aws s3 mb s3://hyou-loki.apps.ocp4.sandbox2710.opentlc.com
```

### 2.2 LokiOperator Install

- 웹 콘솔 접속 > **Operators** > **OperatorHub** 선택 > **LokiOperator** 검색 > **Install**

  ![01_loki_operator](https://github.com/justone0127/LokiStack-on-OpenShift-4.11/blob/main/images/01_loki_operator.png)

- **Installation Mode** 아래에서 **All namespaces on the cluster(클러스터의 모든 네임스페이스-기본)** 선택 > **Installed Namespace** 아래에서 **openshift-operators-redhat** 선택 >  **Approval Strategy (업데이트 승인) **선택 > **Install** 버튼 선택

  ![02_loki_operator_install](https://github.com/justone0127/LokiStack-on-OpenShift-4.11/blob/main/images/02_loki_operator_install.png)

### 2.3  define the object storage location

- `secret` 파일 생성

  ```yaml
  apiVersion: v1
  kind: Secret
  metadata:
    name: logging-loki-s3
    namespace: openshift-logging
  stringData:
    access_key_id: ${ACCESS_KEY_ID}
    access_key_secret: ${ACCESS_KEY_SECRET}
    bucketnames: ${BUCKET_NAME}
    endpoint: ${ENDPOINT}
    region: ${REGION}
    config: |
      {
          "insecure": true  // 자체 서명 인증서 혹은 인증서가 없는 경우 설정 필요!!
      }
  ```

- `LokiStack` custom resource 생성

  ```yaml
  cat << EOF > logging-loki.yaml
    apiVersion: loki.grafana.com/v1
    kind: LokiStack
    metadata:
      name: logging-loki
      namespace: openshift-logging
    spec:
      size: 1x.small
      storage:
        schemas:
        - version: v12
          effectiveDate: '2022-12-26'
        secret:
          name: logging-loki-s3
          type: s3
      storageClassName: gp2
      tenants:
        mode: openshift-logging
  EOF
  ```

- 적용

  ```bash
  $ oc apply -f logging-loki.yaml
  ```

- `ClusterLogging` CR 생성 또는 수정

  ```yaml
  cat << EOF > cr-lokistack.yaml
    apiVersion: logging.openshift.io/v1
    kind: ClusterLogging
    metadata:
      name: instance
      namespace: openshift-logging
    spec:
      managementState: Managed
      logStore:
        type: lokistack
        lokistack:
          name: logging-loki
        collection:
          type: "vector"
  EOF
  ```

- 적용

  ```bash
  $ oc apply -f cr-lokistack.yaml
  ```

-  Enable the RedHat OpenShift Logging Console Plugin 

  - 웹 콘솔 접속 > **Operators** > **Installed Operators** 선택

  - **RedHat OpenShift Logging** Operator 선택

  - 콘솔 플러그인에서 **Disabled** 선택

    ![03_logging_console_plugin_disable](https://github.com/justone0127/LokiStack-on-OpenShift-4.11/blob/main/images/03_logging_console_plugin_disable.png)

  - **Enable** 선택 후 저장 > `openshift-conole` pod가 재시작 되고 새로 반영 됩니다.

    ![04_logging_console_plugin_enable](https://github.com/justone0127/LokiStack-on-OpenShift-4.11/blob/main/images/04_logging_console_plugin_enable.png)

  - `openshift-console` pod 재 시작 확인

    ![05_openshift-console_pod](https://github.com/justone0127/LokiStack-on-OpenShift-4.11/blob/main/images/05_openshift-console_pod.png)

  - 웹 콘솔을 새로고침한 후 왼쪽 메인 메뉴에서 **Observe**를 클릭합니다. **Logs**에 대한 새로운 옵션을 사용 할 수 있습니다.

    ![06_new_logs_options](https://github.com/justone0127/LokiStack-on-OpenShift-4.11/blob/main/images/06_new_logs_options_new.png)
