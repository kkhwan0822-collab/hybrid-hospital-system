# 중소형 병원 하이브리드 클라우드 보안 아키텍처 (Wazuh 기반 SIEM)

- **기간**: 2026.4 ~ 2026.6 (2개월) / **형태**: 팀 프로젝트(4인)
- **담당**: Wazuh 기반 SIEM 설계·구축, 장애 자동복구 아키텍처
- 온프레미스(민감 데이터) + AWS(운영) + GCP(DR) 3중 구조를 Wazuh로 통합 관제

![Wazuh 아키텍처](./images/architecture.png)

## 왜 Wazuh인가

- 서버의 로그와 행위를 실시간으로 감시해 보안 위협을 탐지하는 오픈소스 SIEM. 우리 환경(온프레미스+AWS+GCP 3중 구조)에 맞게 탐지 룰을 처음부터 직접 설계할 수 있음.
- 2023년 세브란스병원 내부자 유출 사건(약 5만 명 환자기록)을 참고해, 가장 큰 위협은 외부 해킹이 아니라 내부자의 비정상적 데이터 접근이라고 판단 → 탐지 룰 설계 우선순위를 내부자 행위 쪽에 뒀다.
- 목표는 사전 차단이 아니라 빠른 탐지로 잡음.

## 왜 Ansible인가

- Wazuh는 설치 순서와 설정이 복잡한 공식 권장 배포 방식이라, 순서가 하나라도 틀리면 설치 자체가 불가능했다.
- 태그 기반 플레이북으로 룰/에이전트/모니터링을 구간별로 분리 → 특정 단계에서 오류가 나도 그 부분만 재실행 가능 (초기 개발 단계에서 특히 유용).
- 서버 추가·룰 일괄 배포가 필요할 때도 플레이북 한 번으로 전체 서버에 동일 적용 가능해 확장성 확보.

## 핵심 설계 포인트

- **Terraform 인프라 코드화**: CloudTrail, GuardDuty, VPC Flow Log, WAF, VPN, S3, Lambda까지 보안 인프라 전체를 코드로 관리. IAM은 리소스/액션 단위 최소 권한(예: KMS Decrypt를 `kms:ViaService` 조건으로 S3 경유만 허용). GitHub-Terraform Cloud VCS 연동으로 푸시 시 Plan 자동 실행, Apply만 수동 승인하는 배포 파이프라인 구성.
- **Bedrock 일일 보안 보고서**: `EventBridge(08:30) → Lambda → S3 로그+취약점 데이터 → Bedrock(Claude Haiku) → Slack`. 로그 외부 유출을 막기 위해 AWS 내부망만 사용, Bedrock이 데이터를 모델 학습에 쓰지 않는다는 점을 확인 후 채택. 보고서 형식이 정형화돼 있어 저비용 Haiku로 충분하다고 판단.
- **Prometheus/Grafana 통합 모니터링**: Wazuh(보안 이벤트)와 별개로 인프라 전반 상태를 볼 단일 EC2 구성. Alloy를 각 서버(AWS/GCP/온프레미스)에 배치해 메트릭을 Push로 수집, 데이터는 별도 EBS에 분리 저장해 인스턴스 재생성과 무관하게 보존. 프로세스 다운과 EC2 자체 장애를 구분해 다른 CloudWatch 알람 경로로 대응.
- **장애 자동복구**: Wazuh Manager·Indexer·모니터링 서버 각각 골든 AMI + Lambda 기반 자동복구 구성. EventBridge가 EC2 중지/종료를 즉시 감지 → Lambda가 최신 AMI로 재생성, 데이터는 별도 EBS로 분리해 인스턴스 재구축돼도 유지.
- **ISMS-P 로그 보존 체계**: 접근기록 1년 이상 보존, 위변조 방지(S3 버저닝 + KMS 암호화) 등 컴플라이언스 요구사항을 아키텍처에 반영.

## 탐지 검증 — 랜섬웨어 시뮬레이션

`드로퍼 배포 → 권한 상승 → 파일 암호화 → 확장자 변경 → 원본 삭제` 전 과정을 재현해 검증.

| 단계 | 결과 |
|---|---|
| 드로퍼 생성 / sudo root 전환 | level 14 / 8 → Slack |
| 대량 암호화·확장자 변경·삭제 | level 15 → Slack |

## 트러블슈팅

**1) Wazuh 이중화 → 단일 매니저 + Lambda 복구**
Master-Worker 클러스터로 짰으나 대시보드가 Master API에만 종속돼 이중화 무의미. Worker 제거 후 골든 AMI + Lambda 자동복구로 전환. **RTO 목표 10분 → 실측 3분 10초.** (ASG 미사용 이유: Agent-Manager 인증서 공유 구조상 새 인스턴스를 기존 Agent가 인식 못함)

