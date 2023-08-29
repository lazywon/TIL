# 모바일에서 <input> 태그를 이용하여 키패드 설정하기

모바일에서 `<input>` 태그 영역을 터치하면 키패드가 나오는데, `<input>` 태그를 이용하여 모바일 키패드를 숫자 또는 특정 문자만 보이도록 설정하는 방법은 다음과 같다.

### `input` 태그의 `type` 속성과 `pattern` 속성 사용

- 숫자 입력 키패드 표시
  ```
  <input type="number">
  ```
- 위 방법은 Android에서는 동작하지만, IOS에서는 여전히 글자 자판이 같이 노출된다. IOS에서도 동작하게 하기 위해서 다음 `pattern` 속성을 추가해야 한다.

  ```
  <input type="number" pattern="\d*">
  ```

  - 정규표현식으로 `\d`는 숫자를 의미하며, `*`는 0번 이상의 반복을 뜻함

### `inputmode` 속성 이용

- inputmode` 속성을 이용하면 더 다양한 방식의 키패드를 컨트롤 할 수 있다.
- text는 default 값으로, type 속성에 따라 제공되는 가상 키보드가 표시된다.
  ```
  <input inputmode="text" />
  ```
- 소수점 제공 필요 시 사용한다. `,` 또는 `.` 또는 `-` 를 제공할 수 있다.

  ```
  <input inputmode="decimal" />
  ```

- 숫자 입력 키패드 표시

  ```
  <input inputmode="numeric" />
  ```

- 전화번호 입력 키패드 표시

  ```
  <input inputmode="tel" />
  ```

- 이메일 입력 키패드 표시

  ```
  <input inputmode="email" />
  ```
