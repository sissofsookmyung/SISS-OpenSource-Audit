# [Falco] 커밋 분석 - be4af6f5
**주제:** 드라이버 로더가 실행되지 않을 때 호스트 `/proc` 마운트를 읽기 전용으로 제한

---

## 🧩 개요

- **커밋 해시:** [be4af6f5dfb0b877f5a29e1d8e007886898637cd](https://github.com/falcosecurity/falco/commit/be4af6f5dfb0b877f5a29e1d8e007886898637cd)
- **작성자 / 작성일 / 반영일:** dervoeti / 2026-10-09 / 2026-10-20
- **선정 근거:** 기존 보안 관련 키워드 검색 결과에 포함된 최근 커밋. 검색 명령은 `git log --all --grep='bug\|security\|fix\|vuln\|patch\|CVE' -i --pretty=format:'%h %ad %an %s' --date=short`이며, 커밋 메시지의 `fix`로 검색된다.
- **파일 변경:** 3개 파일, 45줄 추가·1줄 삭제
  - `chart/falco/templates/pod-template.tpl`: 읽기 전용 마운트 조건 변경
  - `chart/falco/tests/unit/falcoTemplates/procfsMount_test.go`: readOnly 기대값·설정 조합 검사 추가
  - `chart/falco/CHANGELOG.md`: 변경 설명 추가
- **관련 PR/릴리즈:** [수정 PR #4029](https://github.com/falcosecurity/falco/pull/4029), [backport PR #4043](https://github.com/falcosecurity/falco/pull/4043), [Helm Chart 9.2.1](https://github.com/falcosecurity/charts/releases/tag/falco-9.2.1)
- **분석 범위:** SISS 분석 1~5단계. 패치 실행 검증 전까지의 소스·PR·릴리즈·보안 공지 조사.

---

## 🔍 변경 요약

Falco 컨테이너의 `/host/proc` 마운트에 `readOnly: true`를 설정하는 조건을 수정했다.

```diff
         - mountPath: /host/proc
           name: proc-fs
-        {{- if and .Values.driver.enabled (not .Values.driver.loader.enabled) }}
+        {{- if eq (include "driverLoader.enabled" .) "false" }}
           readOnly: true
```

기존 조건은 드라이버가 켜져 있고 사용자가 `driver.loader.enabled=false`로 설정했을 때만 참이었다. 하지만 modern eBPF에서는 loader 설정의 기본값이 true여도 실제 드라이버 로더는 실행되지 않는다. 이 차이 때문에 필요하지 않은 쓰기 가능 마운트가 남을 수 있었다.

수정 후에는 같은 커밋의 `_helpers.tpl`에 정의된 `driverLoader.enabled`를 재사용한다. 이 helper는 `modern_ebpf` 또는 별칭 `modern-bpf` 사용, 드라이버 비활성화, 로더 비활성화 중 하나라도 해당하면 문자열 `false`를 반환한다. `include`의 결과가 문자열이므로 `"false"`와 비교한다.

마운트 존재 여부는 바깥의 `falco.procfsMount.enabled` 조건이 결정한다. 드라이버와 host-info 기능이 모두 꺼져 있으면 마운트 자체가 없어 읽기 전용 설정도 적용 대상이 아니다.

PR #4029의 설명과 코드의 수정 방향이 일치한다. 별도의 최초 신고 Issue 번호는 확인하지 못했다. 관련 기록에 함께 등장하는 `falcosecurity/charts#1053`은 ServiceMonitor 문제이므로 이 마운트 문제의 원인 Issue로 연결하지 않았다.

---

## 🔐 보안 관점 분석

아래는 소스 조건을 읽어 도출한 결과이며, Helm 렌더링이나 실제 배포로 검증한 결과는 아니다.

| 항목 | 이전 버전 | 변경 버전 | 보안적 의미 |
|------|------------|------------|--------------|
| modern_ebpf, driver=true, loader=true | readOnly 명시 안 됨 | readOnly=true | 실제 로더가 실행되지 않는 환경에서 불필요한 쓰기 가능 마운트 제거 |
| driver=false, plugins_hostinfo=true | readOnly 명시 안 됨 | readOnly=true | 호스트 정보 조회만 필요한 경우 읽기 전용으로 제한 |
| kmod, driver=true, loader=true | readOnly 명시 안 됨 | 기존 동작 유지 | 로더가 실행되는 조합의 동작은 유지 |
| driver=true, loader=false | readOnly=true | readOnly=true | 기존 읽기 전용 설정 유지 |
| driver=false, plugins_hostinfo=false | 마운트 없음 | 마운트 없음 | 마운트가 없는 조합과 읽기 전용 조합을 구분 |
| 단위 테스트 소스 | 마운트 존재 중심 확인 | readOnly 값과 드라이버 조합 검사 추가 | 설정 조건 변경에 따른 회귀를 검사할 수 있도록 보완 |

이 변경은 최소 권한 원칙에 따른 배포 설정 강화다. 기존 설정에서 모든 proc 항목에 쓰기가 가능했다거나, 이 조건만으로 컨테이너 탈출이 가능했다고 단정하지 않는다. 실제 영향은 컨테이너 권한, 커널 및 다른 보호 설정에 의존한다. 또한 readOnly 마운트 하나가 모든 경로의 접근을 통제하는 것은 아니다.

---

## 🧾 검증 및 확인

- [x] 관련 PR/릴리즈 로그 확인
  - #4029의 설명과 로컬 master의 반영 커밋을 대조했다. 작성일은 2026-10-09, 반영일은 2026-10-20이다.
  - #4043은 동일 수정을 `release/0.45.x`에 backport한다. 대응 커밋은 `82d5e62b144c62057ae01f9f303129ae72598e57`로, master의 분석 대상 해시와 구분한다.
  - `chart/falco/CHANGELOG.md`의 v9.2.1과 공식 Chart 릴리즈를 확인했다. **수정 반영 버전은 Helm Chart 9.2.1이며 Falco 실행 파일 0.45.1과 다른 버전 체계다.**
  - `6e13dff8`은 배포된 Chart 9.2.1 메타데이터를 master에 동기화하며 변경 설명을 Unreleased에서 v9.2.1로 옮긴다.
- [x] 보안 패치(CVE) 여부 확인
  - 2026-11-09 기준 공식 Falco 보안 공지 목록과 PR·릴리즈 기록을 확인했다.
  - `Falco "4029" "CVE"`, `Falco "proc" "read-only" "CVE" "2026"` 검색에서도 이 수정에 직접 연결된 CVE/GHSA는 확인하지 못했다.
  - CVE 미확인을 보안 문제가 없다는 증거로 해석하지 않았으며, 공식 심각도나 전체 영향 버전 범위를 임의로 부여하지 않았다.
- [ ] 테스트/빌드 정상 동작 확인
  - 사용자 요청에 따라 패치 검증은 진행하지 않았다. 테스트 소스는 읽었지만 실행하지 않았다.
  - Helm 렌더링, Kubernetes 배포, 호스트 procfs 쓰기 실험도 하지 않았다. 상위 PR 작성자가 보고한 검사는 이번 분석의 실행 결과와 구분한다.

확인 명령(Falco 저장소 폴더에서 실행):

```powershell
git show --format=fuller be4af6f5
git show --name-only --format='' be4af6f5
git diff 'be4af6f5^' be4af6f5 -- chart/falco/templates/pod-template.tpl
git show 'be4af6f5:chart/falco/templates/_helpers.tpl'
git show 6e13dff8 -- chart/falco/CHANGELOG.md chart/falco/Chart.yaml
```

---

## 💬 결론

- [x] 업데이트 성격: **보안 강화** — 실제 로더 실행 여부를 기준으로 호스트 procfs 마운트를 최소 권한으로 제한한다.
- [x] CI/CD 신뢰성 영향: **있음(회귀 검사 범위 측면)** — 기존 단위 테스트에 readOnly 단언과 설정 조합을 추가했다. CI 파이프라인 자체를 변경한 것은 아니며, 이번 분석에서 CI 통과를 확인하지는 않았다.
- [ ] 추가 확인 필요사항: 별도 패치 검증 단계에서 변경 전후 Helm 렌더링 결과와 실제 마운트 권한을 확인해야 한다.

설정값 하나와 실제 기능 활성 여부는 다를 수 있다. 배포 템플릿에서 이미 존재하는 활성 여부 계산 로직을 재사용하면 조건 불일치를 줄일 수 있다는 점을 배웠다. 또한 Falco 실행 파일만 업데이트하는 것과 Helm Chart의 배포 설정을 업데이트하는 것을 구분해야 한다.

---

## 📚 참고

- PR 링크: [#4029](https://github.com/falcosecurity/falco/pull/4029), [backport #4043](https://github.com/falcosecurity/falco/pull/4043)
- 릴리즈 노트: [Helm Chart falco-9.2.1](https://github.com/falcosecurity/charts/releases/tag/falco-9.2.1)
- 관련 문헌/참고자료:
  - [분석 커밋과 전체 Diff](https://github.com/falcosecurity/falco/commit/be4af6f5dfb0b877f5a29e1d8e007886898637cd)
  - [커밋 시점 helper 정의](https://github.com/falcosecurity/falco/blob/be4af6f5dfb0b877f5a29e1d8e007886898637cd/chart/falco/templates/_helpers.tpl)
  - [Chart 릴리즈 메타데이터 동기화](https://github.com/falcosecurity/falco/commit/6e13dff8fe017b5dfbb98708e20275ea2feba41f)
  - [Falco 공식 보안 공지 목록](https://github.com/falcosecurity/falco/security/advisories)
