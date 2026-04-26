# AWS VPC Block Public Access (VPC BPA) 통합 정리

> 운영 중인 AWS 환경에 VPC BPA 도입을 검토하는 클라우드 엔지니어를 위한 핵심 정리본
> 출처: AWS 공식 문서 (security-vpc-bpa, basics, assess-impact-main, example)

---

## 1. VPC BPA란 무엇인가

**VPC Block Public Access (BPA)** 는 AWS 계정 단위에서 **인터넷 게이트웨이(IGW) 및 송신 전용 인터넷 게이트웨이(EIGW)를 통한 퍼블릭 인터넷 트래픽을 일괄 차단**할 수 있는 VPC 보안 기능이다.

- **적용 단위**: 리전(Region) 단위 — 계정 + 리전 조합으로 ON/OFF
- **제어 위치**: VPC 콘솔 → 설정 → "퍼블릭 액세스 차단"
- **핵심 가치**: 실수로 만들어진 퍼블릭 서브넷, 잊혀진 EIP, 위험한 라우트 테이블 등으로 인한 **의도하지 않은 인터넷 노출을 한 스위치로 차단**

---

## 2. 도입 효과

| 효과 | 설명 |
|------|------|
| **광범위한 노출 차단** | 실수/오설정으로 인한 퍼블릭 노출을 계정 단위에서 즉시 차단 |
| **중앙 집중식 가드레일** | 개별 SG/NACL/라우트 테이블 점검 없이 일괄 정책 적용 |
| **유연한 예외 처리** | VPC/서브넷 단위 Exclusion으로 필요한 워크로드만 인터넷 허용 |
| **조직 단위 강제** | AWS Organizations 선언적 정책으로 전체 계정에 일괄 적용 가능 |
| **감사/컴플라이언스** | CloudTrail로 변경 추적, Flow Logs로 차단 트래픽 가시화 |

---

## 3. 작동 모드 (Block Mode)

VPC BPA는 두 가지 차단 모드를 제공한다.

### 3.1 양방향 차단 (block-bidirectional)
- IGW/EIGW를 통한 **인바운드 + 아웃바운드 모두 차단**
- 가장 강력한 모드 — 인터넷 통신 자체를 막음

```bash
aws ec2 modify-vpc-block-public-access-options \
  --internet-gateway-block-mode block-bidirectional \
  --region ap-northeast-2
```

### 3.2 수신 전용 차단 (block-ingress)
- **인바운드만 차단**, 아웃바운드는 허용
- NAT GW / EIGW를 통한 외부 호출(패치, API 호출 등)은 그대로 동작
- ⚠️ **로컬 영역(Local Zones)에서는 미지원**

```bash
aws ec2 modify-vpc-block-public-access-options \
  --internet-gateway-block-mode block-ingress \
  --region ap-northeast-2
```

### 3.3 비활성화 (off)
```bash
aws ec2 modify-vpc-block-public-access-options \
  --internet-gateway-block-mode off
```

> 활성화 후 상태가 `enabling` → `enabled`로 전환되는데 **수 분 소요**

---

### 3.4 BPA의 Stateful 동작 (중요)

BPA는 **Stateful**하게 동작한다 — Security Group과 유사한 모델. 단, **Stateful 처리가 적용되는 경로가 정해져 있다**:

| 시나리오 | Stateful 처리 대상 outbound |
|----------|----------------------------|
| `block-ingress` 모드 | **NAT GW / EIGW** 경유 outbound (IGW 직접 경유는 제외) |
| `block-bidirectional` 모드 + `allow-egress` exclusion | exclusion이 적용된 VPC/서브넷의 IGW outbound |

위 두 경우의 outbound connection은 return 트래픽이 자동 허용된다.

- 출처: AWS 공식 블로그 — *"Amazon VPC Block Public Access is stateful when used in ingress-only mode, or when allowing an egress-only exclusion. Return traffic for an allowed connection is automatically permitted. This behavior is analogous to security groups."*

#### Ingress-only 모드에서의 outbound 동작 (중요한 비대칭)

