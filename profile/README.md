<div align="center">

<p><strong>K-Shield Junior 17기 · 침해사고 대응 및 분석 · 4조</strong></p>

# 🛡️ 공급망 공격 기반 랜섬웨어 침해사고 분석 및 대응


<p>
  <img src="https://img.shields.io/badge/STATUS-IN%20PROGRESS-2563EB?style=for-the-badge" alt="Status: In Progress" />
  <img src="https://img.shields.io/badge/PERIOD-2026.09.01--09.17-0F172A?style=for-the-badge" alt="Period: 2026.09.01–09.17" />
  <img src="https://img.shields.io/badge/TEAM-K--SHIELD%204-7C3AED?style=for-the-badge" alt="Team: K-Shield 4" />
  <img src="https://img.shields.io/badge/LAB-ISOLATED-059669?style=for-the-badge" alt="Lab: Isolated" />
</p>

<p>
  <a href="#overview">Overview</a> ·
  <a href="#scenario">Attack Scenario</a> ·
  <a href="#analysis">DFIR</a> ·
  <a href="#deliverables">Deliverables</a> ·
  <a href="#team">Team</a>
</p>

</div>

---

<a id="overview"></a>

## 프로젝트 개요

> **공격 재현 → 증거 수집 → 원인 분석 → 탐지 → 대응**  
> 공급망 침해사고의 전 과정을 하나의 타임라인과 검증 가능한 증거로 연결하는 실전형 DFIR 프로젝트입니다.

| 구분 | 내용 |
| --- | --- |
| **프로젝트** | 공급망 공격 기반 랜섬웨어 침해사고 분석 및 대응 |
| **기간** | 2026.09.01 ~ 2026.09.17 |
| **핵심 시나리오** | 취약한 SW 공급사를 거쳐 정상 B2B 전송망으로 유입된 랜섬웨어가 병원 내부망까지 확산 |
| **분석 범위** | 유포 경로 추적, 로그·패킷·시스템 아티팩트 분석, 랜섬웨어 행위 분석, IoC 도출 |
| **대응 범위** | YARA·Sigma 탐지 규칙, Wazuh Alert, 초동 대응 SOP, 재발 방지 가이드라인 |
| **실습 원칙** | 외부 네트워크와 완전히 격리된 가상 랩에서만 수행 |

### 프로젝트 목표

| 01 · Scenario | 02 · DFIR | 03 · Detection & Response |
| --- | --- | --- |
| 공급망 기반 초기 침투부터 내부망 확산·암호화까지 가상 환경으로 구현 | 로그·패킷·파일·시스템 아티팩트를 교차 분석해 침투 경로와 피해 범위 규명 | IoC와 ATT&CK 매핑을 기반으로 탐지 규칙 및 단계별 초동 대응 체계 수립 |

---

## Ganda-it Transfer

**Ganda-it Transfer**는 공급업체와 병원 사이에서 업무 파일과 소프트웨어 패키지를 전달하는 **가상의 B2B 파일 전송 서비스**입니다. 정상 환경에서는 공급업체가 등록한 파일을 병원 수신 서버가 전달받아 내부 스케줄러로 처리합니다.

> [!IMPORTANT]
> 실제 MOVEit Transfer 침해사고와 CVE-2023-34362 공개 자료는 시나리오 설계의 참고 사례로만 활용합니다. 실습 환경은 특정 상용 제품을 복제하지 않은 자체 가상 서비스이며, 실제 기업·제품·서비스와 관련이 없습니다.

---

<a id="scenario"></a>

## 공격 시나리오

<!--
이미지가 준비되면 아래 주석을 해제하고 경로를 실제 파일에 맞게 수정하세요.

<p align="center">
  <img src="./docs/images/attack-flow.png" alt="공급망 공격 시나리오 및 Wazuh 모니터링 구성도" width="900" />
