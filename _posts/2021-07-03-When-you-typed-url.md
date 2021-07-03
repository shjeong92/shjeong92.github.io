---
layout: article
title: "브라우저에 URL을 입력했을 때 일어나는 일들"
tags: 네트워크 브라우저
---

이번 포스트 에서는 브라우저에 *www.naver.com*을 입력해보고 어떠한 일이 일어나는지 과정을 알아보겠습니다.

### 1. 브라우저의 URL 파싱

![URL](https://user-images.githubusercontent.com/75003424/124287483-0671f200-db8b-11eb-9498-e17442d62bb7.png)

URL을 입력받은 부라우저는 먼저 해당 URL의 구조를 해석합니다.
- 어떤 프로토콜을 사용할지
- 어느 도메인으로 보낼지
- 어떤 포트로 보낼지

해석하게 되는것입니다.

명시적으로 포트를 선언하지 않아도 브라우저에서는 설정된 기본값을 이용하여 요청하게되는데요, HTTP 라면 80번 포트를, HTTPS의 경우 443번 포트를 기본 값으로 요청하는 것입니다.

### 2. HSTS 목록 조회

HSTS(HTTP Strict transport security), HTTP를 허용하지 않고 HTTPS를 사용하는 연결만 허용하는 기능입니다. 만약 HTTP로 요청이 왔다면 HTTP 응답 헤더에 "Strict Transport Security"라는 필드를 포함하여 응답하고 이를 확인한 브라우저는 해당 서버에 요청할 때 HTTPS만을 통해 통신하게 됩니다. 그리고 자신의 HSTS캐시에 해당 URL을 저장하는데 이를 HSTS 목록이라고 부릅니다.

이를 통해 브라우저에서는 이 HSTS 목록 조회를 통해 해당 요청을 HTTPS로 보낼지 판단합니다. HSTS목록에 해당 URL이 존재한다면 명시적으로 HTTP를 통해 요청한다 해도 브라우저가 이를 HTTPS로 요청합니다.

### 3. URL을 IP주소로 변환


**www.google.com** 이라는 주소로는 컴퓨터끼리 통신할 수 없습니다. 이를 인터넷 상에서 컴퓨터가 읽을 수 있는 IP주소로 변환해야 서로 통신이 가능하게 됩니다. 우선 브라우저에서는 자신의 로컬 hosts 파일과 브라우저 캐시에 해당 URL이 존재하는지 확인합니다. 존재하지 않는다면 도메인 주소를 IP주소로 변환해주는 DNS(Domain Name System) 서버에 요청하여 해당 URL을 IP주소로 변환합니다. 

**DNS 서버로 요청하는 과정**

![dns](https://user-images.githubusercontent.com/75003424/124294979-4b018b80-db93-11eb-9b92-8b44d6763a8b.gif)


1. PC는 미리 설정되어 있는 Local DNS에게 IP 주소를 물어봅니다.
2. 만약 Local DNS에 호스트 네임에 대한 정보가 없을 경우 각 Local DNS에 설정된 Root DNS에 질의를 시작합니다.
  - Root DNS는 전세계에 13대가 구축되어 있으며, 우리나라에는 Root DNS 가 없지만, 3대의 미러 서버가 설정되어 있다고 합니다.
3. Root DNS에도 호스트에 대한 정보가 없으면, 다른 DNS 서버에게 질의할 수 있도록 요청합니다.
4.  Local DNS는 .com을 관리하는 DNS에게 호스트 네임에 대한 질의를 요청하고 요청한 결과가 없을 경우 다시 질의할 다른 DNS 서버의 주소를 알려줍니다.
5. Local DNS는 google.com을 관리하는 DNS에게 호스트 네임에 대한 질의를 요청하고 결과가 있을 경우 IP주소에 대한 결과를 반환합니다. 
6. Local DNS는 **'www.naver.com'**에 대한 IP 주소를 캐싱하고, 클라이언트에게 전달합니다. 

**Recursive Query** : Local DNS 서버가 여러 DNS 서버를 차례대로 **Root DNS 서버** -> **com DNS 서버**(Top level Domain) -> **naver.com DNS 서버**(Secondary Level Domain) 질의해서 답을 찾아가는 과정

### 4. ARP 프로세스

ARP (주소 결정 프로토콜, Address Resolution Protocol) 브로드캐스트를 보내기 위해서는 네트워크 스택 라이브러리가 검색할 목적지 IP의 주소를 알아야 합니다. 또, ARP 브로드캐스트를 보내는 데 사용하는 인터페이스의 MAC 주소 역시 알아야 합니다.

#### 항목이 ARP 캐시에 있을때
가장 먼저, ARP 캐시가 목적지 IP의 ARP 항목을 가지고 있는지 점검합니다. 만약 캐시에 있다면 라이브러리 함수는 다음의 형태로 결과를 리턴합니다: 목적지 IP = MAC.

#### 항목이 ARP 캐시에 없을때

- 라우트 테이블을 검색해서 목적지 IP 주소가 로컬 라우트 테이블의 서브넷에 존재하는지 봅니다. 존재한다면, 라이브러리가 그 서브넷에 속하는 인터페이스를 활용합니다. 없다면, 라이브러리는 우리 기본 게이트웨이의 서브넷에 속하는 인터페이스를 활용합니다.
- 선택된 네트워크 인터페이스의 MAC 주소가 검색이 됩니다.
- 네트워크 라이브러리는 Data Link Layer(OSI Layer 2) 에 ARP요청을 보냅니다.

`ARP Request`
~~~
Sender MAC: interface:mac:address:here
Sender IP: interface.ip.goes.here
Target MAC: FF:FF:FF:FF:FF:FF (Broadcast)
Target IP: target.ip.goes.here
~~~

`ARP Reply`
~~~
Sender MAC: target:mac:address:here
Sender IP: target.ip.goes.here
Target MAC: interface:mac:address:here
Target IP: interface.ip.goes.here
~~~

이제 네트워크 라이브러리는 우리 DNS 서버나 DNS 프로세스를 재개할 수 있는 기본 게이트웨이 중 하나의 IP 주소를 갖고 있습니다

- 53번 포트는 DNS 서버에 UDP 요청을 보내기 위해 열려 있습니다 (만약 응답 크기가 너무 크다면, TCP가 대신 사용되구요).
- 로컬/ISP의 DNS 서버가 해당 정보를 갖고 있지 않다면, 재귀적인 탐색이 수행되고 SOA(Service-oriented architecture)가 도달해서 결과값이 되돌아올 때까지 DNS 서버 리스트를 타고 올라갑니다

### 5. 대상과 TCP 통신을 통해 Socket열기


![3wayhswithtls](https://user-images.githubusercontent.com/75003424/124304948-607cb280-db9f-11eb-93c4-008c1425cda2.png)

HTTPS 프로토콜을 사용하므로
3-way handshake를 통하여 소켓을열고

TLS handshake 과정이 추가됩니다.

1. TCP socket 생성
2. TCP 연결을 통해 클라이언트는 실행 중인 TLS protocol의 버전, 사용가능한 암호 세트, 사용할 수 있는 TLS 옵션 목록 등을 평문으로 보냅니다.
3. 84ms : 서버는 통신할 때 사용 할 TLS의 버전을 선택하고, 클라이언트가 제공한 목록에서 암호 조합을 결정하고 인증서를 첨부해서 클라이언트에 보냅니다. 선택적으로 서버는 다른 TLS 확장에 대한 클라이언트 인증서 및 매개 변수에 대해 요청을 보낼 수도 있습니다.
4. 양측이 공통된 버전과 암호를 협상 할 수 있고, 클라이언트가 서버에서 제공한 인증서에 만족하면, 클라이언트는 RSA 또는 Diffie-hellman 키 교환을 시작합니다. 이 교환은 이어지는 세션에서 사용할 대칭키를 설정합니다.
5. 서버는 클라이언트가 전송한 키 교환 매개변수를 처리하고 MAC address을 확인하여 메시지 무결성을 검사하고 암호화 된 Finished message를 클라이언트에 전송합니다.
6. 클라이언트를 협상 된 대칭키를 사용해 message 암호를 해독하고 MAC address를 확인해서 모두 정상이면 터널이 설정되고 통신을 시작합니다.

**TLS**에 대해서는 [이곳](https://shjeong92.github.io/2021/06/09/Learning-Kubernetes-11.html)에 자세히 정리해 놓았습니다.


### 6. HTTP 프로토콜

구글이 만든 웹 브라우저라면, 페이지를 가져오기 위해 HTTP 요청을 보내는 대신, 서버에게 HTTP에서 SPDY로 "업그레이드"할 것을 협상해봅니다.

- **SPDY란**

  구글은 더 빠른 Web을 실현하기 위해 Latency 관점에서 HTTP를 고속화한 SPDY(스피디) 라 불리는 새로운 프로토콜을 구현했습니다.
SPDY는 HTTP를 대치하는 프로토콜이 아니고 HTTP를 통한 전송을 재 정의하는 형태로 구현 되어있습니다.
SPDY는 실제로 HTTP/1.1에 비해 상당한 성능 향상과 효율성을 보여줬고 이는 HTTP/2 초안의 참고 규격이 되었다고 합니다.

만약 클라이언트가 SPDY를 지원하지 않고 HTTP만 쓴다면, 서버에 다으과 같은 요청을  보냅니다

~~~
GET / HTTP/1.1
Host: google.com
Connection: close
[other headers]
~~~

**[other headers]** 부분은 HTTP 사양에 따라 콜론으로 구분되고 각각 새 줄로 나뉘는 일련의 키-값 쌍을 나타냅니다. (이 부분은 사용된 브라우저가 HTTP 스펙을 벗어나는 어떠한 버그도 없을 때를 가정해요. 웹 브라우저가 HTTP/1.1 을 쓴다는 것도 마찬가지인데, 그렇지 않을 경우엔 Host 헤더가 요청에 포함되지 않고 GET 요청에 명시된 버전이 HTTP/1.0 혹은 HTTP/0.9 일 수도 있습니다. )

HTTP/1.1은 송신자측에서 응답을 받은 직후에 연결이 끊어질 것이라는 신호를 보내기 위해 "close"라는 연결 옵션을 정의합니다. 아래의 예처럼 말이죠.

~~~
Connection: close
~~~

영구 접속을 허용하지 않는 HTTP/1.1 어플리케이션은 반드시 "close" 연결 옵션을 모든 메시지에 포함해야 합니다.

요청과 헤더를 보낸 후에, 웹 브라우저는 하나의 빈 줄을 서버에 보내 요청 내용이 모두 보내졌음을 알립니다.

서버는 요청의 상태를 나타내는 코드와 다음과 같은 형태로 응답합니다.

~~~
200 OK
[response headers]
~~~

이다음에 www.naver.com의 HTML컨텐츠를 payload에 실어서 보냅니다.
그 이후 서버는 연결을 종료할 수도있고, 클라이언트가 보낸 헤더가 요청한 경우 추가 요청을 위해 연결을 유지 할 수도 있습니다.

만약 웹 브라우저가 보낸 HTTP header에 웹 브라우저가 캐시한 파일의 버전이 마지막 검색이후 수정되지 않았으면(HTTP header의 ETag 값으로 확인) 서버에선 다음과 같이 응답합니다

~~~
304 Not Modified
[response headers]
~~~
- 이 응답에서는 payload 가 없고 웹브라우저는 캐시에서 HTML을 검색합니다.  
- HTML을 파싱한 후 웹 브라우저와 서버는 GET / HTTP/1.1요청이 아닌 HTML페이지에서 참조하는 모든 자원(Image, CSS, favicon.ico 등)에 대해 이 프로세스를 반복합니다. 
- 만약 HTML이 다른 Domain의 resource를 참조하는 경우 웹 브라우저는 다른 도메인을 확인하는 단계(**3번 단계**)로 돌아가고 해당 도메인에 대해 모든 단계를 수행하고, Host 요청의 header는 해당 서버의 이름으로 설정됩니다.

### 7. HTTP 서버의 응답

HTTPD (HTTP 데몬) 서버는 서버측에서 요청/응답을 처리하는 친구입니다. 대표적으로 자주 쓰이는 nginx 가 있습니다.

1. HTTPD (HTTP 데몬)
2. 서버는 요청을 다음과 같은 파라미터들로 분리합니다.
  - HTTP method(GET, POST, PUT, DELETE, CONNECT, OPTIONS, TRACE, HEAD 중 하나).
  주소창에 URL을 직접 입력한 경우에는 GET 이겠죠
  - 도메인, (naver.com).
  - 요청된 경로/페이지 - (www.naver.com은 홈페이지입니다, 즉 특정 경로와 페이지가 없기에 기본경로인 '/'가 들어갑니다)
3. 서버는 naver.com에 해당하는 가상 호스트가 서버에 설정되어 있는지 확인합니다.

4. 서버는 naver.com이 GET 요청을 받아들일 수 있는지 봅니다.
5. 서버는 해당 클라이언트에게 이 메소드가 허용되는지 봅니다 (IP, 인증, 기타 등등을 통하여).
6. 만약에 서버에 rewrite 설정이 되어있다면 해당하는 경로로 다시 요청을 하게됩니다. (www로 시작하지 않는것을 www로 가게하거나 아래와 같이 http로 들어온 모든요청을 https로 rewrite 하기도 하죠)

~~~
  server {
      listen 80;
      server_name    my.domain.com;
      rewrite ^(.*) https://$host$1 permanent;
  }

  server {
      listen 443;
      server_name    my.domain.com;
      # .....
  }
~~~

7. 서버는 요청에 해당하는 콘텐츠를 가져오고, 기본경로인 "/" 이므로 이 경우 index파일을 해석합니다.

8. 서버는 가져온 파일을 핸들러를 통해 분석하여 결과를 클라이언트로 보냅니다.

### 8. 브라우저 안에서 일어나는 일들

서버가 브라우저에 (HTML, CSS, JS, 이미지, ...)을 제공하면 브라우저는 아래 프로세서를 수행합니다.
- 파싱 - HTML, CSS, JS
- 렌더링: DOM 트리 생성 -> 트리 렌더링 -> 렌더링 된 트리 배치 -> 렌더링 된 트리 색칠

![rendering](https://user-images.githubusercontent.com/75003424/124343524-07e10000-dc07-11eb-8c50-0e692fd76465.png)

웹 브라우저의 기능은 서버에서 요청하고 브라우저 창에 표시하여 선택한 웹 리소스를 표시하는 것입니다. 리소스는 일반적으로 HTML문서이지만 PDF, 이미지 또는 다른 유형의 콘텐츠일 수도 있습니다. 자원의 위치는 URI(Uniform Resource Identifier)를 사용하여 사용자가 지정합니다.

브라우저가 HTML파일을 해석하고 표시하는 방법은 HTML 및 CSS 사양에 지정되어 있습나다. 이 사양은 웹 표준 단체인 W3C(World Wide Web Consortium)에서 관리합니다.

**브라우저의 일반적인 User Interface 요소**

- URI를 입력하기 위한 주소표시줄

- 뒤로 및 앞으로 버튼

- 북마크 버튼

- 새로고침 및 중지 버튼

- 홈페이지 이동 버튼

**브라우저의 구성요소들**

- 유저 인터페이스: 유저 인터페이스는 주소창, 뒤로/앞으로 버튼, 즐겨찾기 메뉴 등등을 포함합니다. 당신이 요청한 페이지를 보는 창을 제외한 브라우저의 모든 부분이죠.

- 브라우저 엔진: 브라우저 엔진은 UI와 렌더링 엔진 사이에 일어나는 일을 통제합니다.

- 렌더링 엔진: 렌더링 엔진은 요청된 내용을 보여주는 부분을 책임집니다. 예를 들어 만약 요청된 내용이 HTML이면, 렌더링 엔진은 HTML과 CSS를 분석하고, 처리된 내용을 화면에 띄워줍니다.

- 네트워킹: 네트워킹은 HTTP와 같은 네트워크 요청을, 플랫폼별로 다른 구현체를 활용해 플랫폼-독립적인 인터페이스 뒤에서 처리하죠.

- UI 백엔드: UI 백엔드는 콤보박스나 창 같은 기본적인 위젯을 그리는 데 쓰입니다. 이 백엔드는 플랫폼에 구애받지 않는 포괄적인 인터페이스를 노출시킵니다. 내부적으로는 운영 체제의 유저 인터페이스 메소드들을 활용하면서요.

- JavaScript 엔진: JavaScript 엔진은 JavaScript 코드를 분석하고 실행하는 데 활용됩니다.

- 데이터 저장소: 데이터 저장소는 유지가 되는 계층입니다. 브라우저가 쿠키같은 갖가지 종류의 데이터를 저장해둬야 할 수도 있거든요. 브라우저는 또 localStorage와 sessionStorage, IndexedDB, WebSQL, 파일시스템과 같은 저장 메커니즘을 지원합니다.

### HTML 파싱

렌더링 엔진은 네트워킹 계층에서 요청한 문서의 내용을 받아오기 시작합니다. 문서는 보통 8KB 단위로 전송됩니다.

HTML 파서의 주된 역할은 HTML 마크업을 파스 트리로 분석해내는 겁니다.

이렇게 나온 트리 ("파스 트리 parse tree") 는 DOM 요소와 속성 노드의 트리입니다. DOM은 Document Object Mode의 줄임말이고요. 이 친구는 HTML 문서와 HTML 요소를 JavaScript 같은 외부 요소와 이어주는 인터페이스의 객체 표현 방식입니다. 이 트리의 루트는 "Document" 객체입니다. 스크립트를 통한 모든 조작보다 앞서, DOM은 마크업과 거의 일대일인 관계를 갖습니다.

**파싱 알고리즘**

HTML은 일반적인 탑-다운이나 바텀-업 방식의 파서로는 분석할 수 없습니다.

그 이유는 아래오 같습니다.
- 관대한 언어적 특성.

- 브라우저는 흔히 알려진, 잘못된 HTML들을 지원하기 위해 전통적으로 에러를 용인해왔다는 사실.

- 파싱 과정은 재진입 가능하다는 것입니다. 다른 언어에서, 소스는 파싱 과정에서 변하지 않지만, HTML에서는, 동적 코드 (예를 들어 document.write() 호출을 담고 있는 스크립트 요소) 가 추가적인 토큰을 추가할 수도 있어서, 파싱 과정이 실제로 입력값을 바꿉니다.


일반적인 파싱 기술을 쓸 수 없으니, 브라우저는 임의의 파서를 활용해 HTML을 파싱합니다. 파싱 알고리즘은 토큰화와 트리생성의 단계로 이루워 져있습니다.

![parser](https://user-images.githubusercontent.com/75003424/124343711-b174c100-dc08-11eb-819f-98ddc4f021c2.png)

자세한 정보는 [이곳](https://www.w3.org/TR/2011/WD-html5-20110405/parsing.html#parsing)에서 확인가능합니다.

**파싱이 끝난후의 동작**

브라우저가 페이지에 링크돼있는 외부 자원 (CSS, 이미지, JavaScript 파일, 기타 등등) 을 가져오기 시작합니다.

이 단계에서 브라우저는 해당 문서가 상호작용 중이라는 표시를 해두고 "deferred" 모드에 있는 스크립트를 파싱하기 시작합니다: 반드시 문서를 분석한 후에 실행되어야 하는 것들이죠. 문서의 상태는 "complete" 으로 설정되고 "load" 이벤트가 발생됩니다.

HTML 페이지에 "Invalid Syntax"에러는 존재하지 않습니다. 브라우저가 어떠한 내용이든 고치고 넘어갑니다.

**CSS 분석**

`<style>` 태그 내용과, `style` 속성값으로 되어있는 CSS 파일들을 ["CSS lexical and syntax grammar"](https://www.w3.org/TR/CSS2/grammar.html) 를 활용해 파싱합니다.
각각의 CSS 파일은 Stylesheet object 로 파싱되는데, 여기서 각 객체는 selector 및 CSS 문법에 해당하는 객체들과 함께 CSS 규칙들을 담고 있습니다.
CSS 파서는 특정한 파서 생성기가 사용됐을 경우에 탑-다운이나 바텀-업도 가능합니다.

**페이지 렌더링**

1. DOM 노드를 탐색하고 각 노드에 대한 CSS 값을 계산하여 "Frame tree" 또는 "Render tree"를 만듭니다.

2. 자식 노드의 width와 수평 margin, border, padding 을 합해서 Frame tree의 아래쪽에 있는 각 노드의 기본 너비를 계산합니다.

3. 각 노드의 사용 가능한 너비를 자식 노드에 할당하여 각 노드의 실제 width 값 계산합니다.

4. 텍스트 배치를 적용하고 하위 노드의 height와 margin, border, padding을 합해 각 노드의 높이를 상향식으로 계산합니다.

5. 위에서 계산 된 정보를 사용해서 각 노드의 좌표를 계산합니다.

6. float, absolutely, relatively 와 같은 속성이 사용되었을 경우 더 복잡한 단계가 수행 됩니다. 

    자세한건  [http://dev.w3.org/csswg/css2/](http://dev.w3.org/csswg/css2/) 와 [http://www.w3.org/Style/CSS/](http://www.w3.org/Style/CSS/)current-work 참조하세요

7. 페이지의 어느 부분을 그룹으로 애니메이션화 할 수 있는지 설명하는 레이어를 만듦. frame/render object는 layer에 할당합니다

8. 텍스처는 페이지의 각 레이어에 할당합니다

9. 각 frame/render object를 통해서 각 레이어 별로 그리기 명령을 실행합니다.

10. 위의 모든 단계를 웹페이지가 렌더링 된 마지막 시간에 계산 된 값을 재사용 할 수 있으므로 점진적 변경은 작업이 덜 필요합니다.

11. 페이지 레이어는 합성 프로세스로 보내져 browser chrome, iframe, addon panels과 같은 시각적인 레이어와 결합됩니다.

12. 최종 레이어 위치가 계산되고 Direct3D / OpenGL을 통해 합성 명령이 실행된다. GPU 명령 버퍼는 비동기 렌더링을 위해 GPU로 출력되고 frame은 window server로 전송됩니다.


**GPU 렌더링**
- 렌더링 프로세스 동안 graphical computing layers는 CPU 또는 GPU를 사용할 수 있습니다.

- graphical rendering 계산에 GPU를 사용하는 경우 그래픽 소프트웨어 레이어에서 작업을 여러조각으로 분할하여 렌더링 프로세스에 필요한 부동 소수점 계산을 위해 GPU 대용량 병렬 처리를 사용 할 수 있습니다.


렌더링이 완료된 후 브라우저는 Javascript 실행을 통해 DOM과 CSSOM이 변경 될 수 있는데 레이아웃이 수정 되는 경우 페이지 렌더링 및 페인팅을 다시 수행합니다.