| outbound 경로 | 동작 |
|--------------|------|
| **NAT Gateway 경유** | ✅ 기본 허용 (Stateful 추적 → return 자동 허용) |
| **EIGW 경유** (IPv6) | ✅ 기본 허용 (Stateful 추적 → return 자동 허용) |
| **Public Subnet EC2가 IGW 직접 경유** (예: 외부 proxy, NAT 인스턴스) | ⚠️ 사실상 차단 |

마지막 케이스가 함정이다:
- Ingress-only 모드는 IGW **인바운드만** 차단하므로 outbound 자체는 IGW로 나갈 수 있음
- 그러나 IGW 직접 outbound는 **Stateful 추적 대상이 아님** → return 트래픽이 인바운드 차단에 걸려 끊김
- 결과: TCP 핸드셰이크 실패 / 응답 미수신 → 실질적 통신 불가

**해결책**: 해당 서브넷에 **`allow-bidirectional` exclusion**을 적용해야 한다.
- ⚠️ `allow-egress` exclusion은 `block-bidirectional` 모드에서만 의미가 있음 — Ingress-only 모드에서는 풀어줄 outbound 자체가 막혀 있지 않으므로 무의미하며, 정작 막혀 있는 인바운드(return)는 풀리지 않는다.

#### ⚠️ Hairpin 함정 — VPC 내부에서 Public IP로 접근하는 경우
VPC 내부 리소스가 같은 VPC의 ALB·EC2를 **Public IP**로 접근하면, 라우팅이 `0.0.0.0/0 → IGW`를 따라가기 때문에 IGW를 한 번 빠져나갔다 다시 들어오는 형태(hairpin)가 된다.
- Ingress-only 모드: 돌아오는 트래픽이 IGW 인바운드로 처리되어 **차단됨**
- 회피책: 같은 VPC 내부 통신은 **Private IP / 프라이빗 DNS**로 호출 (IGW 미경유라 BPA 영향 없음)

---

## 4. 제외 항목 (Exclusions)

특정 VPC 또는 서브넷을 BPA 정책에서 예외로 두는 기능. 보안 가드레일을 켠 채로 **꼭 필요한 영역만 인터넷을 열어준다.**

### 4.1 제외 모드

| 모드 | 동작 | 적용 조건 |
|------|------|-----------|
| `allow-bidirectional` | 양방향 인터넷 트래픽 허용 | 항상 사용 가능 |
| `allow-egress` | 아웃바운드만 허용 (인바운드 차단) | 계정 BPA가 `block-bidirectional`일 때만 의미 있음 |

### 4.2 적용 단위
- **VPC 단위 제외** → 해당 VPC 내 모든 서브넷에 자동 적용
- **서브넷 단위 제외** → 해당 서브넷에만 적용
- **기본 한도**: 리전당 **최대 50개** (Service Quotas로 증액 요청 가능)
- BPA가 비활성화 상태에서도 **사전에 Exclusion을 미리 만들어 둘 수 있음**

### 4.3 주요 명령어

```bash
# 서브넷 제외 (양방향 허용)
aws ec2 create-vpc-block-public-access-exclusion \
  --subnet-id subnet-xxx \
  --internet-gateway-exclusion-mode allow-bidirectional

# VPC 제외
aws ec2 create-vpc-block-public-access-exclusion \
  --vpc-id vpc-xxx \
  --internet-gateway-exclusion-mode allow-bidirectional

# 모드 변경
aws ec2 modify-vpc-block-public-access-exclusion \
  --exclusion-id vpcbpa-exclusion-xxx \
  --internet-gateway-exclusion-mode allow-egress

# 조회 / 삭제
aws ec2 describe-vpc-block-public-access-exclusions
aws ec2 delete-vpc-block-public-access-exclusion --exclusion-id vpcbpa-exclusion-xxx
```

---

## 5. 영향을 받는 / 받지 않는 리소스

### 5.1 영향을 받는 리소스 (BPA가 차단함)

