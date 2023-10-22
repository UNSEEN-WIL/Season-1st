## 블루프린트 가상 머신 구현

일찍이 우리는 가상 머신과 관련된 여러 단어에 대해 간략하게 소개했다. 다음, 우리는 언4의 블루프린트 가상 머신의 구현에 대해서 자세하게 설명할 것이다.

## Byte Code

가상머신의 바이트 코드는 `Script.h` 파일(언3에서는 `UnStack.h` 파일)에 있다. 아래 모든 리스트가 있다. 스크립팅 언어로 짜여져 있기 때문에, proxy-related code와 같이 특별한 바이트코드가 있을 수 있다. 물론, 일반적으로 사용되는 문장들도 있다. 할당, 조건없는 점프 명령어, 조건있는 점프 명령어, 스위치, 등등

```cpp
 // enum EExprToken 목록
enum EExprToken
{
	// Variable references.
	EX_LocalVariable		= 0x00,	// A local variable.
	EX_InstanceVariable		= 0x01,	// An object variable.
	EX_DefaultVariable		= 0x02, // Default variable for a class context.
	//						= 0x03,
	EX_Return				= 0x04,	// Return from function.
	//						= 0x05,
	EX_Jump					= 0x06,	// Goto a local address in code.
	EX_JumpIfNot			= 0x07,	// Goto if not expression.
	//						= 0x08,
	EX_Assert				= 0x09,	// Assertion.
	//						= 0x0A,
	EX_Nothing				= 0x0B,	// No operation.
	//						= 0x0C,
	//						= 0x0D,
	//						= 0x0E,
	EX_Let					= 0x0F,	// Assign an arbitrary size value to a variable.
	//						= 0x10,
	//						= 0x11,
	EX_ClassContext			= 0x12,	// Class default object context.
	EX_MetaCast             = 0x13, // Metaclass cast.
	EX_LetBool				= 0x14, // Let boolean variable.
	EX_EndParmValue			= 0x15,	// end of default value for optional function parameter
	EX_EndFunctionParms		= 0x16,	// End of function call parameters.
	EX_Self					= 0x17,	// Self object.
	EX_Skip					= 0x18,	// Skippable expression.
	EX_Context				= 0x19,	// Call a function through an object context.
	EX_Context_FailSilent	= 0x1A, // Call a function through an object context (can fail silently if the context is NULL; only generated for functions that don't have output or return values).
	EX_VirtualFunction		= 0x1B,	// A function call with parameters.
	EX_FinalFunction		= 0x1C,	// A prebound function call with parameters.
	EX_IntConst				= 0x1D,	// Int constant.
	EX_FloatConst			= 0x1E,	// Floating point constant.
	EX_StringConst			= 0x1F,	// String constant.
	EX_ObjectConst		    = 0x20,	// An object constant.
	EX_NameConst			= 0x21,	// A name constant.
	EX_RotationConst		= 0x22,	// A rotation constant.
	EX_VectorConst			= 0x23,	// A vector constant.
	EX_ByteConst			= 0x24,	// A byte constant.
	EX_IntZero				= 0x25,	// Zero.
	EX_IntOne				= 0x26,	// One.
	EX_True					= 0x27,	// Bool True.
	EX_False				= 0x28,	// Bool False.
	EX_TextConst			= 0x29, // FText constant
	EX_NoObject				= 0x2A,	// NoObject.
	EX_TransformConst		= 0x2B, // A transform constant
	EX_IntConstByte			= 0x2C,	// Int constant that requires 1 byte.
	EX_NoInterface			= 0x2D, // A null interface (similar to EX_NoObject, but for interfaces)
	EX_DynamicCast			= 0x2E,	// Safe dynamic class casting.
	EX_StructConst			= 0x2F, // An arbitrary UStruct constant
	EX_EndStructConst		= 0x30, // End of UStruct constant
	EX_SetArray				= 0x31, // Set the value of arbitrary array
	EX_EndArray				= 0x32,
	EX_PropertyConst		= 0x33, // FProperty constant.
	EX_UnicodeStringConst   = 0x34, // Unicode string constant.
	EX_Int64Const			= 0x35,	// 64-bit integer constant.
	EX_UInt64Const			= 0x36,	// 64-bit unsigned integer constant.
	//						= 0x37,
	EX_PrimitiveCast		= 0x38,	// A casting operator for primitives which reads the type as the subsequent byte
	EX_SetSet				= 0x39,
	EX_EndSet				= 0x3A,
	EX_SetMap				= 0x3B,
	EX_EndMap				= 0x3C,
	EX_SetConst				= 0x3D,
	EX_EndSetConst			= 0x3E,
	EX_MapConst				= 0x3F,
	EX_EndMapConst			= 0x40,
	//						= 0x41,
	EX_StructMemberContext	= 0x42, // Context expression to address a property within a struct
	EX_LetMulticastDelegate	= 0x43, // Assignment to a multi-cast delegate
	EX_LetDelegate			= 0x44, // Assignment to a delegate
	EX_LocalVirtualFunction	= 0x45, // Special instructions to quickly call a virtual function that we know is going to run only locally
	EX_LocalFinalFunction	= 0x46, // Special instructions to quickly call a final function that we know is going to run only locally
	//						= 0x47, // CST_ObjectToBool
	EX_LocalOutVariable		= 0x48, // local out (pass by reference) function parameter
	//						= 0x49, // CST_InterfaceToBool
	EX_DeprecatedOp4A		= 0x4A,
	EX_InstanceDelegate		= 0x4B,	// const reference to a delegate or normal function object
	EX_PushExecutionFlow	= 0x4C, // push an address on to the execution flow stack for future execution when a EX_PopExecutionFlow is executed.   Execution continues on normally and doesn't change to the pushed address.
	EX_PopExecutionFlow		= 0x4D, // continue execution at the last address previously pushed onto the execution flow stack.
	EX_ComputedJump			= 0x4E,	// Goto a local address in code, specified by an integer value.
	EX_PopExecutionFlowIfNot = 0x4F, // continue execution at the last address previously pushed onto the execution flow stack, if the condition is not true.
	EX_Breakpoint			= 0x50, // Breakpoint.  Only observed in the editor, otherwise it behaves like EX_Nothing.
	EX_InterfaceContext		= 0x51,	// Call a function through a native interface variable
	EX_ObjToInterfaceCast   = 0x52,	// Converting an object reference to native interface variable
	EX_EndOfScript			= 0x53, // Last byte in script code
	EX_CrossInterfaceCast	= 0x54, // Converting an interface variable reference to native interface variable
	EX_InterfaceToObjCast   = 0x55, // Converting an interface variable reference to an object
	//						= 0x56,
	//						= 0x57,
	//						= 0x58,
	//						= 0x59,
	EX_WireTracepoint		= 0x5A, // Trace point.  Only observed in the editor, otherwise it behaves like EX_Nothing.
	EX_SkipOffsetConst		= 0x5B, // A CodeSizeSkipOffset constant
	EX_AddMulticastDelegate = 0x5C, // Adds a delegate to a multicast delegate's targets
	EX_ClearMulticastDelegate = 0x5D, // Clears all delegates in a multicast target
	EX_Tracepoint			= 0x5E, // Trace point.  Only observed in the editor, otherwise it behaves like EX_Nothing.
	EX_LetObj				= 0x5F,	// assign to any object ref pointer
	EX_LetWeakObjPtr		= 0x60, // assign to a weak object pointer
	EX_BindDelegate			= 0x61, // bind object and name to delegate
	EX_RemoveMulticastDelegate = 0x62, // Remove a delegate from a multicast delegate's targets
	EX_CallMulticastDelegate = 0x63, // Call multicast delegate
	EX_LetValueOnPersistentFrame = 0x64,
	EX_ArrayConst			= 0x65,
	EX_EndArrayConst		= 0x66,
	EX_SoftObjectConst		= 0x67,
	EX_CallMath				= 0x68, // static pure function from on local call space
	EX_SwitchValue			= 0x69,
	EX_InstrumentationEvent	= 0x6A, // Instrumentation event
	EX_ArrayGetByRef		= 0x6B,
	EX_ClassSparseDataVariable = 0x6C, // Sparse data variable
	EX_FieldPathConst		= 0x6D,
	EX_Max					= 0x100,
};
```

