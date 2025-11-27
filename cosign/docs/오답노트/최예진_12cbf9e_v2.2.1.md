# cosign 커밋 분석 - 12cbf9e
**주제: CVE-2023-46737 (DoS 취약점) 긴급 보안 패치 및 v2.2.1 기능 개선**

---

## 🧩 개요
- **커밋 해시:** `12cbf9ea177d22bbf5cf028bcb4712b5f174ebc6` (Short: `12cbf9e`)
- **주요 변경:** GitHub Security Advisory (GHSA-vfp6-jrw2-99g9) 병합 및 CVE-2023-46737 수정
- **관련 PR/릴리즈:** [v2.2.1 Release](https://github.com/sigstore/cosign/releases/tag/v2.2.1)

---

## 🔍 변경 요약
- **핵심 보안 패치:** 악의적으로 조작된 PEM 형식 입력값 처리 시 발생하는 무한 루프/리소스 고갈(DoS) 취약점(CVE-2023-46737)을 수정했습니다.
- **주요 기능 개선 (Enhancements):**
    - **SLSA 1.0 지원:** `Add SLSA 1.0 attestation support (#3219)` - 최신 공급망 보안 표준 준수
    - **오프라인 검증 강화:** `Rekor bundle`을 컨테이너에 첨부하여 폐쇄망 환경 지원 (#3246)
    - **인증 유연성:** 레지스트리 로그인 시 Basic Auth 및 Bearer Token 지원 (#3310)
    - **CLI 개선:** `cosign copy` 사용 시 서명, 증명, SBOM 동시 복사 지원 (#3247)
- **기타:** SBOM 첨부 기능 Deprecated 처리 (#3256) 및 문서화 개선

---

## 🔐 보안 관점 분석
| 항목 | 이전 버전 | 변경 버전 (v2.2.1) | 보안적 의미 |
| :--- | :--- | :--- | :--- |
| **인증서 파싱 (PEM)** | 악의적인 PEM 입력값에 대한 경계값 처리 미흡으로 무한 루프 발생 가능 | 파싱 로직에 입력값 검증 및 종료 조건 강화 | **가용성(Availability) 확보**: DoS 공격으로 인한 검증 서버/CI 파이프라인 마비 방지 |
| **타임스탬프 검증** | Root 인증서가 없어도 타임스탬프 검증 시도 (잠재적 오류) | `Fail timestamp verification if no root is provided (#3224)` | **무결성 강화**: 신뢰할 수 있는 루트(Root)가 없으면 검증을 실패 처리하여 보안 우회 방지 |
| **SCT 검증** | 특정 상황에서 불필요한 SCT(Signed Certificate Timestamp) 검증으로 오류 발생 | 불필요한 SCT 체크 비활성화 (#3237) | **안정성 개선**: 오탐(False Positive) 감소 및 검증 로직의 정확성 향상 |

---

## 🧾 검증 및 확인
- [x] 관련 PR/릴리즈 로그 확인 (Release v2.2.1)
- [x] 보안 패치(CVE-2023-46737) 포함 여부 확인 (GHSA-vfp6-jrw2-99g9)
- [x] 주요 기능(SLSA 1.0, Rekor Bundle) 업데이트 내역 확인

---

## 💬 결론
- [x] **업데이트 성격:** 보안 강화 (Critical Security Fix) 및 기능 개선 (Enhancements)
- [x] **CI/CD 신뢰성 영향:** 있음 (DoS 공격으로 인한 파이프라인 중단 방지 및 표준 준수 강화)
- [x] **추가 확인 필요사항:** - 구버전 사용 시 즉시 업그레이드 필요 ("Please upgrade to this release ASAP")
    - `cosign copy` 등의 CLI 플래그 변경 사항이 기존 스크립트에 영향을 주는지 확인 필요
    - Deprecated된 SBOM 첨부 기능 사용 여부 점검

---

## 📚 참고
- **PR 링크:** [Merge pull request from GHSA-vfp6-jrw2-99g9](https://github.com/sigstore/cosign/commit/12cbf9ea177d22bbf5cf028bcb4712b5f174ebc6)
- **릴리즈 노트:** [Cosign v2.2.1 Release Notes](https://github.com/sigstore/cosign/releases/tag/v2.2.1)
- **관련 보안 권고:** [Github Security Advisory (GHSA-vfp6-jrw2-99g9)](https://github.com/advisories/GHSA-vfp6-jrw2-99g9)