</p>
<p align="center"><sub>그림 1. 공급망 공격 시나리오 및 Wazuh 모니터링 구성도</sub></p>
-->

| Phase | 공격 단계 | 주요 행위 | 주요 증거 | MITRE ATT&CK |
| :---: | --- | --- | --- | --- |
| **01** | 최초 침투 | SQL Injection 악용 후 ASP.NET 웹 셸을 통해 공급업체 서버 거점 확보 | IIS 로그, 웹 루트 파일, 파일 생성 시각 | `T1190` `T1505.003` |
| **02** | 자격 증명 탈취·정찰 | DB·세션·설정 파일에서 B2B 계정, API 토큰 및 연동 스케줄 탐색 | DB·세션 기록, 설정 파일, 인증 로그 | `T1003` `T1552.001` |
| **03** | 공급망 전파·실행 | 탈취한 정상 계정으로 Dropper를 전송하고 병원 측 예약 작업을 통해 자동 실행 | 전송 기록, PCAP, Sysmon, Task Scheduler 로그 | `T1195.002` `T1053.005` |
| **04** | 영향 | 주요 데이터 암호화, 복구 방해, 랜섬노트 생성 및 유출 가능성 확인 | 프로세스·파일 이벤트, PowerShell 로그, 네트워크 패킷 | `T1486` `T1490` |

> [!NOTE]
> ATT&CK 매핑은 시나리오 기준의 초기 분류입니다. 최종 보고서에서는 실제 관측된 행위와 증거를 기준으로 Technique/Sub-technique를 재검증합니다.

---

<a id="analysis"></a>

## 분석 전략

단일 로그만으로 결론을 내리지 않고, **호스트·네트워크·파일 증거를 시간축으로 교차 검증**합니다.

| 분석 구간 | 핵심 질문 | 주요 데이터 | 결과물 |
| --- | --- | --- | --- |
| **Initial Access** | 공격자는 언제, 어떤 요청으로 공급업체 서버에 진입했는가? | IIS Log, 웹 루트, SQL Server·세션 기록 | 최초 침투 시점, 웹 셸 경로, 관련 IoC |
| **Execution & Evasion** | 어떤 프로세스가 실행되었고 보안 기능·복구 수단은 어떻게 무력화되었는가? | Sysmon, Windows Event Log, PowerShell Log | 프로세스 트리, 계정 행위, 방어 회피 흔적 |
| **Propagation** | 악성 파일이 외부 C2가 아닌 정상 B2B 채널을 통해 유입되었는가? | PCAP, 인증·전송 기록, Sysmon | 전송 경로, 파일 해시, 실행 타임라인 |
| **Impact** | 어떤 데이터가 암호화·유출되었으며 서비스 영향은 어디까지인가? | 파일 이벤트, 랜섬노트, 네트워크 Flow, 시스템 아티팩트 | 피해 범위, 암호화 시점, 유출 가능성 |

### 중점 분석 항목

- IIS 요청과 웹 셸 생성 시점을 연결하여 최초 침투 구간 재구성
- B2B 서비스 계정의 로그인·토큰 사용 이력과 파일 전송 기록 검증
- 예약 작업 등록, Dropper 실행, 하위 프로세스 생성 순서 복원
- Defender 비활성화 및 Volume Shadow Copy 삭제 시도 확인
- 암호화 대상 확장자, 랜섬노트 생성 시점, C2·유출 트래픽 분석
- 서로 다른 시간대의 로그를 UTC 기준으로 정규화해 통합 타임라인 작성

---

## 가상 랩 구성

| 영역 | 구성 요소 | 수집·분석 데이터 |
| --- | --- | --- |
| **공급업체** | Ganda-it Transfer, IIS, Windows Server, SQL Server | IIS Log, 웹 루트 파일, DB·세션 기록 |
| **전송 구간** | 공급업체–병원 B2B 파일 전송 파이프라인 | PCAP, 인증 기록, 파일 전송 기록 |
| **병원 내부망** | 수신 서버, EMR DB, Active Directory, 백업 서버 | Event Log, Sysmon, PowerShell Log, 파일 아티팩트 |
| **관제·분석** | Wazuh, Wireshark | Alert, Network Flow, 상관분석 결과 |

