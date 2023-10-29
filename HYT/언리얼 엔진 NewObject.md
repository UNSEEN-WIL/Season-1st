# UObject Creation / Initialization

### NewObject 요약

1. 같은 이름의 오브젝트가 이미 존재하는지 빠르게 찾는다.
2. 만약 존재한다면, 원래 것을 파괴하고, 해당 메모리 블럭 m을 사용한다.
3. 같은 이름이 없다면, 클래스 리플렉션 정보에 따라 메모리 블럭 m이 할당된다.
4. 생성자 `ObjectBase`는 메모리 블럭 m에서 호출된다.
5. 마지막으로, 메모리 블럭 m에서 목표 클래스의 생성자를 호출한다. 여기서 `UObjectBase` 함수가 오브젝트 등록을 완수한다. 

### 할당, 생성 및 초기화 상세

1. `NewObject`
    
    몇몇 조건 감지에 더해, `StaticConstructObject_Internal`의 이전 짝이 조정된다. 여기 몇몇 인자가 있다. 4번째 인자는 템플릿인데, 이를 통해 당신은 오브젝트를 카피할 수 있다.
    
    ```cpp
    template< class T > 
    FUNCTION_NON_NULL_RETURN_START 
    T* NewObject
    (
    	UObject* Outer, 
    	FName Name, 
    	EObjectFlags Flags = RF_NoFlags, 
    	UObject* Template = nullptr, 
    	bool bCopyTransientsFromClassDefaults = false, 
    	FObjectInstancingGraph* InInstanceGraph = nullptr
    ) 
    FUNCTION_NON_NULL_RETURN_END
    ```
    
2. `StaticConstructObject_Internal` : `StaticAllocateObject`를 사용한 메모리 할당과 와 오브젝트 초기화를 담당
    
    ```cpp
    UObject* StaticConstructObject_Internal 
    ( 
    	UClass* InClass, 
    	UObject* InOuter /*=GetTransientPackage()*/, 
    	FName InName /*=NAME_None*/, 
    	EObjectFlags InFlags /*=0*/,
    	EInternalObjectFlags InternalSetFlags /*=0*/, 
    	UObject* InTemplate /*=NULL*/, 
    	bool bCopyTransientsFromClassDefaults /*=false*/, 
    	FObjectInstancingGraph* InInstanceGraph /*=NULL*/, 
    	bool bAssumeTemplateIsArchetype /*=false*/ 
    )
    ```
    
3. `StaticAllocateObject`: 오브젝트 메모리를 주로 할당하고, 네임스페이스를 설정하고 등록한다.
    
    먼저 새로운 오브젝트를 만들지, 기존 오브젝트를 대체할지 체크한다. 항상 `Name` 인자가 넘어오기 때문이다. 만약 같은 이름의 오브젝트가 존재하는 것을 발견했다면, 해당 오브젝트를 강제로 파괴하고, 그 공간을 다시 사용한다. 그렇지 않다면 직접 공간을 할당한다 :
    
    ```cpp
    UObjectBase* FUObjectAllocator::AllocateUObject
    (
    	int32 Size, 
    	int32 Alignment, 
    	bool bAllowPermanent
    )
    ```
    
    이 주요 작업에 더해서, 오브젝트 간의 연결을 관리하거나, 비동기 쓰레드에서 오브젝트가 만들어졌을 때 알림을 보내는 등 여러 작업을 한다.
    
    ```cpp
    UObject* StaticAllocateObject 
    (
    	UClass* InClass, 
    	UObject* InOuter, 
    	FName InName, 
    	EObjectFlags InFlags, 
    	EInternalObjectFlags InternalSetFlags, 
    	bool bCanRecycleSubobjects, 
    	bool* bOutRecycledSubobject 
    )
    
    /** * Constructor used by StaticAllocateObject * 
    			@param InClass non NULL, this gives the class of the new object, if known at this time * 
    			@param InFlags RF_Flags to assign * 
    			@param InOuter outer for this object * 
    			@param InName name of the new object * 
    			@param InObjectArchetype archetype to assign */
    UObjectBase::UObjectBase(
    	UClass* InClass, 
    	EObjectFlags InFlags, 
    	EInternalObjectFlags InInternalFlags, 
    	UObject *InOuter, 
    	FName InName
    ) : ObjectFlags (InFlags)
    ```
    
