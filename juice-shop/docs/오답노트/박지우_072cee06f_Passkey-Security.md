# OWASP Juice Shop 커밋 분석 - 072cee06f

## 개요

- 프로젝트: OWASP Juice Shop
- 커밋 해시: `072cee06f`
- 커밋 날짜: 2026-10-10
- 작성자: Jannik Hollenbach
- 커밋 제목: Remove security question and increase strength of passkey users password
- 변경 파일: `data/static/challenges.yml`, `data/static/users.yml`
- 관련 PR: 확인되지 않음
- 관련 릴리즈: v20.4.0

## 변경 배경

OWASP Juice Shop은 보안 취약점을 학습하기 위해 의도적으로 취약하게 설계된 웹 애플리케이션이다.

이번 커밋은 Passkey 사용자 계정에 설정된 비밀번호를 강화하고 보안 질문을 삭제하는 변경이다. 개발자는 비밀번호 무차별 대입 공격을 통한 로그인을 원하지 않는다는 이유를 커밋 메시지에 명시했다.

## 변경 내용

### challenges.yml

Passkey Hijack 실습 문제의 설명에 `existing`이라는 단어가 추가되었다.

이는 새로운 패스키를 등록하는 것이 아니라 기존에 등록된 패스키를 대상으로 실습해야 한다는 점을 명확하게 한 변경이다.

### users.yml

`passkey-user` 계정의 기존 비밀번호가 더 길고 복잡한 문자열로 변경되었다.

또한 다음과 같은 보안 질문 정보가 삭제되었다.

```diff
- securityQuestion:
-   id: 2
-   answer: 'Whiskers'
```

## 보안 관점 분석

비밀번호 강화는 무차별 대입 공격을 통한 계정 접근을 어렵게 만드는 조치이다. 보안 질문 삭제는 해당 계정에서 질문과 답변을 이용하는 인증 또는 계정 복구 경로를 줄일 수 있다.

그러나 이번 커밋에서 로그인 요청 횟수 제한이나 패스키 인증 검증 로직 자체가 변경된 것은 확인되지 않았다.

따라서 이번 변경을 패스키 취약점 자체에 대한 수정으로 판단하기보다는, 교육용 계정의 접근 조건을 조정한 변경으로 보는 것이 적절하다.

## 릴리즈 및 CVE 조사

v20.4.0 릴리즈에서 Passkey Bypass, Passkey Hijack 실습 문제와 Passkey/WebAuthn 인증 기능이 추가되었다.

이번 커밋과 직접 연결되는 CVE 번호는 확인되지 않았다.

## 결론 및 시사점

이번 분석을 통해 보안 관련 커밋이 반드시 취약한 인증 코드를 직접 수정하는 것은 아니라는 점을 확인했다.

교육용 웹 애플리케이션에서는 사용자가 의도하지 않은 방법으로 문제를 해결하지 못하도록 실습 계정의 설정을 조정하는 작업도 중요하다.

향후에는 Passkey 인증 과정과 계정 복구 흐름을 추가로 분석하여 해당 변경이 실제 동작에 미치는 영향을 검증할 필요가 있다.

## 참고 자료

- 커밋: https://github.com/juice-shop/juice-shop/commit/072cee06f868efe855eb5dc3a6221a6739abe7f3
- 릴리즈: https://github.com/juice-shop/juice-shop/releases/tag/v20.4.0