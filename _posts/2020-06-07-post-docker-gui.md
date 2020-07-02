---
title: "Docker 컨테이너 안에서 GUI를 사용하는 방법"
categories:
  - how-to-guides
tags:
  - Docker
  - GUI
  - Matplotlib
hidden: true
---

## 배경
연구를 하면서 deep learning 모델 학습을 위한 tensorflow 개발환경이 필요했고, 이때 개발환경을 빠르게 구축하기 위해서 docker를 활용했었다. 그런데 문제는 docker를 쓰면 컨테이너 안에서 Graphic User Interface (GUI)를 쓸 수가 없다는 것이다! 연구 특성상 실험 결과에 대한 그래프를 뽑아 볼일이 많았는데, matplotlib의 `plt.show()`와 같은 함수를 쓸 수가 없으니 여간 불편한 일이 아니었다. 그래서 docker 컨테이너 안에서 GUI를 사용하는 방법을 찾아보게 되었고, X window system을 활용하여 해결한 방법을 찾게 되었다. 이때 방법을 찾아보면서 얻은 정보들을 예시와 함께 블로그에 정리해 두고자 포스팅을 시작하게 되었다.

## 원리
Docker 컨테이너 안에서 GUI를 사용하기 위해서는 먼저 **X window system**에 대해 알아야 한다. X window system은 주로 UNIX 계열 운영체제에서 사용되는 GUI인데, 여기서 window system이란 (MS windows가 아니다.) 말그대로 '창'개념을 도입한 GUI이다. 우리가 흔히 보는 macOS나 MS windows의 GUI는 전형적인 window system에 속하며, Quartz Compositor나 Desktop Window Manager와 같은 고유의 window system을 사용하여 GUI를 표현한다. 그렇다면 X window system은 기존의 window system들과 어떤 점이 다른 것인가? 아래의 내용은 wikipedia의 X window system에 대한 내용 중 일부를 발췌한 것이다.

> X was specifically designed to be used over network connections rather than on an integral or attached display device. - Wikipedia

특이한 점은 network connections, 즉 원격 GUI를 위한 system인 것이다. 그래서 X window system은 네트워크 기반의 client-server 모델을 활용하여 GUI를 구현한다. 이때 사용되는 프로토콜을 X protocol이라 하고, 클라이언트와 서버를 각각 X client, X server라 부른다. 아래는 이해를 돕기 위해 X window system이 동작하는 예시를 그림을 통해 표현한 것이다.

![figure1](/assets/images/posts/docker-gui/figure1.png){: .align-center}
<figure style="display: block; text-align: center;"><figcaption>그림 1: X window system이 동작하는 예시 (컴퓨터 A - 서버)</figcaption></figure>

컴퓨터 A에서 서버에 설치된 firefox 브라우저를 실행하고 그 화면을 컴퓨터 A에서 표시하려는 상황을 생각해보자. Firefox (X client)는 실행 화면을 생성하여 컴퓨터 A에 설치된 X server로 화면 내용을 전송한다. 그러면 이 X server는 전송받은 firefox 화면을 컴퓨터 A의 모니터에 표시한다. 만약 이때 컴퓨터 A에서 이벤트가 발생하면 (가령 웹사이트에 접속하기 위해 마우스를 클릭 하는 경우) firefox에 해당 이벤트를 알려준다. 이벤트 내용을 받은 firefox는 해당 이벤트에 대한 처리 (이벤트가 웹사이트 접속이라면 해당 사이트의 화면을 생성)를 완료한 후 갱신된 화면 내용을 다시 X server로 보내게 된다. X server는 갱신된 화면을 컴퓨터 A의 모니터에 표시하고, 이러한 과정을 반복함으로써 GUI 환경을 원격으로 이용할 수 있는 것이다.
<br><br>
그런데, 여기서 X server가 firefox에게 화면을 요청한다고 생각한다면, "firefox가 X server가 아니냐?" 라고 생각 할 수 있다. 그러나, 이것은 X window system이 응용프로그램 관점에서 설계 되었기 때문에 생기는 오해이다. 즉, X server가 응용프로그램에 I/O 서비스 (화면, 키보드, 마우스 등) 를 제공하고 응용 프로그램은 이러한 서비스를 이용하는 개념이라 보는 것이다.
<br><br>
여기까지 이해를 했다면 왜 docker 컨테이너 안에서 GUI를 사용할 때 X window system을 사용하는지 알 수 있을 것이다.<br>
아래의 그림과 같이 서버를 docker 컨테이너, 컴퓨터 A를 host OS라 생각한다면, 똑같은 원리를 적용하여 docker 컨테이너 안에서 GUI를 이용 할 수 있다.

![figure2](/assets/images/posts/docker-gui/figure2.png){: .align-center}
<figure style="display: block; text-align: center;"><figcaption>그림 2: X window system이 동작하는 예시 (Host OS - Docker 컨테이너)</figcaption></figure>

## 방법
Docker는 리눅스 컨테이너를 이용하여 동작하므로, X window system을 기본적으로 지원한다.<br>
따라서 우리는 host OS에서 X server만 설치해 주면 docker 컨테이너 안의 응용 프로그램의 GUI를 host OS에서 사용할 수 있다.<br>
Host OS에 따라 크게 macOS와 MS Windows로 나누어 설명 하도록 하겠다.

#### 1-a. macOS의 경우
옛날에는 macOS에서도 X window system을 지원 했지만 현재는 더이상 지원하지 않는다.<br>
대신 Xquartz를 사용하여 macOS에 대한 X window system을 계속 이용 할 수 있다.
따라서, host OS (macOS) 에 X server를 설치하기 위해 [Xquartz](https://www.xquartz.org/index.html){: target="_blank"} 최신 버전을 설치하도록 하자.<br>

Docker 컨테이너 안에서 macOS의 X server로 접속할 것이므로, 아래의 Allow connections from network clients 옵션을 켜주도록 하자.

![figure3](/assets/images/posts/docker-gui/figure3.png){: .align-center}
<figure style="display: block; text-align: center;"><figcaption>그림 3: Xquartz 환경설정</figcaption></figure>

다음은 X server에 docker 컨테이너 안의 X client가 접근 할 수 있도록, IP를 허용해 주어야 한다.<br>
그러나 docker 컨테이너는 실제 원격 컴퓨터가 아니므로 localhost 주소로 접속 할 것이다.<br>
따라서, 터미널을 열고 다음과 같이 입력하도록 하자.<br>

``` bash
$ xhost localhost
localhost being added to access control list
```

xhost명령어로 확인 해보면 localhost가 추가되어있는 것을 확인 할 수있다.

``` bash
$ xhost
access control enabled, only authorized clients can connect
INET:localhost
INET6:localhost
```

#### 1-b. MS windows의 경우
MS windows에서 지원하는 X server는 여럿 있지만 나는 vcxsrv를 사용할 것이다.

#### 2. docker 컨테이너 설정
X client는 DISPLAY환경변수에 입력된 내용을 보고 생성된 화면을 X server로 전송한다.<br>
그래서 docker 컨테이너를 생성할때 환경변수 값으로 DISPLAY변수를 아래와 같이 함께 입력 하도록 하자.<br>

``` bash
$ docker create -it -e DISPLAY=host.docker.internal:0 --name tensorflow tensorflow/tensorflow:1.15.2-py3 /bin/bash
```
여기서 코드는 DISPLAY변수의 형식은 아래와 같다.

여기서 display 번호는 ~~~~
screen 번호는 ~~~

따라서 ~~~~~

<link href="/assets/css/page.css" rel="stylesheet" />
