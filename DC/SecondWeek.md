# 코드를 봅시다

```cpp
for (int i = 0; i < 20; i++) { }

for (int i = 0; i < 20; ++i) { }
```

- 위 두 코드가 성능성으로 어떤 차이를 가졌는지는 많이들 들어보셨을 겁니다.
- “ `++i`이 `i++`보다 빠르다” C++하는 사람들은 한 번 쯤은 들어봤을법한 말이죠.
- 그런데, 진짜로 `++i`가 `i++`보다 빠른지 점검 해보신 적 있으신가요?
- 왜 `++i`가 `i++`보다 빠른지 이유는 알고 계신가요?

# 응 아니야~

```cpp
	for (int i = 0; i < 20; i++) { }
003050D2  mov         dword ptr [ebp-0Ch],0  
003050D9  jmp         __$EncStackInitStart+38h (03050E4h)  
003050DB  mov         eax,dword ptr [ebp-0Ch]  
003050DE  add         eax,1  
003050E1  mov         dword ptr [ebp-0Ch],eax  
003050E4  cmp         dword ptr [ebp-0Ch],14h  
003050E8  jge         __$EncStackInitStart+40h (03050ECh)  
003050EA  jmp         __$EncStackInitStart+2Fh (03050DBh)  

	for (int i = 0; i < 20; ++i) { }
003050EC  mov         dword ptr [ebp-18h],0  
003050F3  jmp         __$EncStackInitStart+52h (03050FEh)  
003050F5  mov         eax,dword ptr [ebp-18h]  
003050F8  add         eax,1  
003050FB  mov         dword ptr [ebp-18h],eax  
003050FE  cmp         dword ptr [ebp-18h],14h  
00305102  jge         __$EncStackInitStart+5Ah (0305106h)  
00305104  jmp         __$EncStackInitStart+49h (03050F5h)
```

*(VisualStudio 2019, C++ 14, Debug, x86 기준입니다)*

- 짜잔 어셈블리를 한 번 까봤습니다. 결국 C++을 컴파일하면 어셈블리에 대응될테고 이를 까보면 성능을 알 수 있지 않겠습니까?
- 지금은 어셈블리어를 잘 모르셔도 사실 상관 없습니다. 그저 우리는 `++i`와 `i++`에 해당하는 어셈블리 코드의 양이 **“동일하다”**는 것만 척 보면 알게됩니다.
- 다시 말하면 `++i`나 `i++`이나 성능상으로 “동일하다”는 뜻입니다.
- 컴파일러가 알아서 최적화 해줘서 그런거지 원론적으로 달라야 하는거 아니냐구요? 흠… 글쎄요? 저는 “디버그”모드로 빌드했는걸요?

# 그러면 ++i가 i++보다 빠르다는 말은 왜 나오는가?

- 그러면 왜 `++i`가 `i++`보다 빠르다고들 말하는 걸까요? 그 이유는 연산자 오버로딩에 있습니다.

```cpp
struct CounterStruct {
	int a;
	CounterStruct() : a(0) {}

	CounterStruct& operator++() { a = a + 1; return *this; } // ++CounterStruct 연산
	CounterStruct operator++(int) { a = a + 1; return *this; } // CounterStruct++ 연산
};
```

- 위와같이 간단한 **구조체**를 만들고 증감연산자를 간단하게 오버로딩 해주도록 하겠습니다.

```cpp
for (CounterStruct cs; cs.a < 20; cs++) { }

for (CounterStruct cs; cs.a < 20; ++cs) { }
```

- 그리고 똑같이 위 코드를 어셈블리어로 살펴보도록 하죠.

```cpp
for (CounterStruct cs; cs.a < 20; cs++) { }
00895103  lea         ecx,[ebp-24h]  
00895106  call        CounterStruct::CounterStruct (08913A7h)  
0089510B  jmp         __$EncStackInitStart+72h (089511Eh)  
0089510D  push        0  ; 추가된 코드
0089510F  lea         eax,[ebp-120h]  ; 추가된 코드
00895115  push        eax  ; 추가된 코드
00895116  lea         ecx,[ebp-24h]  
00895119  call        CounterStruct::operator++ (08913B1h)  
0089511E  cmp         dword ptr [ebp-24h],14h  
00895122  jge         __$EncStackInitStart+7Ah (0895126h)  
00895124  jmp         __$EncStackInitStart+61h (089510Dh)  

	for (CounterStruct cs; cs.a < 20; ++cs) { }
00895126  lea         ecx,[ebp-30h]  
00895129  call        CounterStruct::CounterStruct (08913A7h)  
0089512E  jmp         __$EncStackInitStart+8Ch (0895138h)  
00895130  lea         ecx,[ebp-30h]  
00895133  call        CounterStruct::operator++ (089139Dh)  
00895138  cmp         dword ptr [ebp-30h],14h  
0089513C  jge         __$EncStackInitStart+94h (0895140h)  
0089513E  jmp         __$EncStackInitStart+84h (0895130h)
```

