## 0. Objectives
- 어노테이션 적용과 정의
- 리플렉션을 사용해 실행 시점에 객체 내부 관찰하기
- 예제

## 1. 어노테이션 선언과 적용
- 어노테이션을 사용하면 선언에 추가적인 메타데이터를 연관시킬 수 있다

### 1-1. 어노테이션을 적용해 선언에 표지 남기기
- 어노테이션을 이용하면 함수나 여러 클래스에 표지를 남길 수 있다
```.kt
@Test
fun testTrue() {...}
```
- `@Deprecated`는 이 선언이 더이상 사용되지 않을 것임을 명시한다
- `@Deprecated`는 최대 아래 3가지 파라미터를 받는다
- message: 사용 중단 예고 이유 설명
- replaceWith: 어떤 것으로 대체 되는지 설명
- level: 사용 중단에 대한 제한 수준
- level WARNING: 단순 경고
- level ERROR, HIDDEN: 컴파일 X
- HIDDEN: 이전에 컴파일된 코드와 호환성만 유지
```.kt
@Deprecated("Use removeAt(index) instead.", ReplaceWith("removeAt(index)"))
fun remove(index: Int) { ... }
```
- `@Deprecated` 가 명시된 함수를 사용하는 경우 인텔리제이에서 경고 메시지와 퀵 픽스도 제공해준다
- 어노테이셔 인자를 지정하는 문법은 자바와 조금 다르다
- `클래스를 어노테이션 인자로 지정`: @MyAnnotation(MyClass::class)처럼 ::class를 클래스 이름 뒤에 넣어야 한다
- 예를들어 직렬화 라이브러리는 @DeserializeInterface(CompanyImpl::class)처럼 역직렬화 과정에서 쓰이는 구현과 인터페이스를 연결하기 위해 클래스를 받는 어노테이션을 제공할 수 있다
- `다른 어노테이션을 인자로 지정`: 인자로 들어가는 어노테이션의 이름 앞에 @를 넣지 말라
- 예를들어 방금 살펴본 예시의 ReplaceWith앞에 @를 사용하지 않는다
- `배열을 인자로 지정`: 