4. 위 단계에 의해 할당된 공간에, 이렇게 오브젝트를 생성할 수 있다.
    
    ```cpp
    Result = StaticAllocateObject(InClass, InOuter, InName, InFlags, InternalSetFlags, bCanRecycleSubobjects, &bRecycledSubobject);
    (*InClass->ClassConstructor)( FObjectInitializer(Result, InTemplate, bCopyTransientsFromClassDefaults, true, InInstanceGraph) );
    ```
    
    여기에 있는 `ClassConstructor`는 모든 `UClass`가 갖고 있는 함수 포인터 클래스 멤버 변수다. 즉, `UClass`에 있는 모든 포인터는 전역 템플릿 함수를 가리킨다:
    
    ```cpp
    template<class T>
    void InternalConstructor( const FObjectInitializer& X )
    {
      T::__DefaultConstructor(X);
    }
    ```
    
    각 클래스에 있는 `__DefaultConstructor` 또한 공통적으로 매크로를 사용해서 만들어지고, 그 내용은 간단히 새롭게 전해진다:
    
    ```cpp
    static void __DefaultConstructor(const FObjectInitializer& X) 
    { 
    	new((EInternal*)X.GetObj())TClass(X); 
    }
    ```
    
    이 두가지 새로운 형식은 매우 중요하다. `operator new`의 인자인 `[(EInternal *)X.GetObj()]`과 이 클래스의 실질적 생성자로 넘어온 `FObjectInitializer& X`.
    
    후자는 `InClass->ClassConstructor`에 필요한 실질적 인자고, 전자는 `DECLARE_CLASS` 매크로를 통해 각 `UObject`에서 새로 정의된 new 연산자다.
    
    ```cpp
    inline void* operator new( const size_t InSize, EInternal* InMem )
    {
        return (void*)InMem;
    }
    ```
    
    전체 과정을 요약해보면 이렇다:
    
    - `StaticAllocateObject`가 `Result` 메모리 공간을 할당하고, 이를 `FObjectInitializer X`를 생성하기 위한 인자로 사용한다. 그리고 `X.GetObj`가 이 메모리 주소 `Result`를 반환한다.
    - 그리고 오브젝트를 생성하기 위해서 `new (Result) TClass (X)`를 사용하고, `Result`의 주소를 특정짓고, `X`를 그 생성자의 인자로 넘긴다.
5. `FObjectInitializer`에 대해
    
    이전 단계에 이어 생성자가 리턴되고 나면, 임시 인자 `[FObjectInitializer& X]`가 자동으로 파괴된다. 그러므로, UE4는 많은 것을 하기 위해 이 단계를 이용하고, 대부분은 시스템의 내부 상태와 관련되어 있다. 관리를 위해서 완전히 이해하기는 어렵게 만들어져 있다.
    
    그러나 `InitProperties`에서 이루어지는 실행과 관련된 한 연산, 즉 속성 초기화가 존재한다.
    
    ```cpp
    void FObjectInitializer::InitProperties(UObject* Obj, UClass* DefaultsClass, UObject* DefaultData, bool bCopyTransientsFromClassDefaults)
    {
    　　……
        UClass* Class = Obj->GetClass();
    　　……if (!bNeedInitialize && bCanUsePostConstructLink)
        {
            // This is just a fast path for the below in the common case that we are not doing a duplicate or initializing a CDO and this is all native.
            // We only do it if the DefaultData object is NOT a CDO of the object that's being initialized. CDO data is already initialized in the
            // object's constructor.
            if (DefaultData)
            {
                if (Class->GetDefaultObject(false) != DefaultData)
                {
                    QUICK_SCOPE_CYCLE_COUNTER(STAT_InitProperties_FromTemplate);
                    for (UProperty* P = Class->PropertyLink; P; P = P->PropertyLinkNext)
                    {
                        P->CopyCompleteValue_InContainer(Obj, DefaultData);
                    }
                }
                else
                {
                    QUICK_SCOPE_CYCLE_COUNTER(STAT_InitProperties_ConfigEtcOnly);
                    // Copy all properties that require additional initialization (e.g. CPF_Config).
                    for (UProperty* P = Class->PostConstructLink; P; P = P->PostConstructLinkNext)
                    {
                        P->CopyCompleteValue_InContainer(Obj, DefaultData);
                    }
                }
            }
    　　……
    ```
    
    `UClass`에 기록된 속성 메타데이터를 탐색하면서, 현재 객체의 각 특성에 해당하는 값을 할당할 수 있다.
    
    `DefaultData` 인자에서 흥미로운 것은, 가장 처음의 템플릿 인자다. 당연히, 템플릿이 비어있다면 여기의 `DefaultData`는 이 클래스의 CDO가 된다.
    
    이 코드는 두 가지 상황을 다루는 확연히 다른 두 전력을 보여준다.
    
    만약 오브젝트가 템플릿에서 `DefaultData`로 복사되었다면, 클래스에 있는 모든 프로퍼티가 관찰되어야 한다. 왜냐하면 실제로 타겟을 복사하기 위해서 어떤 프로퍼티가 바뀌었는지 알 수 없어서, 전체 디스크를 복사해야 한다.
    
    그리고 만약 CDO로부터만 복사되었다면, `CPF_Config`로 마크된 영역처럼 초기화 상태를 갖고 있는, 명백히 특정된 영역만 해당 클래스에서 처리하면 된다. 그리고 시작할 때 상응하는 설정값을 추출하기 위해 ini 파일을 참고하게 된다.
    

### 전역 오브젝트 테이블

```cpp
/** * Add a newly created object to the name hash tables and the object array * * @param Name name to assign to this uobject */
void UObjectBase::AddObject(FName InName, EInternalObjectFlags InSetInternalFlags)
 { 
		NamePrivate = InName; 
		EInternalObjectFlags InternalFlagsToSet = InSetInternalFlags; 
		if (!IsInGameThread())
		{ 
			InternalFlagsToSet |= EInternalObjectFlags::Async; 
		}
}
```
