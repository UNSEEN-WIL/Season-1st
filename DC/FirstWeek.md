# 코드를 볼까요?

**1번**코드 입니다.

```cpp
	CustomData* allocatedCustomData = new CustomData();
	std::shared_ptr<CustomData> sharedCustomData(allocatedCustomData);
```

**2번**코드 입니다.

```cpp
	std::shared_ptr<CustomData> sharedCustomData = std::make_shared<CustomData>();
```

- 자 위 두 개의 코드를 보면 어떤가요? 코드는 좀 달라도 똑같은 기능을 하는 코드처럼 보이죠?
- 그런데 정말 재밌는 일이 일어납니다.

# weak_ptr와 함께 버무려보아요

```cpp
std::weak_ptr<CustomData> weakCustomData = sharedCustomData;
```

- 위 코드에 weak_ptr을 첨가해봅니다.

---

- **1번 코드**에 weak_ptr을 첨가하면…

![Untitled](https://prod-files-secure.s3.us-west-2.amazonaws.com/4bea4c1d-c6b3-4561-94d4-ca91e89bdb52/ff9fc0ef-d129-435c-a4a3-2829cc296cf3/Untitled.png)

- `sharedCustomData = nullptr;` 를 실행하면 메모리 해제가 잘 됐군요?
- 보세요, shared 포인터 내부에 있는 `_Ptr`값에 쓰레기값이 잔뜩 들어가있잖아요?

---

- **2번 코드**에 weak_ptr을 첨가하면…

![Untitled](https://prod-files-secure.s3.us-west-2.amazonaws.com/4bea4c1d-c6b3-4561-94d4-ca91e89bdb52/2d439099-0ad3-4128-8102-2e98be413f5a/Untitled.png)

- 아니? 왜 메모리해제가 안돼있죠?
- `weakCustomData`가 expired는 됐다고 써져있는데 해도 실제로  free는 호출됐지 않았네요?
- 보세요, `_Ptr`내부에 0으로 초기화해놓은 값이 그대로 있잖아요?

# std::make_shared 의 작동방식

![Untitled](https://prod-files-secure.s3.us-west-2.amazonaws.com/4bea4c1d-c6b3-4561-94d4-ca91e89bdb52/709d86d2-cd60-4b72-bf55-9460dfda5269/Untitled.png)

- `std::make_shared`도 내부적으로는 결국 malloc을 호출해야 하잖아요?
- 함수호출부를 보면 **48byte**크기만큼 메모리를 할당하는군요.
- 48byte는 어떻게 나온걸까요?
    - 제가 만든 `CustomData`클래스의 데이터크기 **32byte**
    - 이를 래핑하고있는`shared_ptr`이 내부적으로 담고있는 메타 데이터 크기 **16byte**입니다.
- 즉, `shared_ptr`이 담고있는 데이터랑 실제 데이터 내부의 메모리 할당은 동시에 하고 이걸 나눠서 쓴다는 뜻입니다.
- 그러면 메모리 해제도 동시에 진행돼야겠죠?
- 근데, `weak_ptr`이 실제 `CustomClass`를 레퍼런스 하지 않더라도 `shared_ptr`의 메타데이터를 내부적으로 레퍼런스 한다면 메모리 해제를 못하겠네요?
- 그래서  `std::make_shared`로 생성된 `shared_ptr`은 `weak_ptr`에 의해 참조되고있으면 프로그래머가 생각한대로 메모리 해제가 되지 않는답니다… 이런…

# 결론

- `std::make_shared`로 생성된 `shared_ptr`은 `weak_ptr`에 의해 참조되고있으면 refcount가 0이더라도 메모리가 해제되지 않는다.

# 코드

```cpp
#include <memory>

class CustomData {
public:
	int a = 0;
	int b = 0;
	float c = 0;
	float d = 0;
	float e = 0;

	CustomData() {}
	virtual ~CustomData() {};
};

int main() {

	printf("%d", sizeof(CustomData));

//	CustomData* allocatedCustomData = new CustomData();
//	std::shared_ptr<CustomData> sharedCustomData(allocatedCustomData);

	std::shared_ptr<CustomData> sharedCustomData = std::make_shared<CustomData>();

	std::weak_ptr<CustomData> weakCustomData = sharedCustomData;
	sharedCustomData = nullptr;
	weakCustomData.reset();

	return 0;
}
```

**[2018_1128_shared_ptr, weak_ptr작동방식 내부.pdf](https://github.com/megayuchi/ppt/blob/main/docs/2018_1128_shared_ptr%2C%20weak_ptr%EC%9E%91%EB%8F%99%EB%B0%A9%EC%8B%9D%20%EB%82%B4%EB%B6%80.pdf)**

**[원본 블로그글](https://velog.io/@jellypower/sharedptr%EC%97%90-weakptr%EC%9D%84-%EB%B6%99%EC%9D%B4%EB%A9%B4-%ED%95%A0%EB%8B%B9%ED%95%B4%EC%A0%9C%EB%A5%BC-%EC%95%88%EC%8B%9C%EC%BC%9C%EC%A4%80%EB%8B%A4%EA%B3%A0)**