- 이번에는 `cs++`코드가 `++cs`코드보다 더 깁니다! 드디어 `++cs`가 이겼군요! 그런데 이런일이 왜 일어날까요?
- 그 이유는 연산자 오버로딩 코드를 다시 보시면 이해가 되실겁니다.

```cpp
CounterStruct& operator++() { a = a + 1; return *this; } // ++CounterStruct 연산
CounterStruct operator++(int) { a = a + 1; return *this; } // CounterStruct++ 연산
```

- `++CounterStruct`연산자는 레퍼런스를 리턴하고 `CounterStruct++`연산자는 밸류를 리턴하도록 되어있죠?
- 연산자 오버로딩을 **“연산자”**라고 생각하지 말고 그냥 **“함수 호출”**이라고 생각해보자구요. `CounterStruct++` 함수는 실제 리턴된 밸류값을 사용하지 않더라도 **“밸류”를 리턴해야 하는 함수이기에 이를 위한 비용이 추가적으로 들어가야 하는게(원론적으론) 의무**입니다.
- 그러니 최적화가 되지 않은 통상적인 상황에선 리턴밸류에 대한 메모리 카피가 일어나기 때문에 `++CounterStruct`연산자가 `CounterStruct++`연산자보다 빠른 것이죠.

# 결론

- 개인적인 생각을 살짝 곁들이며 결론을 지어보도록 하겠습니다.
- 우선 성능상 결론입니다.
    1. 네이티브 타입 기준으론 `++i`와 `i++`의 성능차이가 존재하지 않는다.
    2. 사용자 정의 타입 기준으론 `++cs`와 `cs++`의 성능차이가 존재한다. 

# 개인적인 생각

1. 우리는 종종 어딘가에서 들은 정보를 그 타당성과 이유를 검증하지 않고 받아들이는 경우가 있습니다. 이러한 정보들은 한 번 씩 점검해봐야 할 사항들입니다. 특정 상황에서 그게 옳다고 그 말이 항상 옳은것이 아니기 때문입니다.
2. 증감연산자를 커스텀 클래스, 구조체에 오버로딩 할 일이 있으신가요? 1) 그런 상황에서 밸류를 대입, 카피하는 것을 가정하지 않고 2) 오버로딩된 연산자가 두 타입 모두 존재하며 3) 이를 활용해 증감을 해야하는 상황이라면 `++cs`를 쓰는 것이 맞습니다.
3. 그런데, 우리 솔직해져 보자구요. 사람에 따라서 다르겠지만, 저는 증감연산자를 쓰는 경우가 for문에서 네이티브 타입으로 이터레이션 돌리는 경우 말고는 잘 없거든요? 만약 for문을 저처럼만 사용하시는 분이라면…
    - `++i`가 `i++`보다 빠르다고 너무 쉽게 단언하지 않았는지 생각해 봅시다.
    - 새롭게 받아들이는 정보들에 대해 검증해보는 습관을 길러봅시다.

# 코드 전문

```cpp
#include <cstdio>

struct CounterStruct {
	int a;

	CounterStruct() : a(0) {}

	CounterStruct& operator++() { a = a + 1; return *this; }	// ++CounterStruct 연산
	CounterStruct operator++(int) { a = a + 1; return *this; }	// CounterStruct++ 연산
};

int main(void) {

	for (int i = 0; i < 20; i++) { }
	for (int i = 0; i < 20; ++i) { }

	for (CounterStruct cs; cs.a < 20; cs++) { }
	for (CounterStruct cs; cs.a < 20; ++cs) { }

	return 0;
}
```

**[원본 블로그글](https://velog.io/@jellypower/i-%EC%9D%80-i%EB%B3%B4%EB%8B%A4-%EB%8A%90%EB%A6%AC%EC%A7%80-%EC%95%8A%EB%8B%A4%EA%B5%AC%EC%9A%94)**