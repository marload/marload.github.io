# Risks of Linux Capabilities and Privileged Containers

우리가 친숙한 도커 커맨드를 실행할 때는 가끔 `--privileged` 옵션을 사용하는 경우를 보았을 것이다. 하지만 이 옵션은 보안상 위험도가 매우 높고 사용을 자제하는 것을 권장한다. 이 privileged container, 즉 특권 컨테이너의 위험성을 알아본다.

## 리눅스의 Capability

특권 컨테이너에 대해서 다루기 전에 우선 리눅스의 능력(Capability)에 대해서 살펴봐야 합니다. capability를 적절히 대체할 수 있는 번역어가 없어서 이 글에서는 능력이라고 표현하겠습니다.

리눅스에 친숙한 분들이라면 루트 유저와 그 유저가 어떤 권한들을 가지고 있는지 알고 있을 것입니다. 루트는 리눅스 시스템에서 가장 강력한 권한을 가진 사용자로서, 리눅스에서 우리가 할 수 있는 대부분의 것들을 할 수 있습니다. 하지만 이렇게 넓은 권한을 가진 사용자로 무언가를 실행하는 것은 매우 많은 위험을 수반합니다. 이를 보완하기 위하여 리눅스는 능력(Capability)라는 시스템을 도입하였습니다.

리눅스는 능력이라는 기능을 통해 시스템의 특권을 세분화하고, 그 특권을 좁은 범위로 제한하는 목적을 가집니다. 이는 일반적으로 프로세스 단위에 할당되며 프로세스가 어떤 능력을 가지고 시스템에서 동작을 할 수 있는지를 제한하는 역할을 수행합니다. 이를 통해 프로세스가 필요로 하는 최소한의 권한만을 가질 수 있도록 하여, 시스템의 보안을 강화하는 목적으로 사용됩니다. 리눅스의 능력을 소개하기 위해서 보통 `CAP_NET_BIND_SERVICE`라는 능력을 대표적인 예제로 사용합니다.

일반적으로 아시는 것과 같이 1024번 보다 작은 포트는 오직 루트로만 소켓으로 열 수 있다고 아시는 분들이 꽤 있습니다. 하지만 엄밀히 따지면 루트라서 1024보다 낮은 포트를 열 수 있는 게 아니라 루트가 가진 `CAP_NET_BIND_SERVICE`라는 능력이 포트에 대한 소켓을 열 수 있게 하는 권한을 가질 수 있게 하는 것입니다.

`CAP_NET_BIND_SERVICE` 외에도 리눅스에는 여러 능력들이 있습니다. 이들 중 일부는 파일 시스템, 네트워크, 시스템 관리 등과 같은 여러 리눅스 전반의 기능을 활용하는 권한을 부여하게 됩니다. 각 능력은 일반적으로는 프로세스에 할당시키며, 그 프로세스가 리눅스 시스템에 대해서 사용할 수 있는 능력을 제한하고 허가할 수 있습니다.

따라서 리눅스에서 이 능력을 도입하는 것은 일반적인 보안 콘텍스트들에서 부르는 "최소 권한 원칙"을 구현하기 위한 기능들입니다. `CAP_NET_BIND_SERVICE` 같이 얼핏 보기에는 보안적으로 크게 위험하지 않아 보일 수 있는 권한들도 있지만, `CAP_SYS_ADMIN` 같이 하나의 능력만으로도 굉장히 넓은 범위의 권한을 가지며 매우 위험하게 사용될 수 있는 능력도 존재합니다.

## 특권 컨테이너

가끔 우리는 도커 커맨드로 `--privileged` 옵션을 사용합니다. 이 옵션을 통해 생성된 컨테이너는 이른바 특권 컨테이너라고 부릅니다. 아시다시피 컨테이너는 VM과 다르게 운영체제에서 구동되는 하나의 프로세스입니다. 그리고 리눅스 능력은 프로세스에 부여되는 기능이며, 이 특권 컨테이너는 위에서 살펴본 특권 중 대다수를 컨테이너 프로세스에 부여하여 동작하게 됩니다.

특권 컨테이너는 위험하다고 말씀드렸습니다. 그럼 이렇게 위험한 특권 컨테이너 옵션은 왜 제공되는 것일까요? 당연히 필요에 의해서 존재하고, 컨테이너에서 호스트에 대한 여러 시스템 컨트롤을 필요로 하기에 필요합니다. 특히 DID(Docker in Docker)를 위해서 매우 유용하게 활용됩니다. DID는 컨테이너 안에서 컨테이너를 실행하는 것을 의미하는데, 예를 들어 깃허브 액션을 자체 관리형으로 운영한다고 했을 때 하나의 액션마다 하나의 컨테이너로 동작하도록 만들었다고 가정해 봅시다. 그런데 이 액션 안에서 컨테이너 이미지를 빌드하고 실행하는 기능이 포함될 수 있습니다. 기본적으로 도커 컨테이너는 컨테이너 안에서 컨테이너를 실행할 수 없기에 특권 컨테이너로 실행하여 앞서 말한 기능들을 수행할 수 있게 만들어줘야 할 필요가 있습니다.

한번 특권 컨테이너와 일반 컨테이너를 직접 생성하여 어떤 능력들이 부여되어 있는지 비교해 보겠습니다. `capsh --print` 명령을 통해서 리눅스 특권을 확인해 볼 수 있습니다.


