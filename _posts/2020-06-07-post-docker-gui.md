---
title: "Docker 컨테이너 안에서 GUI Application을 사용하는 방법"
categories:
  - how-to-guides
tags:
  - Docker
  - GUI
  - X Window System
hidden: true
---

## 배경
2020년 기준, Docker는 1050억번의 컨테이너 다운로드 횟수를 기록하고, 매달 200만명의 사용자를 보유한 거대한 소프트웨어 플랫폼이다. 나 또한 대학원 연구를 진행 하면서 deep learning 모델 학습을 위한 TensorFlow 개발환경이 필요했고, 개발환경을 빠르게 구축하기 위해서 Docker를 적극 활용했었다. 그런데 문제는 Docker를 쓰면 컨테이너 안에서 GUI(Graphic User Interface)를 지원하는 application을 쓸 수가 없다는 것이다! 연구 특성상 실험 결과에 대한 그래프를 뽑아 볼일이 많았는데, matplotlib의 `plt.show()`와 같은 함수를 쓸 수가 없으니 여간 불편한 일이 아니었다. 그래서 Docker 컨테이너 안에서 GUI application을 사용하는 방법을 찾아보게 되었고, X window system을 활용하여 해결한 방법을 찾게 되었다. 이때 방법을 찾아보면서 얻은 정보들을 예시와 함께 블로그에 정리해 두고자 포스팅을 시작하게 되었다.

