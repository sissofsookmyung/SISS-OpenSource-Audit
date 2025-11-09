## 개요

---

- **커밋 해시:** 0274a4f3b403162a37a10f199c989f3727ed3ad4
- **발견 날짜:** committed on Jan 13, 2023
- **분석 대상**: sudoedit: do not permit editor arguments to include "--”
- **관련 CVE**: CVE-2023-22809

## 1. 어떤 보안 문제?

---

- sudo의 sudoedit 기능(또는 sudo -e)을 사용할 때, sudoers 파일에 허용되지 않은 **임의의 파일**을 ‘root 권한으로 편집’할 수 있는 로컬 권한 상승(LPE) 취약점
- sudoedit는 원래 root 권한으로 직접 파일을 편집하는 것이 아니라,

1.  파일을 `/var/tmp` 같은 임시 위치에 사용자의 권한으로 복사
2.  사용자가 일반 편집기로 이 '임시 파일'을 수정
3.  편집이 끝나면, 수정된 '임시 파일'을 원본 위치(예: /etc/httpd.conf)로 다시 덮어씀

**취약점**

- 공격자가 'sudoedit -- /etc/passwd'처럼 '--' 문자를 파일명과 함께 입력하면, 'sudo'의 argument parser가 '--'를 "여기서부터는 옵션이 아니라 파일명"이라는 뜻의 메타 문자로 잘못 해석함
  **결과**
  - ’--’ 뒤에 오는 파일(예: /etc/passwd)은 ‘sudoedit'의 안전한 '임시 파일 복사' 로직을 우회하고, 편집할 파일 목록에 날것으로 추가되었음

  - 이로 인해 ‘sudoers' 정책상 편집 권한이 없는 사용자라도 ‘/etc/passwd’ 같은 민감한 시스템 파일을 ‘root’ 권한으로 직접 열어서 수정할 수 있게 되었음

## 2. 왜 발생했는지?

---

- **입력 검증 부족: ‘**sudo’의 메인 인수 파싱 로직이 ‘sudoedit’ 모드에서 ‘--’ 문자를 만났을 때의 엣지 케이스를 제대로 처리하지 못했음
- **로직 결함: ‘**sudoedit’는 사용자가 지정한 파일과 편집기에 전달할 인수를 명확히 구분해야 함
  > > 하지만 ‘--’ 문자로 인해 이 구분이 무너졌고, 검증되지 않은 파일 경로가 편집 목록으로 바로 전달되는 경로 우회(Path Traversal)와 유사한 문제가 발생

## 3. 어떻게 해결? (Diff 분석)

---

- **파일: ‘**plugins/sudoers/parse.c’

````diff
``` diff
--- a/plugins/sudoers/parse.c
+++ b/plugins/sudoers/parse.c
@@ -744,7 +744,14 @@ parse_args(int argc, char **argv, int old_optind,
            }

            /* Handle "--" specified by user. */
-           if (strcmp(*av, "--") == 0) {
+           if (ISSET(sudo_mode, MODE_EDIT) && strcmp(*av, "--") == 0) {
+               /*
+                * The "sudoedit -- file" syntax is ambiguous.
+                * Disallow it to avoid surprises.
+                */
+               errorx(1, U_("sudoedit does not support the -- option"));
+           }
+           if (strcmp(*av, "--") == 0) {
                /* Stop parsing args, everything after is a command. */
                SET(settings, CMND_SAW_DASH_DASH);
                av++;
````

---

- **핵심 변경:** sudoedit 모드일 때(ISSET(sudo_mode, MODE_EDIT)) - 문자를 만나면, 이를 "인수 파싱 종료"로 처리하던 기존 로직 이전에 **새로운 검사 로직을 추가**
- **새로운 로직:**
  1. 현재 sudoedit 모드인지 확인
  2. 이때 사용자가 -를 입력했다면, "sudoedit does not support the -- option” 오류 메시지를 출력하고 즉시 종료(exit(1))
- **의미:** sudoedit 모드에서는 모호함을 유발할 수 있는 - 메타 문자 자체를 아예 금지시킴으로써, 공격자가 파싱 로직을 우회할 수 있는 경로를 원천 차단

## 4. 시사점 및 배울 점 (시스템/보안적 의미)

---

| **구분**            | **기존 로직**                                          | **수정 로직**                                               | **시스템/보안적 의미**                             |
| ------------------- | ------------------------------------------------------ | ----------------------------------------------------------- | -------------------------------------------------- |
| **-- 인수 처리**    | sudoedit 모드에서도 --를 "인수 파싱 종료"로 간주       | sudoedit 모드에서 --를 만나면 **오류로 간주하고 즉시 종료** | **비정상적인 사용자 입력 차단** (입력 유효성 검사) |
| **파일 경로 검증**  | -- 뒤의 파일은 sudoedit의 안전 로직을 우회 (Bypass)    | -- 자체를 허용하지 않으므로 우회 경로 원천 차단             | **공격 벡터(Attack Vector) 제거**                  |
| **프로그램 신뢰도** | 사용자가 sudoers 정책을 우회하여 시스템 파일 접근 가능 | sudoers 정책(Policy)을 엄격하게 강제함                      | **최소 권한 원칙(PoLP)** 및 시스템 무결성 강화     |

### 결론

이번 패치는 **권한이 높은 프로그램(SetUID 바이너리)이 사용자의 입력을 파싱할 때 얼마나 주의해야 하는지** 보여 주는 사례

- -, , , .. 등 쉘이나 프로그램에 의해 특별하게 해석될 수 있는 **메타 문자는 항상 잠재적인 보안 위협**으로 간주해야 함
- "이 입력이 파싱 로직을 건너뛰거나 예상치 못한 동작을 유발할 수 있는가?"를 항상 검증해야 하며, 가장 안전한 방법은 이 패치처럼 모호한 입력은 명시적으로 거부하는 것

### 📚 참고 자료

- **커밋:** git show 0274a4f
- **CVE:** [NVD - CVE-2023-22809](https://nvd.nist.gov/vuln/detail/CVE-2023-22809)
- **최초 분석:** [Synacktiv Blog: Sudoedit vulnerability (CVE-2023-22809)](https://www.synacktiv.com/sites/default/files/2023-01/sudo-CVE-2023-22809.pdf)