```plaintext
Bounding set
=cap_chown, cap_dac_override,cap_dac_read_search, cap_fowner, cap
_fsetid,cap_kill,cap_setgid,cap_setuid,cap_setpcap,cap_linux_i
mmutable, cap_net_bind_service,cap_net_broadcast,cap_net_admin,
cap_net_raw,cap_ipc_lock,cap_ipc_owner,cap_sys_module,cap_sys_
rawio,cap_sys_chroot,cap_sys_ptrace,cap_sys_pacct,cap_sys_admi
n,cap_sys_boot,cap_sys_nice,cap_sys_resource,cap_sys_time, cap_
sys_tty_config,cap_mknod, cap_lease, cap_audit_write, cap_audit_c
ontrol,cap_setfcap,cap_mac_override,cap_mac_admin,cap_syslog,c
ap_wake_alarm,cap_block_suspend, cap_audit_read
```

```plaintext
Bounding set
=cap_chown, cap_dac_override,cap_fowner,cap_fsetid,cap_kill,cap
_setgid,cap_setuid,cap_setpcap,cap_net_bind_service,cap_net_ra
w, cap_sys_chroot,cap_mknod,cap_audit_write,cap_setfcap
```


Bounding set은 해당 시스템이 사용할 수 있는 최대치의 능력 리스트입니다. 한눈에 봐도 위쪽의 특권 컨테이너는 상당히 많은 능력을 가지고 있고, 일반 컨테이너는 그보다 적은 능력을 가지고 있습니다.

## 특권 컨테이너의 위험성

보시는 것과 같이 특권 컨테이너는 아주 많은 리눅스 능력을 가지고 있습니다. 이는 호스트 시스템에 있는 거의 모든 리소스에 접근할 수 있다는 의미입니다. 예를 들어 CAP_SYS_ADMIN 능력은 호스트 시스템의 네임스페이스나 cgroup을 컨트롤할 수 있는 권한을 가집니다. 그리고 여러분들이 아시다시피 컨테이너는 리눅스 네임스페이스와 cgroup으로 구현되어 있으며, 이는 곧 컨테이너의 핵심 기능인 격리를 무력화시킬 수 있다는 것을 의미합니다.

이런 많은 능력들로 인해 여러 보안 이슈가 아주 쉽게 발생할 수 있습니다. 해커가 컨테이너에 대한 접근 권한을 이미 가지고 있다면 호스트 시스템의 거의 모든 리소스에 접근할 수 있다는 것을 의미합니다. 이는 호스트의 파일 시스템을 마운트 하거나, 호스트 커널 모듈을 직접 로드하거나, 시스템 콜을 조작하는 등의 작업을 수행할 수 있는 등의 아주 심각한 보안 위험을 초래할 수 있다는 것을 의미합니다.

특히 대표적인 컨테이너상 보안 이슈인 컨테이너 탈출(container escape) 문제가 발생하기 쉽습니다. 실제로 구글에 "privileged container escape"라고 검색만 해도 특권 컨테이너에서 호스트 시스템으로 탈출하는 익스플로잇을 설명한 아주 많은 글을 찾을 수 있습니다. 구체적인 익스플로잇 방법은 이 글의 주제와 조금은 떨어져 있기에 관심 있으실 만한 분들을 위해 몇 가지 글을 링크해 드리겠습니다.

- [Understanding Docker container escapes](https://blog.trailofbits.com/2019/07/19/understanding-docker-container-escapes/)
- [Privileged Containers Aren't Containers](https://ericchiang.github.io/post/privileged-containers/)
- [Docker Breakout / Privilege Escalation](https://github.com/HackTricks-wiki/hacktricks/blob/master/linux-hardening/privilege-escalation/docker-security/docker-breakout-privilege-escalation/README.md)

위 글들을 얼핏 살펴보시는 것만으로도 아실 수 있겠지만, 몇 가지 조건만 충족된다면 정말 매우 쉽게 컨테이너를 탈출할 수 있음을 보여줍니다.

## 특권이 필요한 상황에서의 대안

당연히 필요에 따라 특권이 필요할 수 있습니다. 사실 엄밀히 말하자면 대부분의 상황은 특권이 아니라 몇몇 능력이 필요할 수 있습니다. 즉, 특권 컨테이너를 통해서 수많은 능력을 부여하는 방법이 아닌 최소한으로 필요한 몇몇 능력만 부여함으로 이런 문제를 상당히 많이 필할 수 있습니다. 가장 좋은 방법으로는 컨테이너가 실행될 때 정말 필요한 능력만을 부여하는 것입니다.

```bash
docker run --cap-add CAP_NET_ADMIN ubuntu bash
```

위와 같이 --cap-add 옵션을 통해 필요한 능력을 컨테이너에 따로 부여할 수 있습니다. 하지만 모든 컨테이너마다 능력을 트레이싱하여 필요한 능력을 찾는 것은 복잡한 작업이 될 수 있으므로, 비교적 단순한 방법으로 AppArmor나 SELinux를 활용하는 것도 도움이 될 수 있습니다. 보통 이런 시스템의 기본 프로파일은 일반적이지 않은 접근 패턴을 차단하도록 구성되어 있어 기본 설정만으로도 상당히 많은 위험을 줄일 수 있습니다. 실제로 도커의 경우 컨테이너를 실행 시 AppArmor가 설정된 상태로 실행됩니다.