| 리소스 | 차단 방향 | 비고 |
|--------|-----------|------|
| Internet Gateway (IGW) | 양방향 | 직접 대상 |
| Egress-Only IGW (EIGW) | 아웃바운드 | IPv6 |
| NAT Gateway | 양방향 (IGW 의존) | 인터넷 NAT은 IGW 통해 동작하므로 차단됨 |
| Internet-facing ALB / NLB | 양방향 | IGW 의존 |
| CloudFront VPC Origin | 양방향 | |
| Direct Connect Public VIF | 양방향 | 퍼블릭 IPv4/IPv6 트래픽 |
| AWS Global Accelerator | 인바운드 | |
| AWS Network Firewall | 양방향 | **Exclusion 영역도 차단됨** |
| Gateway Load Balancer (GWLB) | 양방향 | **Exclusion 영역도 차단됨** |
| AWS Wavelength Carrier Gateway | 양방향 | |

### 5.2 영향을 받지 않는 리소스 (프라이빗 연결)

IGW를 사용하지 않는 프라이빗 경로는 그대로 동작한다.

- AWS Site-to-Site VPN
- AWS Client VPN
- AWS Transit Gateway
- AWS CloudWAN
- AWS Outposts Local Gateway
- AWS Verified Access
- VPC Peering, VPC Endpoint(PrivateLink)
- Route 53 Resolver, VPC 내부 AWS 서비스 트래픽

---

## 6. 제약 사항 및 주의점

### 6.1 기능적 제약
- **로컬 영역(Local Zones)**: `block-ingress` 모드 미지원
- **Exclusion 한도**: 리전당 50개 (증액 가능)
- **Network Firewall / GWLB**: Exclusion으로도 우회 불가 — 양방향 모드에서 차단됨
- **IPv6 영향 평가 한계**: Network Access Analyzer는 EIGW의 IPv6 트래픽 영향 평가를 일부 지원하지 않음

### 6.2 가시성/모니터링 제약
- **VPC Flow Logs**: BPA로 거부된 레코드는 `reject-reason=BPA`로 표시되지만,
  - **건너뛴 레코드(skipped) 미포함**
  - **bytes 필드는 BPA Flow Logs에 기록되지 않음**

### 6.3 주의해야 할 트래픽
- **VPC 내부 트래픽 / Route 53 Resolver**: IGW를 거치지 않으므로 BPA로 차단되지 않음 → **SG/NACL 등 별도 통제 필요**
- **써드파티 보안 어플라이언스(EC2 기반 방화벽 등)**: 해당 서브넷을 반드시 Exclusion으로 빼야 함

### 6.4 VPC 공유(RAM) 시나리오
- **서브넷 소유 계정**: BPA 설정/해제 권한 보유
- **참가자(공유 사용) 계정**: 소유자 설정의 영향을 받지만 직접 제어 불가

### 6.5 AWS Organizations 선언적 정책
- 조직 차원에서 모드(양방향/수신전용)와 "Exclusion 허용 여부"를 강제 가능
- **단, 개별 Exclusion 생성은 각 계정에서 수행해야 함** (정책에서 만들지 않음)

---

## 7. 관련 IAM 권한

| 권한 | 용도 |
|------|------|
| `ec2:ModifyVpcBlockPublicAccessOptions` | BPA 모드 변경 |
| `ec2:DescribeVpcBlockPublicAccessOptions` | BPA 상태 조회 |
| `ec2:CreateVpcBlockPublicAccessExclusion` | Exclusion 생성 |
| `ec2:ModifyVpcBlockPublicAccessExclusion` | Exclusion 모드 변경 |
| `ec2:DeleteVpcBlockPublicAccessExclusion` | Exclusion 삭제 |
| `ec2:DescribeVpcBlockPublicAccessExclusions` | Exclusion 조회 |

---

## 8. 도입 전 고려사항 (가장 중요)

운영 환경에 도입할 때는 **켜기 전 영향도를 반드시 평가**해야 한다.

### 8.1 사전 영향 평가 — Network Access Analyzer
**현재 인터넷 게이트웨이를 사용하는 리소스를 식별**하여 BPA 활성화 시 어떤 트래픽이 영향을 받을지 미리 파악.

