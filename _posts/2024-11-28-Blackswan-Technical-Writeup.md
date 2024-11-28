---
layout: post
title: "Blackswan Technical Writeup 이해하기"
date: 2024-11-28 10:45:13
category: "Tech"
---



## Windows Kernel IO
사용자 모드의 응용프로그램은 시스템 콜을 이용하여 커널과 interaction 한다. 일반프로그램은 보통 API 들을 이용하지만 직접 시스템 콜을 할 수도 있다. 대부분의 시스템 콜은 인자를 입력받아서 메모리 버퍼에 포인터를 전달하는데 커널은 이 때 유저 모드 메모리에서 데이터를 안전하게 가져오기 위해 메모리를 매핑 해제하거나 내용을 바꿀 수 있는 어플리케이션을 핸들링 한다.

커널은 유저 메모리에 안전하게 액세스하기 위해 여러 함수들을 제공한다. (`ProbeForRead`, `ProbeForWrite` 등등) 이러한 함수들은 들어온 메모리 주소가 유저모드 공간인지, 매핑되어 있는지 확인할 수 있다. 안전한 경우 커널은 유저메모리에 있는 데이터를 읽을 수 있으며 이 때 매핑 해제와 같은 조작을 방지하기 위해서 exception handler 내에서 어떠한 접근도 이루어지면 안된다.

데이터가 수정되는 것을 막기 위해 커널은 데이터를 한 번만 읽기 위해 복사본을 저장한다. 이 때 모든 데이터를 커널 메모리에 복사하는 것은 성능상 좋지 않다. 그래서 사용자 메모리를 잠근 후(`MmProbeAndLockPages`) 커널 주소 공간의 주소를 같은 메모리 페이지를 가리키도록 매핑하는 경우도 많다.(`MmMapLockedPagesSpecifyCache`) 이 때, 유저모드의 어플리케이션이 여전히 데이터를 수정할 수 있으므로 exception handler를 이용하는 것이 좋다.

<p align="center">
  <img src="/img/posts/Blackswan/Untitled.png" style="width: 60%;">
</p>

## Windows Sockets

윈도우 소켓 프로그래밍은 보통 winsocket API를 이용한다. 다른 운영체제들은 syscall 인터페이스로 구현되는 POSIX 소켓 API를 사용한다. 따라서 마이크로소프트도 winsock에 POSIX 소켓 호환 API를 추가하였지만 커널인터페이스는 여전히 언도큐먼트인 AFD이다.

AFD가 커널 TCP/IP스택의 유저모드 인터페이스지만 커널드라이버들에 의해 사용되는 두가지 인터페이스가 있다. Transport Device Interface (TDI), Winsock Kernel (WSK)이다.

TDI는 윈도우 네트워킹 스택의 transport 계층에 액세스할 수 있는 드라이버용 레거시 인터페이스이다. (which originally included TCP/IP, NetBIOS and AppleTalk transport providers)

네트워킹 스택은 윈도우 비스타 부터 생겼으며 TDI 확장 드라이버는 TDI의 기능을 새로운 TCP/IP 인터페이스에 매핑한다. WSK는 소켓과 같은 인터페이스를 모든 새로운 드라이버를 위해 TCP/IP 스택에 제공한다.
<p align="center">
  <img src="/img/posts/Blackswan/Untitled1.png" style="width: 60%;">
</p>

## Ancillary Function Driver

AFD는 사용자모드 `winsock` 라이브러리에서만 사용할 수 있는 인터페이스를 사용자 모드에게 보여준다. 하지만 소켓을 사용할 수 있는 어플리케이션은 AFD와 직접적으로 통신할 수 있다.

AFD와 통신하기 위해서는 핸들은 반드시 원하는 리소스에 열려있어야 한다. 소켓핸들을 위해서 이는 open packet structure로 세팅된 확장된 어트리뷰트들로 AFD device를 여는 것으로 이루어진다.

open packet은 소켓 생성을 위한 transport와 match되게 하는 transport parameter를 명시한다.

Default transport layer providers는 다음과 같다. TCP, UDP, Raw and Unix

AFD와 통신은 79개의 다른 함수로 이루어져 있는 IOCTL 콜을 이용해야 한다.

