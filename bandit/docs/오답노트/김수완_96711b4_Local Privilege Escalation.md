## 개요

- **커밋 해시:** ea0d187d78b2e6365e35f676d2eb9b1be264c091
- **발견 날짜:**  November 23, 2025
- **분석 대상** Eric Brown Check whether Constant value is str 

## 분석

- Python 3.14에서 문자열 리터럴을 나타내던 ast.Str 클래스가 제거되고, 모든 종류의 리터럴(문자열, 숫자, 불리언 등)을 처리하는 ast.Constant 클래스로 통합되었다.
- 기존 코드(bandit/core/utils.py의 concat_string 함수)는 ast.Constant 노드를 처리할 때, 노드 자체가 상수인지(isinstance(x, ast.Constant))만 확인했다. 그러나 ast.Constant의 value 속성은 문자열 외에도 정수, 실수, True/False 등 다양한 타입일 수 있다.

**취약점**

- bandit/core/utils.py의 concat_string 함수에서 ast.Constant 노드의 value 속성이 실제로 문자열 (str) 타입인지 확인하는 과정이 누락되었다.

**결과**
  - 문자열이 아닌 상수의 값(예: 정수 123)이 문자열 연결(" ".join(...)) 리스트에 포함되어 런타임 오류가 발생하거나(코드의 의도와 다른 타입으로 인해), 의도하지 않은 비문자열 값이 연결 문자열에 포함되어 로직 오류를 일으킬 수 있다.

  

## 변경내용

@@ -302,7 +302,13 @@ def concat_string(node, stop=None):
         _get(node, bits, stop)
     return (
         node,
-        " ".join([x.value for x in bits if isinstance(x, ast.Constant)]),
+        " ".join(
+            [
+                x.value
+                for x in bits
+                if isinstance(x, ast.Constant) and isinstance(x.value, str)
+            ]
+        ),
     )



## 결론

- concat_string 함수가 문자열 연결을 수행할 때 문자열 타입의 값만을 안전하게 처리하게 되어, Python 3.14의 ast.Constant 변경 사항으로 인해 발생할 수 있는 잠재적인 타입 에러(TypeError) 또는 잘못된 문자열 연결 문제를 해결했다.



### 📚 참고 자료

- **커밋:** git show ea0d187
- **레포 링크** https://github.com/PyCQA/bandit.git