## Stack Frame Details

`Stack.h` 파일에서 `FFrame`의 정의를 찾을 수 있다. 구조체로 정의되어 있지만 현재 코드를 실행하는 로직은 이 안에 은닉되어 있다. 데이터 멤버를 살펴보도록 하자.

```cpp
// Variables.
UFunction* Node;
UObject* Object;
uint8* Code;
uint8* Locals;

FProperty* MostRecentProperty;
uint8* MostRecentPropertyAddress;

/** The execution flow stack for compiled Kismet code */
FlowStackType FlowStack;

/** Previous frame on the stack */
FFrame* PreviousFrame;

/** contains information on any out parameters */
FOutParmRec* OutParms;

/** If a class is compiled in then this is set to the property chain for compiled-in functions. In that case, we follow the links to setup the args instead of executing by code. */
FField* PropertyChainForCompiledIn;

/** Currently executed native function */
UFunction* CurrentNativeFunction;

bool bArrayContextFailed;
```

여기서 스택 프레임이 스크립트가 실행하는 UObject, 현재 코드 실행 위치, 지역 변수, 이전 스택 프레임, 함수 콜에 의해 리턴되는 인자들, 현재 실행 중인 네이티브 함수 등을 볼 수 있다. 함수 호출의 리턴값은 함수 호출 전에 저장되고, 호출이 끝나면 복구된다. 다음과 같다.