<p align="center">
  <img src="/img/posts/Blackswan/Untitled2.png" style="width: 60%;">
</p>

### Socket Creation

IOCTL 호출을 AFD 드라이버로 전송하기 위해서는 핸들을 열어야 한다. extended attributes buffer의 내용에 따라 여러 유형의 엔드포인트를 열 수 있다. 일반 적인 소켓 엔드포인트는 ‘AfdOpenPacetXX’ 형식의 확장 특성을 사용하여 생성되며, 여기서 contents 는 일반적인 소켓 생성 옵션과 optional transport name을 지정하는 구조체이다.

생성된 엔드포인트에는 open packet의 contents에 따라 설정된 옵션과 플래그를 가지고 있으며 이러한 옵션은 IOCTL 호출을 처리할 때 엔드포인트가 어덯게 동작할지를 결정한다. transport name은 TDI 장치 개체의 하위의 경로이며 (i.e. ‘\Device\Tcp’) 지정된 경우 디바이스에 대한 내부핸들이 열린다.

### Socket Options

소켓 핸들에 대한 일반적인 동작은 TCP/IP 스택의 다른 layer들의 속성을 회수하고 세팅하는 것이며 이에 대한 API 함수들은 `getsocketopt`, `setsocketopt` and `ioctlsocket`.

이러한 기능은 **AfdTliIoControl IOCTL**에 의해 호출된다. 이 인터페이스는 하위 수준 드라이버의 여러 엔드 포인트로 전달되기 때문에 집중해서 봐야 한다.

## CVE-2021-34514 ALPC TOCTOU LPE

어플리케이션이 ALPC를 만들 때 성능향상을 위해 ALPC 메세지 수신을 위한 공유 메모리 영역을 지정할 수 있다. 이때, 메세지가 ALPC 포트로 전송되면 커널은 메시지와 메타데이터를 이 **completion list**로 알려진 공유메모리 영역으로 copy 한다. 이는 `AlpcpCompleteDispatchMessage` 함수로 관리된다.