---

## 탐지 및 대응

### Detection Engineering

- **YARA** — 악성 웹 셸, Dropper 및 랜섬웨어 특징 기반 탐지
- **Sigma** — 비정상 예약 작업, 백업 삭제, 보안 기능 비활성화 행위 탐지
- **Wazuh** — Sysmon·Windows 이벤트 수집, 상관분석 및 Alert 구성
- **Integrity Check** — B2B 수신 파일의 해시·서명·승인 상태 검증

### Incident Response

1. **식별** — Alert 검증, 영향 자산·계정·파일 범위 파악
2. **격리** — 감염 호스트와 B2B 전송 채널 분리, 악성 세션 차단
3. **보존** — 원본 로그·메모리·디스크·PCAP 확보 및 해시 기록
4. **제거** — 웹 셸·예약 작업·악성 파일 제거, 탈취 자격 증명 폐기
5. **복구** — 검증된 백업으로 복구하고 단계적으로 서비스 재개
6. **개선** — 탐지 규칙 보완, 서비스 계정 권한 분리, 재발 방지 대책 반영

---

<a id="deliverables"></a>

## 핵심 산출물

| 산출물 | 핵심 내용 |
| --- | --- |
| **공격 시나리오** | 공급사 최초 침투부터 병원망 랜섬웨어 실행까지의 재현 절차 |
| **통합 타임라인** | 로그·파일·패킷 증거를 표준 시간으로 정규화한 사고 타임라인 |
| **IoC / TTP 목록** | 해시, IP, 경로, 프로세스, 계정 행위 및 ATT&CK 매핑 |
| **탐지 규칙** | YARA Rule, Sigma Rule, Wazuh Alert와 검증 결과 |
| **IR 보고서** | 침투 원인, 피해 범위, 분석 근거 및 개선 권고사항 |
| **초동 대응 SOP** | 식별, 격리, 증거 보존, 제거, 복구, 사후 개선 절차 |

---

## 기술 스택

<div align="center">