```cpp
uint8* SaveCode = Stack.Code;

// Call function
...

Stack.Code = SaveCode 
```

다음은 FFrame 안에서 실행과 관련 있는 중요한 함수들이다.

```cpp
// Functions.
COREUOBJECT_API void Step( UObject* Context, RESULT_DECL );

/** Replacement for Step that uses an explicitly specified property to unpack arguments **/
COREUOBJECT_API void StepExplicitProperty(void*const Result, FProperty* Property);

/** Replacement for Step that checks the for byte code, and if none exists, then PropertyChainForCompiledIn is used. Also, makes an effort to verify that the params are in the correct order and the types are compatible. **/
template<class TProperty>
FORCEINLINE_DEBUGGABLE void StepCompiledIn(void* Result);
FORCEINLINE_DEBUGGABLE void StepCompiledIn(void* Result, const FFieldClass* ExpectedPropertyType);

/** Replacement for Step that checks the for byte code, and if none exists, then PropertyChainForCompiledIn is used. Also, makes an effort to verify that the params are in the correct order and the types are compatible. **/
template<class TProperty, typename TNativeType>
FORCEINLINE_DEBUGGABLE TNativeType& StepCompiledInRef(void*const TemporaryBuffer);

COREUOBJECT_API virtual void Serialize( const TCHAR* V, ELogVerbosity::Type Verbosity, const class FName& Category ) override;

COREUOBJECT_API static void KismetExecutionMessage(const TCHAR* Message, ELogVerbosity::Type Verbosity, FName WarningId = FName());

/** Returns the current script op code */
const uint8 PeekCode() const { return *Code; }

/** Skips over the number of op codes specified by NumOps */
void SkipCode(const int32 NumOps) { Code += NumOps; }

template<typename TNumericType>
TNumericType ReadInt();
float ReadFloat();
FName ReadName();
UObject* ReadObject();
int32 ReadWord();
FProperty* ReadProperty();

/** May return null */
FProperty* ReadPropertyUnchecked();

/**
 * Reads a value from the bytestream, which represents the number of bytes to advance
 * the code pointer for certain expressions.
 *
 * @param	ExpressionField		receives a pointer to the field representing the expression; used by various execs
 *								to drive VM logic
 */
CodeSkipSizeType ReadCodeSkipCount();

/**
 * Reads a value from the bytestream which represents the number of bytes that should be zero'd out if a NULL context
 * is encountered
 *
 * @param	ExpressionField		receives a pointer to the field representing the expression; used by various execs
 *								to drive VM logic
 */
VariableSizeType ReadVariableSize(FProperty** ExpressionField);
```

`ReadInt(), ReadFloat(), ReadObject()`와 같은 함수들은 이름을 보면 무슨 일을 하는 지 알 수 있다. 상응하는 정수, 소수, 오브젝트 등을 읽는 역할을 한다. 여기서는 `Step()` 함수에 대해 주로 이야기한다. 코드는 다음과 같다.

```cpp
void FFrame::Step(UObject* Context, RESULT_DECL)
{
	int32 B = *Code++;
	(Context->*GNatives[B])(*this, RESULT_PARAM);
}
```

메인 함수에서 명령어를 가져오고, 네이티브 함수 배열에서 명령어에 상응하는 함수를 실행하기 위해 찾는 것임을 알 수 있다.

## 바이트 코드에 상응하는 함수