```
콘솔: Network Insights → Network Access Analyzer
     → "VPC 퍼블릭 액세스 차단의 영향 평가" 템플릿 사용
```

### 8.2 경로별 검증 — Reachability Analyzer
특정 IGW → 특정 인스턴스 경로를 BPA 활성화 후 시뮬레이션.
결과의 `ExplanationCode = VPC_BLOCK_PUBLIC_ACCESS_ENABLED`이면 BPA로 인한 차단.

### 8.3 운영 모니터링 — VPC Flow Logs
- **사용자 지정 형식**으로 Flow Logs를 만들고 **`reject-reason` 필드를 반드시 포함**
- `reject-reason=BPA` 레코드를 모니터링하면 BPA로 인한 차단을 실시간 추적 가능

### 8.4 변경 감사 — CloudTrail
- 리소스 타입 `AWS::EC2::VPCBlockPublicAccessExclusion` 으로 필터
- Exclusion 생성/수정/삭제 이벤트를 추적 → Exclusion이 임의로 늘어나지 않도록 감사

### 8.5 도입 체크리스트

- [ ] **인벤토리 파악**: 현재 IGW를 직접 사용하는 워크로드(웹서버, 외부 API 호출 인스턴스 등) 목록 작성
- [ ] **Network Access Analyzer 실행**: 영향받을 VPC/서브넷 식별
- [ ] **NAT GW 의존 워크로드 확인**: `block-bidirectional` 적용 시 NAT 자체가 차단되므로, 아웃바운드가 필요한 경우 `block-ingress` 또는 Exclusion 검토
- [ ] **Exclusion 설계**: 인터넷 노출이 필수인 서브넷/VPC 목록 작성 (양방향 vs egress-only 모드 결정)
- [ ] **Direct Connect Public VIF 사용 여부 점검**: Public VIF 트래픽은 퍼블릭 IP를 거치므로 BPA로 차단됨
  - Private VIF / Transit VIF, VPN, TGW는 IGW를 거치지 않아 영향 없음 — 점검 불필요
- [ ] **3rd-party 어플라이언스 식별**: 인라인 방화벽/IDS 등이 있는 서브넷은 사전에 Exclusion 등록
- [ ] **Network Firewall / GWLB 확인**: Exclusion으로 우회되지 않으므로 BPA 모드 결정 시 신중히
- [ ] **Flow Logs 사용자 지정 포맷에 `reject-reason` 추가**
- [ ] **CloudTrail 모니터링 설정**: Exclusion 변경 알림
- [ ] **단계적 롤아웃**: Dev/Stage 리전 → Prod 리전 순서, 또는 일부 리전에서 `block-ingress`부터 시작
- [ ] **롤백 절차 준비**: `--internet-gateway-block-mode off` 명령으로 즉시 해제 가능함을 운영팀에 공유
- [ ] **AWS Organizations 정책 검토**: 조직 차원 강제 적용이 필요하면 선언적 정책으로 확장

---

## 9. 권장 도입 패턴

### 패턴 A — "수신만 차단" (가장 안전한 시작점)
- 모드: `block-ingress`
- 효과: 외부에서 들어오는 트래픽만 차단, 아웃바운드(패치/외부 API)는 그대로 동작
- 적합 환경: 대부분의 백오피스/내부 워크로드, NAT 경유 아웃바운드가 많은 환경

### 패턴 B — "기본 양방향 차단 + 핀홀(Exclusion)"
- 모드: `block-bidirectional` + 웹/공개 서비스 서브넷만 `allow-bidirectional` Exclusion
- 효과: 강력한 가드레일 + 명시적으로 허용된 영역만 인터넷 통신
- 적합 환경: 인터넷 노출이 명확하게 정의된 멀티티어 아키텍처(웹 서브넷만 노출, 그 외 차단)

### 패턴 C — "관리 전용 VPC"
- 모드: `block-bidirectional` (Exclusion 없음)
- 외부 접근은 Session Manager / VPN / TGW 통해서만
- 적합 환경: 관리 계정, 보안 도구 계정, 데이터 격리 VPC

