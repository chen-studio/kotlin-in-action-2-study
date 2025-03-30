## 0. Objectives
- 코틀린의 nullable 타입에 대한 이해

## 1-1 NullPointerException을 피하고 값이 없는 경우 처리: nullable
- 코틀린은 NullPointerException을 피하기 위한 nullable 타입이 별도로 존재
- 코틀린은 이 방법으로 null에 대한 접근 방법을 컴파일 시점으로 옮겨 컴파일러가 널처리를 할 수 있도록 함

## 1-2 널이 될 수 있는 타입으로 널이 될 수 있는 변수 명시
```.java
int strLen(String s) {
  return s.length();
}
```
- 이 자바 함수의 파라미터로 null을 전달하면 NPE가 발생한다
- 이 함수를 코틀린으로 다시 쓸때는 파라미터로 null을 받을 수 있는지 고려해야 한다
```.kt
fun strLen(s: String) = s.length // null을 파라미터로 받지 못하는 경우

fun main() {
  strLen(null) // 컴파일 에러 
}
```
- 이 함수가 null을 파라미터로 받을 수 있는 경우 타입 뒤에 물음표를 명시 해야 한다
```.kt
fun strLenSafe(s: String?) = ...
```
- String? Any? MyCustomType등 어떤 타입이든 타입 이름 뒤에 물음표를 붙이면 그 타입의 변수나 프로퍼티에 null 참조를 저장할 수 있다
- 어떤 널이 될 수 있는 타입의 값이 있다면 그 값에 대해 수행할 수 있는 연산의 종류가 제한된다.

  