**2) Wodle 중복 수집 + 감지 지연**
CloudWatch 기본 평가(60초×5회)로 감지에만 5분 이상 소요. Manager Cron으로 30초마다 메트릭 전송 + period 30초로 단축 → **감지 5분→60초.**

**3) Indexer 단일 장애점**
3대 클러스터는 비용상 불가 → 단일 Indexer + DLM 1시간마다 스냅샷(24개 보관) + Lambda 자동복구 이중 안전장치.

**4) 모니터링 Pull→Push 전환**
node_exporter Pull 방식은 서버 다운 중 메트릭 유실. Alloy Push로 전환(8시간 버퍼링) → 대신 Prometheus 서버는 고정 IP 필요해 ASG 대신 CloudWatch Alarm(Reboot/Recover)으로 가용성 확보.

**5) Lambda 복구 시 EBS 충돌 + KMS 권한**
골든 AMI가 데이터 볼륨 붙은 채로 찍혀 복구 시 충돌 → 빈 볼륨 상태로 AMI 재생성. KMS 암호화 볼륨이라 Lambda 역할에 권한(`kms:Encrypt/Decrypt/ReEncrypt*/GenerateDataKey*/CreateGrant`) 5종 추가.

## 로그 소스별 수집 구조

12개 소스 각각 실시간성 요구 수준과 AWS 서비스별 제약이 달라, 같은 방식으로 묶지 않고 소스별로 경로를 다르게 설계했다.

| 소스 | 수집 경로 | 비고 |
|---|---|---|
| 온프레미스 (auth.log, DB/EMR/Vault 감사, FIM) | Agent → Manager (직접) | Cron으로 S3 원본 백업(KST 02:05) |
| ECS (FastAPI 감사) | Agent(journald) → Manager | Vector로 S3 별도 보존 |
| CloudTrail | CloudWatch Logs → Metric Filter → Alarm → SNS → Slack | 원본은 S3 보존 |
| GuardDuty | S3 → Wodle (대시보드용) / EventBridge → Lambda → Slack (실시간) | 중복 알림 방지를 위해 Slack 경로 이원화 |
| VPC Flow Logs | REJECT만 S3 → Wodle | ACCEPT는 노이즈로 제외 |
| **WAF** | S3 → Wodle | Firehose 경유 시 순수 JSON이라 Wodle 파싱 가능 |
| **VPN** | CloudWatch → Wodle (직접) | CloudWatch 래퍼 포맷이라 S3 경유 시 Wodle 파싱 불가 → 터널 DOWN은 Lambda로 실시간 알림 병행 |
| RDS (pgaudit) | CloudWatch Logs → Firehose → S3 → Wodle | 정상 트래픽(헬스체크·DR 복제 등) 선(先) 억제 후 이상행위만 탐지 |
| **ALB** | 전용 SSE-S3 버킷 → Wodle | KMS CMK 버킷은 ALB 서비스 계정 권한 부재로 저장 불가 |
| Cloud SQL (GCP DR) | HAProxy GCE(Agent) 경유 → Manager | Wazuh가 GCP Cloud Logging 직접 수집 모듈이 없어 우회 |
| 모니터링 서버 (Grafana/Keycloak/OAuth2-Proxy) | Agent → Manager | SSO 통합으로 로그인 1회로 3개 대시보드 접근 |


```
[레포지토리 구조]
hospital-hybrid-infrastructure/
├── .github/
│   └── workflows/          # GitHub Actions (CI/CD) 워크플로우 정의
├── terraform/              # 인프라 코드 (IaC)
│   ├── modules/            # 공통 재사용 모듈 (VPC, SG, IAM 등)
│   ├── aws/                # AWS 인프라 (EKS, RDS, S3, WAF, Wazuh 등)
│   ├── gcp/                # GCP 인프라 (Cold Standby DR, Cloud Storage 등)
│   └── onprem/             # On-Premise (VMware, EMR 서버, Wazuh Manager 등)
├── app/                    # 병원 웹 사이트 소스 코드 (Dockerize 대상)
│   ├── dashboard/          # 운영 통합관리 대시보드
│   └── portal/             # 사용자 전용 웹 사이트
├── docs/                   # 산출물 (ISMS-P 정책서, 아키텍처 다이어그램 등)
├── scripts/                # 공통 유틸리티 스크립트 (NTP 동기화, 로그 수집 등)
└── README.md               # 프로젝트 개요 및 구축 가이드
```

