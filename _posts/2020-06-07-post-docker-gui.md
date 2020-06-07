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

## 포스팅을 하게 된 배경
연구를 하면서 deep learning 모델 학습을 위한 tensorflow 개발환경이 필요했다.<br>
이때 개발환경을 빠르게 구축하기 위해서 docker를 활용했고, 구현은 모두 docker 컨테이너 안에서 진행하게 되었다.<br>
그런데 문제는 docker를 쓰면 Graphic User Interface (GUI)를 쓸 수가 없다는 것이다!<br>
연구 특성상 실험 결과에 대한 그래프를 뽑아 볼일이 많았는데, matplotlib의 `plt.show()`와 같은 함수를 쓸 수가 없으니 여간 불편한 일이 아니었다.<br>
그래서 docker 컨테이너 안에서 GUI를 쓰는 방법을 찾아보게 되었고, X window system을 활용하여 해결한 방법을 찾게 되었다.<br>
이때 방법을 찾아보면서 얻은 정보들을 예시와 함께 블로그에 정리해 두고자 포스팅을 시작하게 되었다.

## 원리
Docker 컨테이너 안에서 GUI를 사용하기 위해서는 먼저 **X window system**에 대해 알아야 한다.<br>
X window system은 주로 UNIX 계열 운영체제에서 사용되는 GUI이다.<br>
여기서 window (MS windows가 아니다.) system이란 '창'이라는 개념을 도입한 GUI이다.<br>
우리가 흔히 보는 macOS나 MS windows의 GUI는 모두 이 window system에 속하는 것이다.<br>
그렇다면 X window system은 그냥 window system과 어떤 점이 다른 것인가?<br>
아래의 내용은 wikipedia의 X window system에 대한 내용 중 일부를 발췌한 것이다.

> X is an architecture-independent system for remote graphical user interfaces and input device capabilities. - Wikipedia

특이한 점은 remote, 즉 원격 GUI를 위한 system인 것이다.<br>
그래서 X window system은 네트워크 기반의 client-server 모델을 활용하여 GUI를 구현한다.<br>
이때 사용되는 프로토콜을 X protocol이라 하고, 클라이언트와 서버를 각각 X client, X server라 부른다.<br>
아래는 이해를 돕기 위해 X window system이 동작하는 예시를 그림을 통해 표현한 것이다.

예를들어 집에있는 MS windows 컴퓨터에서 연구실에 있는 리눅스 서버에 접속하여 해당 서버에 설치된 firefox브라우저의 gui를 집에있는 컴퓨터에서 띄우려고한다.<br>
firefox(X클라이언트)는 모니터에 그릴 화면을 내 컴퓨터에 설치된 X서버로 보낸다. 그럼 이 X서버는 전송 받은 firefox 화면을 내컴퓨터에 그려주고 마우스나 키보드를 통해 입력한 이벤트를 firefox(X클라이언트)에 보내주면, 그에 상응하는 화면을 다시 X서버에 요청하게 된다.
