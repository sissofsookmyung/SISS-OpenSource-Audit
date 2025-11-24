# ModSecurity 커밋 분석 - 6d7e8eb
**주제:** 2.9.12 버전 업데이트

---

## 🧩 개요
- **커밋 해시:** 6d7e8eb18f2d7d368fb8e29516fcdeaeb8d349b8
- **발견 날짜:** issue opened on Feb 16, 2021
- **파일 변경:** `apache2/apache2_io.c`, `apache2/mod_security2.c`
- **관련 CVE:** CVE-2025-54571
---

## 🔍 변경 요약
잘못된 요청에 대한 처리(return value)가 충분하지 못하게 설정되어서 XSS 공격 또는 소스코드 노출로 이어질 수 있었다.
기존에 `0` 또는 `-1`등의 정수 반환값을 `HTTP_INTERNAL_SERVER_ERROR` 등 구체적인 에러 매크로 반환으로 수정했다.
또한, 오류 처리에 있어서 모듈이 활성화 되어 있을 때만 커넥션을 종료하여 분기문에서 다루지 못한 부분에 있어서 커넥션이 alive 상태로 남아있던 것을, 모든 오류에 대해서 커넥션을 종료하도록 하여 의도하지 않은 접근을 사전에 방지하도록 하였다.
```diff
- if (bb_in == NULL) return -1;
+ if (bb_in == NULL) {
+   return HTTP_INTERNAL_SERVER_ERROR;
+ }

-  if (rc < 0 && msr->txcfg->is_enabled == MODSEC_ENABLED) {
-    switch(rc) {
-  ...
-      default:
-      break;
+  if (rc != OK) {
+    if (my_error_msg != NULL) {
+      msr_log(msr, 1, "%s", my_error_msg);
```

---
## 1. 어떤 보안 문제?
1. 400 Bad Request 이후에 Apache httpd server 측에서 `<html>It works!</html>` 메시지를 얻은 것으로 최초 발견되었다. 해당 응답은 `index.html`의 컨텐츠가 유출된 것이다.
2. 이는 클라이언트의 요청에 대해서 단일 응답이 반환되어야 한다는 원칙이 깨진 것이며, 이슈에서 제시된 오류 사례와 같이 내부 컨텐츠가 의도와는 달리 유출될 가능성이 존재한다는 문제점이 존재한다.
3. 또한, 커넥션이 올바르게 종료되지 않고 keep-alive 상태로 남으면서 Dos 공격 또는 예기치 않은 요청 흐름이 발생할 수 있다.

## 🔐 보안 관점 분석
| 항목 | v2.9.11 | v2.9.12 | 보안적 의미 |
|------|------------|------------|--------------|
| 연결 종료 보장 | mod_security2.c 파일에서 비교적 복잡한 분기문을 통해서 모듈이 활성화된 경우에만 종료 | 분기문을 간략화하여 모든 오류 발생에 대하여 `keepalive = AP_CONN_CLOSE`로 종료 처리 | 오류 발생 시 커넥션을 종료함으로써 오류를 통해서 다른 흐름을 발생시킬 수 있는 위험성 사전 차단 |
| 오류 코드 반환 방식 개선 | `return -1` 등으로 단순 음수 리턴 | `HTTP_INTERNAL_SERVER_ERROR` 등의 각각의 오류 상황을 그에 일치하는 HTTP 상태 코드로 반환함으로써 리턴 흐름을 명확하게 표기 | 추후 서비스 이용 중 오류 발생 시 즉시 연결을 종료하고, 다음 응답이 나올 수 없도록 다른 코드에서 보강 |

---

## 🧾 검증 및 확인
✔️ 관련 PR/릴리즈 로그 확인 : [v2.9.12 Release](https://github.com/owasp-modsecurity/ModSecurity/releases/tag/v2.9.12)
✔️ 보안 패치(CVE) 여부 확인 : CVE-2025-54571

---

## 💬 결론
✔️ 업데이트 성격: 보안 강화
✔️ CI/CD 신뢰성 영향: 없음
✔️ 추가 확인 필요사항: 없음

---

## 📚 참고
- PR 링크:  
- 릴리즈 노트:  
- 관련 문헌/참고자료: [CWE-252](https://cwe.mitre.org/data/definitions/252.html)