급한 분들은 아래 [방법](#방법)을 바로 참고 하면 될 것이다.

## 원리
X window system을 활용하여 컨테이너 안에서 GUI를 사용 할 수 있다고 했는데, 그럼 **X window system**란 무엇일까? X window system은 주로 UNIX 계열 운영체제에서 사용되는 GUI인데, 여기서 window system이란 말그대로 '창'개념을 도입한 GUI이다. 우리가 흔히 보는 macOS나 MS windows의 GUI는 전형적인 window system에 속하며, 각각 Quartz Compositor나 Desktop Window Manager와 같은 고유의 window system을 사용하여 GUI를 표현한다. 그렇다면 X window system은 기존의 window system들과 어떤 점이 다른 것인가? 아래의 내용은 wikipedia의 X window system에 대한 내용 중 일부를 발췌한 것이다.

> X was specifically designed to be used over network connections rather than on an integral or attached display device. - Wikipedia

특이한 점은 network connections, 즉 원격으로 GUI를 제공하기 위한 system인 것이다. 그래서 X window system은 네트워크 기반의 client-server 모델을 활용하여 GUI를 구현한다. 이때 사용되는 프로토콜을 X protocol이라 하고, 클라이언트와 서버를 각각 X client, X server라 부른다. 아래는 이해를 돕기 위해 X window system이 동작하는 예시를 그림을 통해 표현한 것이다.

<br>
![figure1](/assets/images/posts/docker-gui/figure1.png){: .align-center}
<figure style="display: block; text-align: center;"><figcaption>그림 1: X window system이 동작하는 예시(컴퓨터 A - 서버)</figcaption></figure>

컴퓨터 A에서 서버에 설치된 Firefox 브라우저를 실행하고 그 화면을 컴퓨터 A에서 표시하려는 상황을 생각해보자. Firefox(X client)는 실행 화면을 생성하여 컴퓨터 A에 설치된 X server로 화면 내용을 전송한다. 그러면 X server는 전송받은 화면을 컴퓨터 A의 모니터에 표시한다. 그 후 만약 컴퓨터 A에서 이벤트가 발생하면(가령 웹사이트에 접속하기 위해 마우스를 클릭 하는 이벤트가 발생하는 경우) 컴퓨터 A는 서버의 Firefox에 해당 이벤트를 알려준다. 이벤트 내용을 받은 Firefox는 해당 이벤트에 대한 처리(이벤트가 웹사이트 접속이라면 해당 사이트의 화면을 생성)를 완료한 후 갱신된 화면 내용을 다시 X server로 보내게 된다. X server는 갱신된 화면을 컴퓨터 A의 모니터에 표시하고 새로운 이벤트가 발생하면 위의 과정이 반복된다. 이러한 과정을 통해 X window system은, GUI 환경을 원격으로 제공 할 수 있는 것이다.

그런데, X server가 Firefox에게 화면을 요청한다고 생각한다면, "Firefox가 X server" 라고 생각 할 수 있다. 그러나, 이것은 X window system이 응용프로그램 관점에서 설계 되었기 때문에 생기는 오해이다. 즉, X server가 응용프로그램에 I/O 서비스 (화면, 키보드, 마우스 등)를 제공하고 응용 프로그램은 이러한 서비스를 이용하는 개념이라 보는 것이다.

여기까지 이해를 했다면 왜 Docker 컨테이너 안에서 GUI를 사용할 때 X window system을 사용하는지 알 수 있을 것이다. 아래의 그림과 같이 서버를 Docker 컨테이너, 컴퓨터 A를 host OS라 생각한다면, 똑같은 원리를 적용하여 Docker 컨테이너 안에서 GUI를 이용 할 수 있다.

<br>
![figure2](/assets/images/posts/docker-gui/figure2.png){: .align-center}
<figure style="display: block; text-align: center;"><figcaption>그림 2: X window system이 동작하는 예시 (Host OS - Docker 컨테이너)</figcaption></figure>

## 방법
Docker는 리눅스 컨테이너를 기반하여 동작하므로, X window system을 기본적으로 지원한다. 따라서 우리는 host OS에서 X server만 설치해 주면 Docker 컨테이너 안에 있는 응용 프로그램의 GUI를 host OS에서 제공 받을 수 있다. X server 설치와 설정 방법을 host OS에 따라 크게 macOS와 MS Windows로 나누어 설명 하도록 하겠다.

#### 1-a. macOS의 경우
전에는 macOS에서 기본적으로 X window system을 지원 했지만 현재는 더이상 지원하지 않는다. 대신 Xquartz를 사용하여 macOS에 대한 X window system을 계속 이용 할 수 있다. 따라서, host OS(macOS)에 X server를 설치하기 위해 [여기서](https://www.xquartz.org/index.html){: target="_blank"} Xquartz 최신 버전을 설치하도록 하자. 설치를 마쳤다면, Xquartz의 환경 설정에서 아래에 표시된 옵션을 켜주도록 하자.

<br>
![figure3](/assets/images/posts/docker-gui/figure3.png){: .align-center}
<figure style="display: block; text-align: center;"><figcaption>그림 3: Xquartz 환경설정</figcaption></figure>

우리는 Docker 컨테이너 안에서 macOS의 X server로 접속할 것이므로, "네트워크 클라이언트에서의 연결을 허용" (Allow connections from network clients) 옵션을 체크 해주어야 Docker 컨테이너 안에서 접속이 가능하다.

다음은 Docker 컨테이너 안의 X client가 X server에 접근 할 수 있도록, IP를 허용해 주어야 한다. 그러나 Docker 컨테이너는 실제 원격 컴퓨터가 아니므로 내부접속을 위한 loopback주소(localhost)를 통해 접속 하면 될 것이다. 따라서, 터미널을 열고 다음과 같이 입력하도록 하자.

``` bash
$ xhost localhost
localhost being added to access control list
```

xhost 명령어로 확인 해보면 아래와 같이 localhost가 추가되어있는 것을 확인 할 수있다.

``` bash
$ xhost
access control enabled, only authorized clients can connect
INET:localhost
INET6:localhost
```

#### 1-b. MS windows의 경우
MS windows에서 지원하는 X server는 여럿 있지만 그 중 해외 여러 커뮤니티에서 비교적 평이 좋은 VcXsrv를 사용해 볼 것이다. [여기서](https://github.com/ArcticaProject/vcxsrv){: target="_blank"} VcXsrv 최신버전을 설치 하도록 하자.

VcXsrv를 설치 하고, XLaunch를 실행하면 아래와 같이 설정창이 나타난다.

<br>
![figure4](/assets/images/posts/docker-gui/figure4.png){: .align-center}
<figure style="display: block; text-align: center;"><figcaption>그림 4: Display settings</figcaption></figure>

원하는 display type을 선택하고, display number를 입력하도록하자. 이 예시에서는 display type의 경우 muliple windows, display number의 경우 -1 (기본값 - 자동설정)을 선택했다. display number의 경우 0을 사용해도 무방하다. Display number의 의미는 뒤에서 설명 하도록 하겠다.

<br>
![figure5](/assets/images/posts/docker-gui/figure5.png){: .align-center}
<figure style="display: block; text-align: center;"><figcaption>그림 5: Client startup</figcaption></figure>

Start no clinet를 선택하고 넘어가도록하자.

<br>
![figure6](/assets/images/posts/docker-gui/figure6.png){: .align-center}
<figure style="display: block; text-align: center;"><figcaption>그림 6: Extra settings</figcaption></figure>

필요한 부분이 있다면 체크 하고 넘어 가도록하자. 이번 예시에서는 위와같이 설정하였다.

<br>
![figure7](/assets/images/posts/docker-gui/figure7.png){: .align-center}
<figure style="display: block; text-align: center;"><figcaption>그림 7: Finish configuration</figcaption></figure>

지금까지의 설정 내용을 Xconfig 파일로 저장하는 버튼이 있는데, 이때 이 파일을 저장하여 시작프로그램(win-R shell:startup)에 등록해두면 컴퓨터를 켤때 자동으로 실행되어 유용하다. 설정이 끝났다면 마침 버튼을 누르도록 하자.

#### 2. Docker 컨테이너 설정
X server 설치를 마쳤다면, 이제 Docker 컨테이너에 필요한 설정을 해줄 차례이다. X client는 X server에 접속 하기위해 컨테이너 내부에 설정된 DISPLAY 변수값을 사용한다. DISPLAY 변수의 형태는 다음과 같다.

*`hostname:displaynumber.screennumber`*
{: .text-center}

각각의 의미는 다음과 같다.

* *`hostname`*_:_ 사용할 display가 연결된 컴퓨터의 이름 이나 IP주소를 의미한다.(google.com, 10.2.3.4)
* *`displaynumber`*_:_ 입력 장치와 연결된 모니터를 나타내는 번호이다. 보통 모니터당 하나의 키보드, 마우스를 사용하므로 대부분 0번을 사용하면된다. 그러나 멀티 유저 시스템에서는 다수의 사용자가 모니터와 입력장치로 x client 어플리케이션을 이용할 수 있는데 이때 이 사용자들을 구별하기 위해서 사용하는 번호이다. 다수의 X server를 구별하기 위한 번호라고 이해해도 무방하다.
* *`screennumber`*_:_ 듀얼모니터와 같은 다수의 모니터와 하나의 입력장치를 사용할 때, 표시할 모니터의 번호를 의미하고 이 값은 생략이 가능하다.

예를 들어 DISPLAY 변수가 `DISPLAY=172.17.0.1:1.0`과 같이 설정 되어있다면, 172.17.0.1 주소의 1번 디스플레이 안의 0번째 스크린(모니터)에 출력을 하라는 의미이다. 그림으로 표시하면 아래와 같다.

<br>
![figure8](/assets/images/posts/docker-gui/figure8.png){: .align-center}
<figure style="display: block; text-align: center;"><figcaption>그림 8: DISPLAY 변수 값이 172.17.0.1:1.0 일 때 화면이 출력되는 위치(붉은 테두리) 예시</figcaption></figure>

이제 Docker 컨테이너 안에서 host OS의 X server에 접속하려면 DISPLAY 변수를 어떻게 설정해야 할까? displaynumber는 X server를 설치할때 설정(따로 설정이 없으면 0) 했으니 알 수 있고 screennumber도 모니터의 개수에 따라 알수있다. hostname 값이 문제가 되는데, localhost로 설정한다면 이 loopback주소는 host OS의 컴퓨터를 의미하는 것이 아닌 컨테이너 자체를 의미한다. 이런 문제로 인해 Docker는 host OS에 접근하기 위한 DNS를 다음과 같이 내부적으로 지원한다.

*`host.docker.internal`*
{: .text-center}

그렇다. 예를들어 host OS의 8000번 포트에 접근하고 싶다면 host.docker.internal:8000과 같이 접근하면 되는 것이다.

따라서 Docker 컨테이너를 생성 할때 다음과 같이 환경변수를 추가해 주면 될 것이다.

``` bash
$ docker create -it -e DISPLAY=host.docker.internal:0.0 --name ubuntu ubuntu:18.04 /bin/bash
```

## 예시
이제까지 알게된 내용이 잘 동작 하는지 실제로 테스트를 해보자. 기본 Ubuntu 컨테이너에 Firefox를 설치 한 후 Firefox의 GUI를 host OS에서 실행 하도록 해 볼 것이다.

[방법](#방법)의 X server 설치를 마쳤다면, ubuntu 컨테이너를 환경변수와 같이 생성 해보자. 이 예시에서는 아래와 같이 생성 하였다.

``` bash
(hostOS) $ docker run -it -e DISPLAY=host.docker.internal:0.0 --name ubuntu ubuntu:18.04 /bin/bash
```

이제 컨테이너 안에서 아래와 같이 FireFox를 설치 해보자.

``` bash
(ubuntu container) $ apt update
(ubuntu container) $ apt install firefox
```

마지막으로 Firefox를 실행하면 아래와 같이 잘 실행되는 것을 볼 수 있다.

``` bash
(ubuntu container) $ firefox
```
// 이미지 추가

## 요약
1. host OS에 X server를 설치
2. Docker 컨테이너 내부에 DISPLAY 환경변수 설정
3. Application 실행 

## 참고
* X.Org Foundation, [https://www.x.org/wiki/](https://www.x.org/wiki/){: target="_blank"}
* Wikipedia (X Window System), [https://en.wikipedia.org/wiki/X_Window_System](https://en.wikipedia.org/wiki/X_Window_System){: target="_blank"}

<link href="/assets/css/page.css" rel="stylesheet" />
