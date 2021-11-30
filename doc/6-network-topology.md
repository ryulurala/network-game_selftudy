---
title: "네트워크 게임 토폴로지"
---

## 클라이언트-서버

- n 개의 클라이언트가 접속하면 총 O(2n)개의 연결이 존재(비대칭적)
- 서버의 대역폭을 확보해야 함
- 물론, 클라이언트도 많은 클라이언트가 접속한다면 월드 내에 리플리케이션해야 하는 객체 수가 많아지므로, 각 클라이언트의 대역폭도 증가하게 된다.

- 대부분은 권한 집중형으로 서버를 구현

  - 각 클라이언트는 시뮬레이션을 독자적으로 돌리는 대신 그것이 올바른지 판단하는 것은 서버가 한다.
  - 만약 올바르지 않다면 게임 상태를 갱신해야 한다.

- 서버에 심판 권한을 부여하기 때문에 클라이언트가 수행하려는 액션에 약간의 랙 or 지연이 발생하는 것을 피할 수 없다.

- RTT가 100ms 이하면 나쁘지 않는다고 본다.

### 예제 게임: 로보캣 액션 게임(CS)

- 각 플레이어가 조종하는 고양이는 실뭉치를 던질 수 있지만, 권한 집중형 서버 모델에선 실뭉치가 다른 플레이어에 명중하는지 클라이언트 스스로 판단하지 않는다.
- 대신 클라이언트가 실뭉치를 던지고 싶다고 서버에 알려주고, 서버는 먼저 그 클라이언트가 실뭉치를 던질 수 있는지 판단하고, 실제로 다른 플레이어에 명중하는지 아닌지에 대한 판단도 서버가 한다.

- 중요한 점

  - 서버에서 돌아가는 코드가 클라이언트에서 돌아가는 코드와는 다르게 해야 함.

- 입장 과정

  - 클라이언트
    - "hello" 패킷을 보냄(플레이어 이름도 포함)
    - 클라이언트가 "welcome" 패킷을 받으면 Player ID를 저장하고 리플리케이션 정보를 서버와 주고받기 시작함.
  - 서버
    - "hello" 패킷을 받으면 Player ID를 해당 플레이어에게 할당하고 따로 기억해둠. 그리고 "welcome" 패킷을 클라이언트에게 보낸다.

- 만약, 패킷이 손실되더라도 단순히 중복 전송해 손실을 극복한다. 물론, 서버는 받았는데도 또 받으면 무시해야 하는 처리가 필요하다.

- Client State

  - Uninitialized
  - SayingHello
    - 네트워크 관련 상태가 초기화된 상태
    - 계속해서 헬로 패킷을 보내는 상태
  - Welcomed
    - 웰컴 패킷을 받은 상태
    - 이제 입력 패킷을 보냄

- 입력 공유 과정
  - 클라이언트
    - 클라이언트는 매 프레임마다 자신의 입력 이벤트를 처리
    - InputManager가 매 프레임마다 클라이언트의 입력을 스냅샷(snapshot), 당시 상태대로 남겨둠.
    - 입력 내용 중 서버에서 처리할 것은 그 이벤트를 서버에 보냄.
      - ex) 고양이 움직임, 실뭉치 던지기
  - 서버
    - 서버는 클라이언트가 보낸 입력 패킷을 받아서 클라이언트 프록시에 저장하고 상태를 추적
    - 개별 클라이언트의 상태를 추적하고, 프록시에 저장된 입력 정보를 토대로 서버가 시뮬레이션을 갱신하고 각 객체의 행동을 결정
    - 클라이언트 모두에게 리플리케이션 패킷을 보내는 경우가 아니므로 각 클라이언트에 대응되는 리플리케이션 관리자를 둔다.
    - 서버 프레임 기준이 아닌 클라이언트의 이동 조작 데이터에서 가져온다.
      - 가능한 클라이언트 조작을 있는 그대로 재현해 시뮬레이션한다.
      - 서버와 클라이언트가 다른 프레임 주기로 동작하더라도 문제없이 처리 가능
      - 대신, 물리를 시뮬레이션 하기 힘듦

### CS 구조 서버의 종류

- 전용 서버(Dedicated server)
  - 서버 프로세스는 클라이언트 프로세스와 완전히 분리된다.(Headless)
  - ex) 배틀그라운드, 오버워치 등
- 리스닝 서버(Listen server)
  - 플레이어 자신이 서버를 띄운 머신을 가지고 클라이언트로 직접 게임에도 참여하는 방식
  - Peer hosting 방식
  - ex) 스타크래프트 LAN 플레이 방식, 서든어택(?)
  - Host Migration
    - 리스닝 서버의 연결이 끊어지면 클라이언트 중 하나가 새로 서버 역할을 맡는다.
    - P2P 방식을 섞어 하이브리드 토폴로지로 구현해야 함.