일찍이 우리는 가상 머신의 모든 바이트코드를 나열하고, 코드와 상응하는 각 바이트 코드의 정확한 실행부(언4에서는 `ScriptCore.cpp`, 언3에서는 `UnScript.h`)에서 각 바이트코드가 `GNatives`와 `GCasts`와 연결되는 것을 볼 수 있다.

다음과 같이 함수 정의가 이루어진다.

```cpp
// 인자 2개를 받는다는 키워드 typedef
typedef void (UObject::*Native) (FFrame& TheStack, RESULT_DECL);

// 언리얼스크립트 내부적으로 쓰이는 리턴값 정의
#define RESULT_DECL void* const Result

// 한 언리얼 오브젝트 내에 다양한 함수를 담는 함수 테이블
extern Native GNatives[];
extern Native GCasts[];
```

이런 방식으로, 바이트 코드는 각 네이티브 함수마다 registration method를 호출하는데, 이는 `IMPLEMENT_VM_FUNCTION`과 `IMPLEMENT_CAST_FUNCTION` 매크로로 구현되어 있다.

다음은 매크로의 구현 형식이다.

```cpp
// UDK - 인자를 3개 받는 함수
#define IMPLEMENMT_FUNCTION(cls, num, func) \
	extern "C" { Native int##cls##func = (Native)&cls::func; } \
	static BYTE cls##func##Temp = GRegisterNative(num, int##cls##func);

---

// UE4 - 인자 1,2개만 받음
#define IMPLEMENT_FUNCTION(func) \
	static FNativeFunctionRegistrar UObject##func##Registar(UObject::StaticClass(),#func,&UObject::func);

#define IMPLEMENT_CAST_FUNCTION(CastIndex, func) \
	IMPLEMENT_FUNCTION(func); \
	static uint8 UObject##func##CastTemp = GRegisterCast( CastIndex, &UObject::func );

#define IMPLEMENT_VM_FUNCTION(BytecodeIndex, func) \
	STORE_INSTRUCTION_NAME(BytecodeIndex) \
	IMPLEMENT_FUNCTION(func) \
	static uint8 UObject##func##BytecodeTemp = GRegisterNative( BytecodeIndex, &UObject::func );
```

전역 정적 오브젝트를 정의하므로, 프로그램의 main 함수가 실행되기 이전에 이 함수는 배열에 상응하는 위치에 놓이게 될 것이다.

(매크로 읽는게 어려워서 도움을 좀 받아야 할 것 같다)

### 구현 과정

일찍이 블루프린트에 대해 이야기했고, 어떻게 블루프린트가 C++과 상호작용 하는지(C++에서 블루프린트 호출 혹은 블루프린트에서 C++ 호출) 이야기해보자.

### C++에서 블루프린트 함수 호출

```cpp
UFUNCTION(BlueprintImplementableEvent, Category = "TestGameMode")
void ImplementableFuncTest();

void TestGameMode::ImplementableFuncTest()
{
	ProcessEvent(FindFunctionChecked(TEST_ImplementableFuncTest), NULL);
}
```

이 함수는 인자가 없기 때문에, NULL이 모든 `ProcessEvents`에 들어간다. 만약 인자와 리턴값이 있다면, UHT가 자동으로 리턴값과 인자를 저장할 구조체를 생성해서, 블루프린트 함수가 C++에서 불렸을 때 UHT가 `TEST_ImplementableFuncTest` 이름에 상응하는 블루프린트 UFunction을 찾아서 추가적인 과정을 위해 `ProcessEvent`를 실행하게 된다. 

### ProcessEvent 과정

![Untitled](https://github.com/UNSEEN-WIL/Season-1st/assets/103979407/9639f298-2d13-4c8f-b769-4a77397205dd)

중간의 `UberFunction`은 쉐이더에서 쓰이는 함수라서 지금은 크게 관련이 없다.

### 블루프린트에서 C++ 함수 부르기

```cpp
UFUNCTION(BlueprintCallable, Category = "TestGameMode";
void CallableFuncTest;

DECLARE_FUNCTION(execCallableFuncTest)
{
	P_FINISH;
	P_NATIVE_BEGIN;
	this->CallableFuncTest()l
	P_NATIVER_END
}
```

만약 블루프린트에서 C++ 함수를 호출한다면, UHT가 상단의 코드를 생성하고, 만약 인자가 있으면 상응하는 인자를 담는다. 리턴값이 있으면 리턴값 또한 할당된다.
