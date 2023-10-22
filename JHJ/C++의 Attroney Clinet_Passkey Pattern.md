## C++ 에서 특정 클래스만 함수를 호출할 수 있게 하는 방법

### 서론

그냥 갑자기 문득 '입력을 한곳에서 받은 뒤 다른 곳에서는 읽기만 할 수 있게 못할까?' 라는 생각이 들었습니다. 유니티에서 `Input.~~`로 표현해 사용하는걸 자체적으로 만들 수 없을까 하는 생각이었습니다.

여기서 드는 생각이 '그러면 입력을 받아주는 곳은 한곳이고 나머지는 전부 읽기 전용이어야 한다' 입니다. 그러면 **어떻게 특정 클래스에게만 이런 권한을 줄 수 있냐**하는 의문이죠. 결국 이 의문이 커져서 이번에 소개할 `Attorney and Client idiom`과 `Passkey Pattern`을 보게 되었습니다.

## Attorney and Client idiom

일단 이 방식은 간단합니다. C++에서 제공하는 `friend` 키워드를 사용하는 것입니다.
```cpp
class Some
{
private :
	void WriteA() {}
	void WriteB() {}

friend class SomeAttroney;
};


class Attorney
{
public:
	void CallSomeWriteA(Some* SomePtr)
	{
		SomePtr->WriteA();
	}
	
friend class SomeClient;
};

class SomeClient
{
// 이곳에서 Attorney를 통해 함수를 호출하면 됩니다
// Attorney에서 제공해주는 Some의 함수들을 사용할 수 있습니다.
};
```

`Some`과 `SomeClient` 사이 바로 friend를 선언하게 되면 Some의 모든 private 함수에 접근이 가능하게 되므로 중간에 `Attorney`를 둬서 Attorney가 제공할 함수들을 선정하고 이 Attorney를 통해서 함수를 사용하는 것입니다.

간단하고 복잡하지 않은 구조라 이해가 쉽다는 장점이 있는 구조입니다. 다만 `friend`라는 만능 키워드를 사용하게 되기에 모든 캡슐화와 객체지향적인 설계가 무너지게 되며 클라이언트가 늘어날 때마다 friend를 계속해서 선언해줘야 한다는 단점이 있는 패턴입니다.

실제로는 잘 보지 못하게 될 패턴으로 생각됩니다. 제가 할 일도 없어 보이구요.

## Passkey Pattern
이전과는 다르게 템플릿을 사용하는 패턴입니다.
```cpp
template<typename T>
class Key
{
	friend T;
	Key() = default;  // 생성자가 private
};
```
이런 템플릿 클래스가 이제 키가 되는 것입니다.
```cpp
class Client;

class Actor
{
public:
	Vector GetActorLocation(Key<Client>)
	{
		return Location;
	}
};

class Clinet
{
public:
	void CalculateByActorLoaction(const Actor* SomeActor)
	{
		//...
		SomeActor->GetActorLocation({});
	}
}
```

위에서 Actor 클래스는 GetActorLocation 에서 `Key<Client>`를 인자로 받고 있습니다. 이로 인해서 Clinet만 접근 권한을 가지게 되는 것이죠.


> 외부에서 부를 수 없는 이유는 T 만이 friend 클래스가 되기 때문

이로 인해 Client 클래스 안에서만 저 함수를 호출할 수 있는 권한이 주어진 것입니다. 그리고 인자로 `{}`를 준 이유는 저렇게만 해도 컴파일러가 알아서  `Key<Camera>`를 넣어준다고 합니다.