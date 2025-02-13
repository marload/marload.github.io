
# When Ztunnel Dies

최근 Istio는 오랫동안 `Service Mesh = Sidecar`라는 관념을 넘어 Ambient Mesh를 출시했다. 이 Ambient Mesh에서 가장 중요한 컴포넌트를 하나만 꼽으라고 한다면 단연 Ztunnel을 꼽을 것이다.

Ztunnel은 기존 사이드카 프록시 모델과는 조금은 다른 특징을 가지는데, 기존 사이드카 프록에서는 각 애플리케이션 파드에 Envoy가 사이드카 패턴으로 함께 배치가 되어있었지만 Ztunnel은 노드 단위로 Daemonset으로 배치되어 하나의 프로세스가 전체 노드의 L4 트래픽을 제어한다. 그래서 자원 효율성이 증가한다. 그런데 이제 모델 자체가 바뀌다보니 프록시 자체의 업데이트나 재시작 과정에서 발생할 수 있는 트래픽 분산이나 커넥션을 유지하는 부분에서 또 다른 이슈가 발생할 수 있다.

## Ztunnel의 업그레이드와 종료 절차

새로운 Ztunnel 데몬셋이 생겨날 때는 기존 시스템과 같이 운용되는 것을 전제로 한다. 이 때 새로운 데몬셋 기반의 시스템이 완전히 준비되기 전까지는 기존의 사이드카 시스템 혹은 데몬셋 기반의 이전 버전 Ztunnel이 계속해서 모든 트래픽을 처리한다. 새로운 Ztunnel이 부팅되고 초기화 과정을 거치면 시스템은 점진적으로 트래픽을 분산하여 넘긴다. 이 과정에서 SO_REUSEPORT와 같은 커널 기능을 사용해서 동일한 포트에서 여러 프로세스가 동시에 요청을 수신할 수 있도록 해서 두 시스템 사이에 트래픽을 동적으로 재분배한다.

**새 Ztunnel 시작**

업그레이드가 시작되면 현재 실행중인 기존 Ztunnel 인스턴스하고 같은 노드에서 새 버전의 Ztunnel이 뜨게 된다. 이 새로운 Ztunnel은 초기화를 시작하고 구성파일을 로드하고 자체 네트워크 소켓을 설정한다. 이후 새 Ztunnel 파드와 기존 파드 모두 SO_REUSEPORT 옵션을 사용해서 동일한 포트에 바인딩 된다. 참고로 SO_REUSEPORT는 리눅스 커널의 옵션으로 하나의 포트에 여러 프로세스를 바인딩하게 해주는 옵션이다.

**준비 상태 확인**

추가로 더 진행되기 전에 새 인스턴스는 여러 헬스체크나 Readiness Probe를 거치게 된다. 이 테스트에서는 새 인스턴스가 완전히 초기화되고 설정을 로드하고 실제로 트래픽을 처리할 수 있는지 확인하는 과정이며, 모든 검사가 통과되어야지 새로운 Ztunnel이 운영 가능한 상태라고 판단할 수 있다.

**기존 Ztunnel 연결 종료**

이제 새로운 Ztunnel이 잘 동작한다는 것이 확인되었다면, 기존 파드는 새로운 연결 수신을 받지 못하도록 만든다. 이 시점에서 두 Ztunnel 모두 활서상태이기는 기존 Ztunnel은 더이상 신규 요청을 받지 못하고 새로운 요청은 모두 신규 노드가 받기 시작한다. 그리고 위에서 설명한 SOREUSEPORT를 사용하여 하나의 포트에서 두 프로세스가 동작하고 있기 때문에, 신규 연결은 동적으로 새로운 Ztunnel로 라우팅 시키기가 쉽다. 단순히 기존 Ztunnel의 Listener가 신규 트래픽을 받지 않도록 비활성화 되기만 하면 된다.

**기존 연결 관리**

여기서 문제 발생하는건 TCP는 Graceful Shutdown을 할 수 있는 방법이 따로 없다는 것이다. L7에서는 여러 방식들이 있어서 서로 종료 합이를 할 수 있는데, TCP는 그런게 없다. 즉, Ztunnel에서는 클라이언트에게 이제 커넥션을 종료하자는 신호를 보낼 수 없다는 것이다. 그렇기에 선택한 방법은 그냥 종료 안하는거다. 사실 선택지가 없는거긴한데, 그냥 종료 안하고 자연스럽게 종료될 수 있도록 기다리는거다. 그리고 이 기다리는 시간을 제어하기 위해서 termination grace period를 지정하고, 그 시간만큼 대기한다. 그리고 이 시간이 만료되도 종료되지 않는다면 강제 종료하고 Connection Reset을 걍 때려버린다.