## 피어-투-피어(P2P)

- 게임에 참여하는 머신이 동일 게임에 참여하는 다른 모든 머신과 서로 연결됨.
- P2P 게임에선 판정 권한이 어디에 있는지가 불분명하다.
- 모든 피어가 다른 피어와 직접 통신하기 때문에 CS 구조보다는 레이턴시가 줄어든다.
- 모든 피어의 동기화를 맞추기가 까다롭다.
- 에이지 오브 엠파이어의 경우,

  - 게임 진행은 각 200ms의 턴으로 잘게 나누어진다.
  - 200ms 동안 입력된 명령들은 대기열에 쌓이고, 이 시간이 끝나면 모든 피어에 대기열의 전체 명령이 전송된다.
  - 예를 들어, 제 1턴의 결과를 화면에 표시할 때 대기열에 쌓인 명령들은 한 턴 건너서 제 3턴에 실행된다.

- 따져야 되는 중요한 점
  - 게임 상태가 모든 피어 사이에선 일관되어야 한다.(완전히 결정론적으로 구현할 필요가 있음)
  - 피어 간 게임 상태의 일관성 검사를 위해 체크섬을 도입 or 난수 발생기를 동기화하는 등
  - 신규 플레이어가 참가하려 할 때 어디에 접속해야 하는가
  - 피어 중 하나의 연결이 끊어지면 해당 피어가 제거될 동안 잠시 게임을 멈춤, 접속 해제가 완전히 처리되면 남은 피어들이 계속 시뮬레이션 진행.

### 예제 게임: 로보캣 액션 게임(P2P)

- NAT 투과

  - 랑데뷰 서버(Rendezvous server)
    - 피어가 다른 피어에 처음 접속을 맺는 것을 도와줌.
    - 랑데뷰 서버를 거쳐 다른 피어와 연결되므로, 각자의 공인 IP 주소로 다른 피어와 연결됨.
  - 릴레이 서버(Relay server)
    - 중앙에 서버를 두고 트래픽이 중앙에 집중되었다가 각 피어로 분배됨.
    - 다른 피어의 IP 주소를 알 필요가 없다.
    - 보안 면에서도 피어가 다른 피어에 DDos 공격을 해 먹통으로 만드는 행위를 막을 수 있다.

- 명령과 입력을 구분: 락스텝 턴(1턴 당 100ms)

  - 고양이를 클릭한 것은 입력이긴 하지만 그 선택 자체로 게임에 어떤 영향을 끼치진 않는다.
  - 만약, 이동하거나 공격하는 입력이 들어가면 게임의 상태를 변화시키므로 각각 명령으로 생성해 처리한다.

  - 명령도 곧바로 실행되지 않는다. 즉시 처리하는 대신 그 턴 동안 명령을 수집하고 대기열에 넣어둔다. 그리고 턴이 끝나면 피어는 자신이 모아둔 명령 리스트를 다른 모든 피어에 전달한다.
  - 턴 x에 내려진 명령은 x+2가 되어야 실행된다.(레이턴시가 200ms가 항상 발생)

- 동기화
  - Random()
    - 각 피어가 특정 턴에 난수 발생기가 특정 숫자를 생성하게 규칙을 통일하면 여러 피어가 항상 같은 결과를 얻게 됨.
    - 마스터 피어가 카운트 다운을 시작할 때 난수 값을 생성해서 이를 새 시드 값으로 공유
    - C 표준의 rand, srand 는 동기화에 적합하지 않다. 어떤 유사 난수 발생 알고리즘을 쓰는지도 명시하고 있지 않다.
    - C++ 11에 이르고 메르센 트위스터 알고리즘을 쓰고 있어 게임 용도로 충분한 유사 난수 발생기이다.
  - 부동 소수점
    - 연산이 하드웨어 구현에 따라 다른 결과가 나오기도 함.
  - 체크섬(Checksum)
    - 턴 패킷에 체크섬을 실어 보냄.
    - 모든 게임 인스턴스가 턴 종료 시에 서로 계산한 값이 모두 일치하는 결과에 도달하는지 검사
  - 만약, 난수 + 체크섬이 불일치해 동기화가 깨지면 종료된다.
    - 다른 방법으로는 투표 식으로 누구는 맞고 누구는 틀리면 틀린 누구만 나가도록 하는 경우도 있다.
    - 또, 전체 게임 상태를 리플리케이션해 게임의 동기화를 다시 맞추는 경우도 있다.(동기화가 깨졌다고 플레이어를 내쫓아서는 안되는 종류의 게임이라면 염두해볼 필요)