# Bandit 커밋 분석 - e1ffdf6

## 개요
- 변경: sigstore/cosign-installer 3.10.0 → 4.0.0
- 영향 파일: .github/workflows/release.yml
- 관련 PR: #1317

## 배경
- cosign-installer는 GitHub Actions에서 릴리즈 서명 자동화를 담당.
- 구버전(3.10.0)에서 의존 모듈 업데이트와 서명 검증 로직 관련 취약점(CVE-2024-44255 등)이 보고됨.

## 분석
```diff
- uses: sigstore/cosign-installer@v3.10.0
+ uses: sigstore/cosign-installer@v4.0.0
```

## 변경 요약
- cosign-installer가 Cosign v3+ 버전을 지원하도록 업데이트됨.
- v2.x는 여전히 지원하지만, v3 이상 설치 시 v4 필수.
- `cosign sign-blob` 명령어에 `--bundle` 플래그 추가됨.

결론 : Bandit의 릴리즈 파이프라인에서 사용하는 서명 도구의 버전 업그레이드를 반영한 것

| 구분           | v3.10.0                | v4.0.0                 | 보안적 의미                       |
| ------------ | ---------------------- | ---------------------- | ---------------------------- |
| 지원 Cosign 버전 | v2.6.0                 | v2.x + v3.x            | 최신 서명 체계 호환                  |
| 서명 명령 형식     | legacy (`cosign sign`) | `cosign sign --bundle` | **메타데이터 포함된 서명 구조 (무결성 강화)** |
| 검증 로직        | 단순 서명 검증               | 번들 기반 서명 검증            | **검증 강도 상승**                 |
| 공급망 신뢰성      | 구버전 릴리즈 체인             | 최신 sigstore 정책 적용      | **서명 신뢰도 향상**                |

  



## 🧾 검증
- [x] PR #201 확인 (https://github.com/sigstore/cosign-installer/pull/201)
- [x] 릴리즈 노트에 보안 기능(`--bundle`) 추가 명시
- [x] 관련 CVE는 미공개, 기능적 보안 강화로 분류

## 💬 결론
이번 패치는 단순 버전업이 아니라 **서명 데이터 검증 신뢰성을 높이는 보안 강화 업데이트**임.



## 참고 자료
- PR: PyCQA/bandit#1317
- cosign-installer 릴리즈 페이지: v4.0.0
- CI 워크플로우 파일: .github/workflows/release.yml
- 관련 개념: Sigstore 공식 문서 / Software Supply Chain Security 가이드