이 함수는 [비트맵 섹션](https://docs.microsoft.com/ko-kr/windows/win32/wmp/bitmaps-section)을 이용해서 **completion list**의 공간을 할당한다. 그 후 메세지의 헤더와 내용을 shared user mode region에 복사한다. 만약 메세지가 attributes를 포함하면 해당 attributes도 복사된다. Attributes는 보통 메세지 데이터 바로 다음에 저장된다. 이 때, 이 attributes의 위치를 계산할 때 메세지 길이(TotalLength)는 shared user mode region의 메세지 헤더에서 읽는다.

악의적인 어플리케이션이 커널이 메세지 헤더를 복사한후 attributes 를 가리킬 포인터가 계산되기 전에 공유 메모리의 메세지 길이를 변경하면 버퍼를 지나쳐서 쓰게된다. 이로 인해 메모리 코럽션과 BSOD가 발생할 수 있다. 

이러한 취약점을 exploit하는 방법을 찾던중 Windows의 커널의 수많은 영역을 뒤져서 커널 주소 공간을 제어하는 방법을 찾았다. 이 영역 중 하나는 windows Sockets 이다.

## Socket IOCTL Validation Bypass

소켓이 특정한 transport name으로 만들어질때, TDX 핸들은 소켓이 주소에 바인딩되 때 endpoint에 열릴 것이다. 만약 socket을 생성하는 correct flags 가 세팅되어 있을 때, address handle은 `AfdQueryHandles IOCTL`에 의해 유저모드에게 반환될 수 있다. 

이 핸들을 이용하여 internal socket IO control requests(?) 는 `TdxIssueIoControlRequest` 함수를 이용하여 전달된 코드 유형에 제한이 없이 직접 요청될 수 있다. 이 함수는 유저 모드에서 사용될 수 없다. 

AFD에서 유효성 검사를 하고  TDX 드라이버에게 copy하는 식으로  validation bypass는 패치되었다.

- AFD 내에서 `AfdAllowedUserIOControlRequest` 함수를 통해 validation을 하고 TDX 드라이버의 `TdxTdiAllowedUserIOControlRequest` 함수에게 전달한다.

세가지의 다른 메모리 코럽션 취약점 및 정보 노출 취약점이 발견되었다.

<p align="center">
  <img src="/img/posts/Blackswan/Untitled3.png" style="width: 60%;">
</p>

## CVE-2021-38629 Socket Query Security InfoLeak

TCP address endpoint 로 전달된 Socket options calls는 IOCTL 의 타입을 특정하는 핸들링 루틴을 호출하는 `TcpTlEndpointIoControlEndpointCalloutRoutine` 라는 함수에 의해 TCPIP.sys 드라이버로 전달된다. (i.e. `setsockopt`, `ioctlsocket`, `internal`)

만약 IOCTL validatioin bypass가 internal type과 함께 사용하는 경우 요청은 `TcpIoControlEndpoint` 함수에서 처리되며 0x20 코드가 `TcpQuerySecurityEndpoint` 함수를 호출한다. 이 함수는 엔드포인트 security descriptor를 읽고 output buffer에서 포인터를 반환한다. 

security descriptor는 **PagePool**에 할당되고 이 포인터의 leak은 ASLR bypass 이다.  이 정보의 누출은 메모리 코럽션 버그를 exploit 하기 더 쉽게 해준다.

이 취약점은 UDP에서도 똑같이 나타난다.

## CVE-2021-38638 #1 Socket Set Security LPE

TCP address endpoint으로 전송된 소켓 옵션 호출은`TcpTlEndpointIoControlEndpointCalloutRoutine`함수에 의해서 TCPIP.sys로 수신된다.

만약 IOCTL validatioin bypass가 internal type과 함께 사용하는 경우 요청은 `TcpIoControlEndpoint` 함수에서 처리되며 0x18 코드가 `TcpSetSecurityEndpoint` 함수를 호출한다. 이 함수는 엔드포인트 security descriptor를 사용자 모드에서 지정한 임의의 포인터로 바꾼다.

`ObReferenceSecurityDescriptor`가 호출되에 따라 security descriptor에 대한 referece count가 증가하고 이 결과로 인해 커널 메모리에 full read와 write access로 활용할 수 있는임의의 커널 주소가 증가한다.

### Impact

이 취약점을 이용하여서 커널메모리에 대한 full read 와 write access를 얻어서 시스템 전체를 조작 가능

TCP 또는 UDP transport device(’\Device\Tcp’)에 대한 핸들을 열 수 있는 모든 프로세스에서 취약한 코드를 이용할 수 있다. 이 때의 권한은 ‘Everyone’이다.

### Root Cause Analysis

정상석인 ioctlsocket call은 `AfdTiloControl`에 의해 처리되고 이 함수는 유저모드가 IOCTL socket codes의 subset만 호출할 수 있는지 확인한다. 이 검사는 `AfdAlloedUser` 에서 수행되는데 이는 `TcpIoControlEndpoint` 가 호출되도록 하는 IOCTL 타입을 특별히 막는 함수이다.

socket IOCTL 메세지를 TDX 드라이버를 통해 전송하여서 AFD 드라이버의 검증 루틴을 우회할 가능성은 고려되지 않은 것같다. 따라서 하위 계층의 TCP/IP 드라이버는 항상 다른 커널 모드 드라이버에 의해 호출된다는 것은 사실이 아니다. 이는 역시 information leak에 사용될 수 있다.

## CVE-2021-38638 #2 Socket Associate QoS LPE

UDP address endpoint에 전송되는 Socket options calls는 IOCTL 의 타입을 특정하는 핸들링 루틴을 호출하는`UdpTlEndpointIoControlEndpointCalloutRoutine` 함수에 의해서 PACER.sys로 전달된다. (i.e. `setsockopt`, `ioctlsocket`, `internal`)

만약 ioctlsocket이나 setsockopt 타입들이 이용되면 request는 `UdpSetSockOptEndpoint`에 의해 핸들링된다. socket IOCTL validation bypass를 이용하고 SIO_RESERVed_1(0x8800001A)를 이용하면 QimInspectAssociateQoS 에 의해 핸들링 되도록 할 수 있지만 AFD.sys에서 일반적으로 발생하는 메시지 변환이 없다.

QoS는 일반적으로 PACER.sys에 의해 핸들링되고 Associate QoS message는 결과적으로 input 데이터에서 임의의 object pointer를 읽는 `PcpValidateAndReferenceFlow` 함수로 전달된다.

PcpReferenceFlow의 object에 대한 reference count가 증가하므로 임의의 커널메모리에 대한 full read와 write access를 얻을 수 있는 kernel memory 주소가 증가한다.

이 취약점은 TCP address endpoints에도 적용이된다.

### Root Cause Analysis

일반적인 ioctlsocket call은 `AfdTilIoControl`에 의해 처리되며 이 함수에는 특별하게 `AfdTliIoControlHandleAssociateQoS` 함수를 호출하여 Associate QoS 요청을 처리할 수 있다. 이 루틴에서는 핸들을 열고 FILE_OBJECT 포인터 저장을 하는 등 사용자 모드에서 전달된 데이터의 유효성 검사하고 복사본을 만든다.

socket IOCTL 메세지를 TDX 드라이버를 통해 전송하여서 AFD 드라이버의 검증 루틴을 우회할 가능성은 고려되지 않은 것같다. 따라서 하위 계층의 TCP/IP 드라이버는 항상 다른 커널 모드 드라이버에 의해 호출된다는 것은 사실이 아니다. 이는 역시 Set QoS LPE vulnerability에 이용될 수 있다.

## CVE-2021-38638 #3 Socket Set QoS LPE

UDP address endpoint에 전송되는 Socket options calls는 IOCTL 의 타입을 특정하는 핸들링 루틴을 호출하는`UdpTlEndpointIoControlEndpointCalloutRoutine` 함수에 의해서 PACER.sys로 전달된다. (i.e. `setsockopt`, `ioctlsocket`, `internal`)

만약 ioctlsocket이나 setsockopt 타입들이 이용되면 request는 `UdpSetSockOptEndpoint`에 의해 핸들링된다. socket IOCTL validation bypass를 이용하고 SIO_RESERVed_1(0x8800000B)를 이용하면 QimInspectSetQoS 에 의해 핸들링 되도록 할 수 있지만 AFD.sys에서 일반적으로 발생하는 메시지 변환이 없다.

IOCTL의 input으로 기대되는 구조체는 다음과 같다.

```c
typedef struct _QualityOfServie
{
	FLOWSPEC SendingFlowspec; //the flow spec for data sending
	FLOWSPEC ReceivingFlowspec; //the flow spec for data receiving
	WSABUF Provider Specific;  //additional provider specific stuff
} QOS, FAR * LPQOS;
```

QoS 기능은 PACER.sys에 의해서 처리되면서 Associate QoS 메시지는 결국 PcpValidateFlowParameters 함수에 전달되는데 이 함수는 ProviderSpecific buffer에 유저모드 버퍼를 제대로 pobing과 locking하지 않고 액세스하는 함수이다. 이로 인해 메모리 액세스 위반이 발생할 수 있다.

더욱 흥미로운 점은 QoS 구조가 포함된 버퍼가 유저모드 메모리에 존재해서 액세스 중에 변경할 수 있다는 것이다. 이 때 TOCTOU가 발생하여 버퍼의 길이를 읽어 할당 크기를 계산한다음 다시 내용을 읽어 버퍼에 복사할 때, 버퍼 오버플로를 발생 시켜 full read와 write access를 얻을 수 있다.

## Exploitation of Vulnerabilities

여기서는 메모리 코럽션 취약점을 LPE공격으로 전환하는 방법에 대한 설명한다. 

CVE-2021-38638 #1과 CVE-2021-38638 #2는 둘 다 임의 커널 주소 증가를 허용하는 반면 CVE-2021-38638 #3과 CVE-2021-38628은 둘 다 NonPaged pool의 버퍼 오버플로이다. 일부 윈도우즈 커널 내부 정보를 이해하면 커널 풀 손상을 안정적으로 이용하는 것이 비교적 쉽다.

최신 버전의 윈도우즈 커널에서는 segment heap을 Paged and NonPaged pools에 대한 할당자로 사용한다. segment heap 은 할당 크기 및 정렬에 따라 사용되는 몇 가지 백엔드가 있다. 가변 크기 백엔드는 공격의 대상이 될 수 있다.

<p align="center">
  <img src="/img/posts/Blackswan/Untitled4.png" style="width: 60%;">
</p>

VARIABLE SIZE BACKEND STRUCTURES

### Variable Size Backend Baseics

Segment heap의 VS(Varable Size) 백엔드는 512바이트 이상, 128kB 이하의 크기의 할당에 사용된다. VS 백엔드는 가변 크기의 메모리 청크로 분할된 하위 segment의 링크된 목록을 포함한다. 기존 청크로 만족할 수 없는 할당 요청이 터리되면 새 하위 세그먼트가 생성된다. 하위 세그먼트는 할당 요청의 두 배 크기로 반올림되어 생성된다. (0xf800 이면 0x20000으로) Subsegment의 가장 첫 페이지는 헤더가 포함되어 있으며 사용되지 않은 메모리에서 빈 청크가 만들어지고 VS FreeChunkTree에 저장된다.

<p align="center">
  <img src="/img/posts/Blackswan/Untitled5.png" style="width: 60%;">
</p>

0xf800bytes VS Subsegmet after an inital allocation LAYOUT

### Pipe Grooming

Windows pipe는 커널에서 제어되는 할당을 얻는데 유용한 방법이다. Pagedpool pipe를 grooming하기 위해 attributes를 이용할 수 있다. NonPagedPool pipe를 grooming 하기 위해 data queue 엔트리가 사용될 수 있다

<p align="center">
  <img src="/img/posts/Blackswan/Untitled6.png" style="width: 60%;">
</p>

PIPE STRUCTURES

Pipe attruibutes 는 ucocumented IOCTL code인 `NtFsControlFile`을 이용하여 추가된다. 다른 이름의 attributes는 linked list에 새로운 entry로 저장된다. Data queue 엔트리들은 pipe에 기록하여서 작성되며 파이프의 다른 끝에서 읽을 때까지 linked list에 남아있는다. 

두 구조의 공통점은 doubly linke list entry로 시작한다는 것이다. 공격자가 overflow 데이터를 완전히 제어할 수 있으면 전체 항목을 직접 제어할 수 있다.

attribute와 data queue에 있는 데이터는 유저 모드에서 다시 읽을 수 있으므로 이러한 헤더 구조 중 하나를 제어하면 임의의 커널 읽기가 발생한다.

파이프가 생성되면 이 context structure(CCB라고도 불림)도 만들어진다. CCB에는 receive data queue와 file object 에 대한 포인터가 포함되어 있다.

### Exploiting NonPagedPool Overflows

CVE-2021-38638#3과 CVE-2021-38628  모두 원하는 데이터와 길이를 가지고 NonPagedPool에 오버플로를 발생시킬 수 있다. 이 두 버그의 차이점은 CVE-2021-38628의 오버플로우 길이는 IP주소 크기의 배수여야 한다는 것이다. (IPv4의 경우 4바이트, IPv6의 경우 16바이트). 따라서 플링크 포인터의 least significatn byte를 손상시킬 수 없어서 전체 data queue entry header를 덮어써야 한다.

Grooming은 NonPagedPool allocation을 alternation 하여 아래와 같은 VS segment를 얻는다.

<p align="center">
  <img src="/img/posts/Blackswan/Untitled7.png" style="width: 60%;">
</p>

NONPAGEDPOOL GROOMING LAYOUT

이 레이아웃에 있는 여러 Subsegment들을 아무거나 덮어 씌울 수 있다. 첫번째 청크는 다른 free chunk 보다 크다. (0xfc0 bytes) 

할당크기를 0xf80, 오버플로 길이 0x60으로 원하는취약점을 트리거할 수 있으며 이로 인해 대상 청크의 heap header와 target data queue entry structure를 덮어씌울 수 있다.

<p align="center">
  <img src="/img/posts/Blackswan/Untitled8.png" style="width: 60%;">
</p>

NONPAGEDPOOL OVERFLOW LAYOUT