![Windows Server](https://img.shields.io/badge/Windows%20Server-0078D4?style=flat-square&logo=windows&logoColor=white)
![Microsoft SQL Server](https://img.shields.io/badge/SQL%20Server-CC2927?style=flat-square&logo=microsoftsqlserver&logoColor=white)
![Wazuh](https://img.shields.io/badge/Wazuh-005571?style=flat-square&logo=wazuh&logoColor=white)
![Sysmon](https://img.shields.io/badge/Sysmon-334155?style=flat-square&logo=windows-terminal&logoColor=white)
![Wireshark](https://img.shields.io/badge/Wireshark-1679A7?style=flat-square&logo=wireshark&logoColor=white)
![MITRE ATT&CK](https://img.shields.io/badge/MITRE%20ATT%26CK-E31B23?style=flat-square)
![YARA](https://img.shields.io/badge/YARA-111827?style=flat-square)
![Sigma](https://img.shields.io/badge/Sigma-4338CA?style=flat-square)
![Git](https://img.shields.io/badge/Git-F05032?style=flat-square&logo=git&logoColor=white)

</div>

---

<a id="team"></a>

## 팀 구성

| 담당 | 팀원 | 주요 업무 | 핵심 산출물 |
| --- | :---: | --- | --- |
| **IR / PM Lead** | 윤희찬 | 시나리오·일정·조사 방향 및 증거 기준 관리 | 전체 Timeline, IR 보고서 |
| **Infrastructure / Network** | 안현민 | 공급업체·병원 서버 및 B2B 네트워크 구축 | 인프라 구성도, Network Flow |
| **Attack / Malware** | 조영우 | 공격 시나리오, Dropper 및 랜섬웨어 행위 구현 | 공격 시나리오, IoC |
| **Detection / SIEM** | 선우민정 | Wazuh·Sysmon 구축, Alert 및 탐지 규칙 구성 | Wazuh Dashboard, Sigma Rule |
| **Digital Forensics** | 김신아 | Event·IIS Log, 파일 및 프로세스 분석 | 포렌식 분석, Timeline |
| **Threat Hunting / Reporting** | 김서영 | IoC·TTP 분석 및 MITRE ATT&CK 매핑 | 공격 그래프, 최종 보고서 |

> 각자 담당 영역을 중심으로 수행하되, 전 구성원이 침해사고 대응 역량을 전체적으로 경험할 수 있도록 공격 재현부터 분석·탐지·대응까지의 전체 흐름을 함께 수행함.

---

## 프로젝트 일정

| 기간 | 단계 | 주요 과업 |
| :---: | --- | --- |
| **09.01 ~ 09.04** | 착수 | 자료 조사, 시나리오 확정, 가상 공격·방어 랩 구축 |
| **09.05 ~ 09.11** | 수행 1차 | 공급업체 최초 침투 및 B2B 전파 재현 |
| **09.12 ~ 09.16** | 수행 2차 | 랜섬웨어 실증, DFIR 수집·분석, 탐지 규칙 및 방어 대책 수립 |
| **09.17** | 결과 보고 | 최종 보고서 작성 및 결과 발표 |

---

## 📁 저장소 구조

```text
.
├── README.md
├── docs/
│   ├── architecture/          # 시스템 및 네트워크 구성도
│   ├── attack-scenario/       # 공격 시나리오와 재현 절차
│   ├── forensics/             # 포렌식 분석 기록
│   ├── threat-hunting/        # IoC·TTP 및 ATT&CK 매핑
│   └── response/              # 초동 대응 SOP와 개선 권고안
├── evidence/
│   ├── README.md              # 증거 관리 정책 및 메타데이터
│   ├── hashes/                # 증거 파일 해시 목록
│   ├── timeline/              # 정규화된 통합 타임라인
│   └── sanitized/             # 비식별·무해화된 공개 가능 자료
├── rules/
│   ├── yara/                  # YARA 탐지 규칙
│   ├── sigma/                 # Sigma 탐지 규칙
│   └── wazuh/                 # Wazuh Decoder 및 Alert Rule
├── scripts/
│   ├── collection/            # 증거 수집 자동화 스크립트
│   ├── analysis/              # 로그·아티팩트 분석 스크립트
│   └── validation/            # 탐지 규칙 검증 스크립트
├── reports/
│   ├── incident-response/     # 침해사고 분석 보고서
│   └── presentation/          # 최종 발표 자료
├── assets/
│   ├── diagrams/              # 아키텍처 및 공격 흐름도
│   └── screenshots/           # 대시보드 및 분석 화면
├── .gitignore
└── LICENSE
```

> 위 구조는 권장안이며 실제 프로젝트 진행 상황에 따라 변경될 수 있습니다.  
> 원본 증거와 악성 샘플은 공개 저장소에 업로드하지 않고 별도의 통제된 저장소에서 관리합니다.

---

## 분석 및 보안 원칙

- 모든 공격 실증은 **외부 네트워크와 완전히 차단된 가상 랩**에서만 수행합니다.
- 모든 결론은 PCAP, OS·웹 서버 로그, 파일 해시 등 **재현·검증 가능한 증거**에 근거합니다.
- 증거 원본은 읽기 전용으로 보존하고 수집 시각, 출처, 담당자 및 해시값을 기록합니다.
- 공개 저장소에는 실제 악성코드, 자격 증명, 개인정보 및 즉시 악용 가능한 공격 자료를 업로드하지 않습니다.

---

<sub>K-Shield Junior 17기 · 침해사고 대응 및 분석 · 4조</sub>

</div>