### 사용 케이스별 정책 매핑

세 가지 대표 퍼블릭 액세스 패턴(① Public ALB 인바운드 / ② EIP 인스턴스 인바운드(SSL VPN 등) / ③ NAT GW 아웃바운드)을 **모두 만족**시키려면 다음 두 방식 중 선택:

| 방식 | 계정 BPA 모드 | 필요한 Exclusion | 장점 |
|------|--------------|------------------|------|
| **방식 1 (권장)** | `block-ingress` | 없음 (NAT/EIGW outbound는 기본 허용, ALB·EIP 인스턴스는 인바운드 차단되므로 해당 서브넷에 `allow-bidirectional` exclusion) | 운영 단순, exclusion 최소화 |
| **방식 2** | `block-bidirectional` | • Public ALB 서브넷: `allow-bidirectional`<br>• EIP 인스턴스(SSL VPN) 서브넷: `allow-bidirectional`<br>• NAT GW 서브넷: `allow-egress` | 명시적 화이트리스트, 가드레일 강함 |

**방식 1 상세**:
- 기본 `block-ingress` 만으로 NAT GW outbound(Case 3) 자동 허용
- Public ALB(Case 1) / EIP 인스턴스(Case 2)가 있는 서브넷에는 `allow-bidirectional` exclusion 추가 → 외부 인바운드 세션 허용

**방식 2 상세**:
- 모든 인터넷 트래픽이 기본 차단 → 명시적으로 허용한 서브넷만 통신
- NAT GW는 outbound 전용 서비스이므로 `allow-egress` 모드로 충분
- ⚠️ NAT GW가 위치한 서브넷 자체에 exclusion을 적용해야 함 (다른 서브넷에 적용하면 NAT GW 트래픽에 영향 없음)
- ⚠️ `allow-egress`는 계정 BPA가 `block-bidirectional`일 때만 의미 있음

---

## 10. 빠른 참조 — 핵심 명령어 모음

```bash
# 상태 확인
aws ec2 describe-vpc-block-public-access-options --region <region>

# 활성화 (수신만 차단)
aws ec2 modify-vpc-block-public-access-options \
  --internet-gateway-block-mode block-ingress

# 활성화 (양방향 차단)
aws ec2 modify-vpc-block-public-access-options \
  --internet-gateway-block-mode block-bidirectional

# 비활성화
aws ec2 modify-vpc-block-public-access-options \
  --internet-gateway-block-mode off

# 서브넷 Exclusion 추가
aws ec2 create-vpc-block-public-access-exclusion \
  --subnet-id subnet-xxx \
  --internet-gateway-exclusion-mode allow-bidirectional

# Exclusion 목록
aws ec2 describe-vpc-block-public-access-exclusions

# CloudTrail에서 Exclusion 변경 추적
aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=ResourceType,AttributeValue=AWS::EC2::VPCBlockPublicAccessExclusion
```

---

## 11. 한 장 요약

| 항목 | 내용 |
|------|------|
| **무엇** | 계정+리전 단위로 IGW/EIGW 트래픽을 차단하는 보안 가드레일 |
| **모드** | `block-bidirectional` (양방향) / `block-ingress` (수신만) / `off` |
| **예외** | VPC 또는 서브넷 단위 Exclusion (`allow-bidirectional` / `allow-egress`), 리전당 50개 |
| **차단 대상** | IGW·EIGW·NAT GW·인터넷 ALB/NLB·DX Public VIF·CloudFront VPC Origin 등 |
| **영향 없음** | VPN·TGW·Client VPN·VPC Peering·PrivateLink·Outposts LGW |
| **사전 평가 도구** | Network Access Analyzer, Reachability Analyzer, VPC Flow Logs (`reject-reason=BPA`), CloudTrail |
| **주요 제약** | Local Zones는 `block-ingress` 미지원, Network Firewall/GWLB는 Exclusion 우회 불가, Flow Logs `bytes` 필드 미기록 |
| **권장 시작** | Dev 리전 → `block-ingress` → 영향 모니터링 → Prod 확장 |
