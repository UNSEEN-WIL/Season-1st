# 언리얼 네트워크
이번에 언리얼 네트워크에 대해 공부해야 할 기회가 있어 간단하게 공부를 하게 되었습니다. 

## 전반적인 요약
전체적으로 NetDriver->Connection->Channel의 흐름으로 멀티플레이가 이루어지게 됩니다.

언리얼 엔진에서는 `UGameEngine`이 먼저 로딩이 된 다음 `LoadMap`을 통해 `NetDriver`를 받아와서 초기 설정을 하게 됩니다. 그런 다음 클라이언트와 서버의 연결을 `NetDriver`가 책임져 줍니다.

이후 연결이 되고 난 후 `Connection`이 생성이 되게 되며 드라이버는 이 `Connection`에 대한 정보를 계속 가지게 됩니다.
- 서버는 연결된 클라이언트들에 해당하는 `Connection` 배열
- 클라이언트는 서버 단일에 대한 `Connection` 변수 한개

를 소유하게 되며 배열의 원소 유무로 클라이언트인지 서버인지도 알 수 있게 됩니다.

이후 레벨의 Rick이 돌게 되면 `NetDriver::RickDispatch`를 호출하게 됩니다. (클라이언트의 경우 `TickNetClient`)
이런 과정에서 `UWorld::Tick` –> `NetDriver::TickFlush()` -> `UNetConnections::Tick` -> `UChannel::Tick`이 불리게 됩니다.

결론적으로는 Tick을 통해 확인을 하게 됩니다. 서버는 보낼 것이 있는지 확인하게 되고 클라이언트는 받을 것이 있는지 확인하게 됩니다.

## 액터의 리플리케이트

리플리케이트는 '서버 -> 클라이언트'방향으로 진행되기에 서버 먼저 보게 됩니다.

### 서버의 흐름

- 서버에서 `UWorld::ServerTickClients`가 불리면서 `UActorChannel::ReplicateActor`가 불리게 됩니다.
- 이후 액터의 정보들을 Bunch에 넣은 뒤 `UChannel::SendBunch`를 통해 전송이 됩니다.
- `SendBunch` 에서는 받은 `Bunch`를 `UNetConnection::SendRawBunch`로 주게 됩니다.
- `UNetConnection::SendRawBunch` 에서는 받은 `Bunch`를 자신의 필드인 `Out (BitWriter)`에게 넣어준 뒤 `UNetConnection::PostSend`를 호출합니다.
- 이후 `UNetConnection::FlushNet`을 통해서 `Out` 필드에 있는 것들이 `UNetConnection::
LowLevelSend`로 보내지게 됩니다.
	- 위 `LowLevelSend`함수는 `UNetConnection`에서 구현이 되어 있지 않음
	- `UTcpipConnection::LowLevelSend`에서 RemoteAddr을 통해 `Socket->SendTo`를 보냄
	

### 클라이언트의 흐름

- `UTcpipConnection::TickDispatch`에서 `Socket->RecvFrom`을 함
- 이후 `UNetConnection::ReceivedRawPacket`에 받은 데이터를 전달
- 이후 데이터를 `FBitReader`로 만들어 `UNetConnection::ReceivedPacket`에 전달
- 이거를 여차저차 Bunch로 만들어서 `UChannel::ReceivedRawBunch`로 전달
- 또 이를 `UChannel::ReceivedSequencedBunch`에 전달
- 이후 `UChannel::ReceivedBunch`에 전달
	- `UChannel의 ReceivedBunch`는 순수 가상 함수
	- ActorChannel로 할당된다면 클래스 정보를 가져와 리플리케이트, 노티파이 (ReplicatedEvent) 순으로 함수 콜을 보게 됨


## RPC의 경우

- `CallFunction -> ProcessRemoteFunction -> InternalProcessRemoteFunction`을 통해 Bunch가 되고 `UChannel::SendBunch`를 통해 보내진다.
	- (UE3) RPC에 전달된 파라미터의 경우 `optional` 인지 등을 체크해 파라미터를 넣어줍니다.
	- `InternalProcessRemoteFunction`안에서 reliability를 체크해 Bunch에 넣어 줍니다

- 수신에 있어서 `UActorChannel::ReceivedBunch`에서 받게 됩니다.
- 이 중 캐싱한 FieldCache의 Field가 UFunction으로 변환이 된다면 이를 RPC로 판단하게 됩니다.
- 이후 Actor Channel에서 `Actor::ProcessEvent`를 호출해 인자와 함께 넣어주게 됩니다.
- ProcessEvent를 호출한 다음 받은 인자들을 삭제하는 작업을 거칩니다.


결국 수신에 있어서는 리플리케이트와 방법이 같게 되지만 발신에 있어서 RPC는 **Call이 있을 때 SendBunch를 하기 때문**에 리플리케이트보다 더 즉각적으로 받을 수 있게 되는 것입니다.


## 여러 궁금증
### UDP 인가 TCP 인가
기본적으로 UDP 기반이라고 알려져 있지만 Ack를 처리하거나 TcpipConnection이 정의되어 있는 모습도 있었습니다. 다만 Socket->SendTo를 할 때 UDPEncryption으로 패킷을 만드는 작업으로 UDP임을 확인할 수 있습니다.

### 신뢰성 확보는 어디서 하는가
- bunch 에서 bReliable 라는 부분이 있음
- `ReceivedRawBunch`에서 `reliable`하다면 checkslow 를 통해 이전에 받지 못한 bunch들을 기다린다는 내용이 있음
- 그리고 `Bunch.ChSequence > Connection->InReliable[ChIndex]` 에서 이를 체크 하는 것으로 보인다
- `Bunch.ChSequence`는 `UChannel::SendBunch`에서 `++Connection->OutReliable[ChIndex]` 로 해주는걸 볼 수 있습니다.
- 또한 `UNetConnection::ReceivedPacket`에서도 패킷을 Bunch로 풀 때 reliable 여부에 따라 `Bunch.ChSequence`를 설정해주는걸 볼 수 있습니다.
