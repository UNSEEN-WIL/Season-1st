## Terminology

프로그래밍 언어는 일반적으로 컴파일 언어와 해석 언어로 나뉘어진다.

## Compiled Language(컴파일 언어)

프로그램이 실행되기 이전에, 프로그램을 기계 언어로 컴파일하기 위해서는 특별한 컴파일 진행 과정이 필요하다. 런타임에 재해석될 필요가 없고, 컴파일된 결과는 직접적으로 사용된다. 프로그램 실행 효율성이 높고 컴파일러에 따라 다르지만 크로스 플랫폼 성능은 떨어진다. C, C++, Delphi 등이 있다.

## Interpreted Language(인터프리터 언어)

프로그램은 사전 컴파일 없이 쓰여지고, 텍스트 형태로 코드가 저장된다. 프로그램을 출시할 때는, 컴파일 과정은 이미 저장된 것 처럼 보인다. 그러나, 프로그램을 실행할 때 실행 전에 해석 언어가 해석되어야 한다.

그러나, 자바나 C#같은 언어가 해석 언어인지에 대해서는 논란의 여지가 있다. 왜냐하면 이들의 주류적인 구현은 직접적으로 해석되거나 실행되지 않고, 바이트코드로 컴파일 된 다음 jvm과 같은 가상 머신에서 실행되기 때문이다.

UE4 블루프린트의 구현은 Lua의 구현과 더욱 유사하다. 독립적으로 실행될 수 없지만, 호스트 언어에 임베디드된 확장된 스크립트의 형태로는 실행될 수 있다. Lua는 직접적으로 해석되거나 실행될 수 있고, 바이트코드로 컴파일되어 디스크에 저장된다. 실행을 위한 다음 호출시에 당신은 컴파일된 바이트코드를 직접적으로 로드할 수 있다.

## 가상머신은 무엇인가?

가상머신은 실제 기계의 독립적인 복사로 원래 정의되었다. 현재 가상 머신은 실제 기계와는 아무런 관련이 없다. 가상머신은 이들의 사용과 기계와의 직접적인 연관성에 따라 2가지 종류로 나뉜다. VirtualBox와 같은 시스템 가상 머신은 완전한 OS를 구동할 수 있는 완전한 시스템 기반을 제공한다. 반면, Java JVM과 같은 프로그램 가상 머신은 단 하나의 컴퓨터 프로그램, 즉 단 하나의 프로세스를 실행하기 위해 디자인되었다. 가상머신의 핵심적 특징은 가상머신에서 돌아가는 소프트웨어가 가상 세계를 초과할 수 없는 가상 머신에 의해 제공된 자원에 제한된다는 점이다.

여기서는 프로그램 가상 머신을 주로 다룬다. VM이 “기계"라고 불리기 때문에, 인풋은 일반적으로 ISA(instruction set architecture, 명령어 집합)를 만족하는 명령 시퀀스로 간주된다. 만약 실행되면, 아웃풋은 프로그램의 실행 결과이며, 이것이 VM이다. 소스와 타겟 ISA는 같을 수 있는데, 이것이 소위 same-ISA VM이라 불리는 것이다.

## 분류

가상머신의 구현은 레지스터-기반 가상 머신과 스택-기반 가상 머신으로 나뉜다.

## Three-address instruction

`a = b + c;`

다음과 같이 바꿀 수 있다:

`add a, b, c`

이게 더 기계어처럼 보인다. 이것이 소위 3-주소 명령이라고 불리고 일반적 형식은

`op dest, src1, src2`

보통 2개와 + 연산자가 많이 쓰인다. 3-주소 명령은 정확히 두 소스와 한 타겟으로 이루어지고, 이진 연산과 할당의 조합을 유연하게 지원한다. ARM 프로세서의 주요 명령은 3-주소 형식을 띈다.

## Two address instructions

`a += b;`

become:

`add a,b`

이것이 소위 2-주소 명령이고, 일반적인 형태는

`op dest, src`

이진 연산을 지원하기 위해, 동시에 소스들 중 하나만 타겟으로 잡을 수 있다. 위에 있는 add a, b를 실행하고 나면, 원래 값은 사라진다. 반면 b의 값은 바뀌지 않고 남아있다. x86 시리즈 프로세서는 2-주소 명령 형식으로 이루어져 있다. 

## One address instruction

명백히, 명령어는 자연수 “n개 주소"가 있어야 된다. 그렇다면 1-주소로 이루어진 명령어는 무엇일까?

다음과 같은 명령어들을 생각해보자.

```cpp
add 5
sub 3
```

이 명령어는 오직 연산자와 소스만 정의한다. 타겟은 어디에? 일반적으로, 이런 종류의 연산 타겟은 “accumulator"라고 불리는 특수한 레지스터다. 모든 연산은 축적자의 상태를 업데이트함으로써 끝난다. 그럼 아래 C로 쓰여진 두 명령어를 봐보자.

```cpp
acc += 5;
acc -= 3;
```

즉 acc가 “숨겨진" 타겟이 된다. 축적자-기반 아키텍쳐는 오늘날에는 희미하며, 종종 아주 오래된 기계에서만 쓰였다.

## Zero address instruction

만약 n개의 주소에서 n이 0이면 어떻게 될까?

다음의 자바 바이트코드 예시를 보자.

```java
Java bytecode code
iconst_1
iconst_2
iadd
istore_0
```

iadd(정수 덧셈) 명령어가 인자가 없음에 주목하자. 심지어 소스는 정해지지도 않았다. 0-주소 명령어의 쓰임새는 무엇일까?

0-주소는 소스와 타깃이 명백한 인자임을 의미하고, 이것의 구현이 일반적인 데이터 구조(스택)에 의존하고 있음을 의미한다. 위의 두 명령어(iconst_1과 iconst_2)는 정수 상수 1,2를 “evaluation stack"이라고 하는 곳에 쌓는다. 이 스택은 operand stack이나 expression stack이라고도 불린다. iadd 명령어는 두 값을 evaluation stack의 탑으로부터 빼내고, 값을 더하고, 이 결과값을 스택의 탑으로 다시 쌓는다. istore_0 명령어는 스택의 탑에서 값을 빼내고 로컬 변수 지역의 첫번째 위치(slot 0)에 저장한다.

0-주소의 형태로 구성된 명령어는 일반적으로 스택 기반 아키텍쳐에서 구현된다. 여기서의 스택이 시스템 콜 스택이 아니라 “evaluation stack"이라는 점을 명심하자. 어떤 가상 머신은 시스템 콜 스택에 evaluation stack을 구현했는데, 그런 머신은 개념적으로 다른 머신이다.

명령어의 소스와 목표가 명확하기 때문에, 0-주소 명령어의 밀집도는 매우 높다. 더 많은 명령어가 더 적은 공간에 쌓인다. 그러므로, 공간이 중요한 환경에서 0-주소 명령어는 고려할만한 디자인이다. 그러나 0-주소 명령어는 한 개의 명령어만 완수해야 한다. 일반적으로는 2-주소 명령어나 3-주소 명령어가 훨씬 많다. 다음 사례는 두개의 x86 명령어를 사용하면서 완수되었다.

```java
mov eax, 1
add eax, 2
```

## 스택-기반과 레지스터-기반 구조의 차이

1. 임시값이 저장되는 장소가 다르다.
    1. 스택-기반 : evaluation stack
    2. 레지스터-기반 : 레지스터
2. 코드가 차지하는 볼륨이 다르다.
    1. 스택-기반 : 코드가 컴팩트하고 작지만, 많은 코드 조건이 필요하다.
    2. 레지스터-기반 : 코드가 상대적으로 크지만, 적은 코드 조건이 필요하다.

스택에 기반한다고 했을 때 스택이 의미하는 것은 “evaluation stack"이며, 자바 JVM에서 이는 “operand stack"이라고 한다.
![image](https://github.com/UNSEEN-WIL/Season-1st/assets/103979407/f1fa4bd4-1188-4767-a326-8ab9d1026b2a)

## 스택 프레임

스택 프레임은 프로세스 활동 기록이라고도 불린다. 스택 프레임은 컴파일러가 프로시저 / 함수 콜을 구현하기 위해서 사용하는 데이터 구조다. 논리적으로, 스택 프레임은 함수 실행 환경이다. 함수 인자, 함수 로컬 변수, 함수가 실행된 곳으로 되돌아갈 장소 등등으로 구성된다.

## 출처

https://ikrima.dev/ue4guide/engine-programming/blueprints/bp-virtualmachine-overview/
