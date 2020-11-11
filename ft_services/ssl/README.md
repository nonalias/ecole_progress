# SSL / TLS

참고 : ([https://opentutorials.org/course/228/4894](https://opentutorials.org/course/228/4894))

## 우선, https와 http의 차이에 대해 알아보자.

- HTTP(HyperText Transfer Protocol)
    - HyperText 인 html(Hypertext Markup Language)를 전송하기 위한 통신 규약을 의미함.
- HTTPS(HyperText Transfer Protocol over Secure socker layer)
    - http 보다 보안이 강화된 것
    - http는 암호화 되지 않은 방법으로 데이터를 전송하기 때문에, server와 client가 주고받는 메시지를 감청하기가 쉽다.
    - 예를 들어, 로그인을 위해서 서버로 비밀번호를 전송하거나 혹은 중요한 기밀 문서를 열람하는 과정에서 악의적인 감청(sniffing)이나 데이터 변조 등이 일어날 수 있다는 것이다.
    - 이를 보완한 것이 https 이다.

## SSL 과 TLS의 차이점은?

- 완전히 같은 말이다. netscape에 의해 SSL이 발명되었고 이를 점차 폭넓게 사용되다가, 표준화 기구인 IETF의 관리로 명칭이 바뀌었다.
- 즉, SSL이라는 이름이 TLS로 바뀌었다는 것이다.
- TLS1.0 은 SSL3.0을 계승한다.
- 하지만 TLS 라는 이름보다 SSL이라는 말이 더욱 널리 쓰이고 있다.

## SSL 디지털 인증서

- SSL 인증서는 클라이언트와 서버간의 통신을 제 3자가 보증해주는 **전자화된 문서**이다.
- 클라이언트가 서버에 접속한 직후에 서버는 클라이언트에게 이 인증서의 정보를 전달한다.
- 클라이언트는 이 인증서 정보가 신뢰할 수 있는 것인지르르 검증한 후에 다음 절차를 수행한다.
- SSL과 SSL 디지털 인증서를 이용했을 때의 이점은 다음과 같다.
    - 통신 내용이 공격자에게 노출되는 것을 막을 수 있다.
    - 클라이언트가 접속하려는 서버가 신뢰할 수 있는 서버인지를 판단할 수 있다.
    - 통신 내용의 악의적인 변경을 방지할 수 있다.

## SSL에서 사용하는 암호화의 종류

- 크게 두 가지로 나뉜다. **대칭키**과 **공개키**.

<details>
    <summary>대칭 키</summary>
- 암호화를 할 때 사용하는 비밀번호를 key 라고 한다.
- 이 key에 따라서 암호화를 한 결과가 달라진다.
- 따라서 key 를 모르면 복호화를 할 수 없다.
- 대칭 키는 동일한 키로 암호화와 복호화를 할 수 있는 방식의 암호화 기법을 의미한다.
- 다음 과정을 따라해보자.
- `echo 'this is the plain text' > plaintext.txt`
- `openssl enc -e -des3 -salt -in plaintext.txt -out out.bin`

    ![SSL%20TLS%2088ee8f96df594b46821fc7c16aa42cb6/Untitled.png](SSL%20TLS%2088ee8f96df594b46821fc7c16aa42cb6/Untitled.png)

- key 를 입력하면 암호화가 된 파일이 만들어진다.
- 이를 다시 복호화 해보자.
- `openssl enc -d -des3 -salt -in out2.bin -out origintext.txt`

    ![SSL%20TLS%2088ee8f96df594b46821fc7c16aa42cb6/Untitled%201.png](SSL%20TLS%2088ee8f96df594b46821fc7c16aa42cb6/Untitled%201.png)

- 방금 입력한 패스워드를 다시 입력하면 복호화가 완료된다. 잘 되었는 지 확인해보자.
- `cat origintext.txt`

    ![SSL%20TLS%2088ee8f96df594b46821fc7c16aa42cb6/Untitled%202.png](SSL%20TLS%2088ee8f96df594b46821fc7c16aa42cb6/Untitled%202.png)

- 대칭 키 방식은 단점이 있다. 암호를 주고 받는 사람들 사이에 대칭키를 전달하는 것이 어렵다는 점이다. 대칭키가 유출되면 키를 획득한 공격자는 암호의 내용을 복호화할 수 있기 때문에 암호가 무용지물이 된다.
- 이런 배경에서 나온 암호화 방식이 공개키 방식이다.
</details>
<details>
    <summary>공개 키</summary>
    
- 공개 키 방식은 두 개의 키를 갖게 되는데,
- **A키로 암호화를 하면 B키로 복호화를 할 수 있고, B 키로 암호화를 하면 A키로 복호화할 수 있는 방식이다.**
- 비공개 키는 자신이 가지고 있고, 공개키를 타인에게 넘겨주면 이 공개키로 정보를 암호화를 한다. 암호화 된 그 정보를 필요한 곳에 전달하면 비공개 키를 가지고 있는 다른 누군가가 그 정보를 복호화하여 사용할 수 있다.
- 전달하는 과정에서 공개키가 유출된다고 하더라도 비공개키를 모르면 정보를 복호화할 수 없기 때문에 안전하다.
- 이런 방식을 전혀 다르게 응용해보자.
- 비공개 키를 가진 자가 정보를 비공개키로 암호화를 하고, 암호화된 그 정보와 공개키를 타인에게 넘겨준다.
    - 이렇게되면 공개키를 가진 어떤 누구라도 그 정보를 해석할 수 있는 문제가 발생하는데, 왜 이런 방식을 사용하는 지 의구심이 들 것이다.
    - 하지만 이 방식을 사용하면 그 암호화된 정보가 어떤 공개키와 매칭이 되는 지 알 수 있다.
        - 즉, 누가 만들었는 지 **신원을 알 수** 있다는 것이다.
        - ~~게다가, Website를 운영할 땐 정보를 굳이 숨기고 있지 않아도 된다!~~
    - 이러한 방식을 **전자서명**이라고 한다.
- 그럼 다음과 같은 과정을 따라해보자. (전자서명 아님)
    - `openssl genrsa -out private.pem 1024`
        - private.pem 이라는 이름의 키를 생성한다.
        - 이 key는 1024bit의 크기를 갖고, 이 숫자가 높을수록 안전하다.
    - `openssl rsa -in private.pem -out public.pem -outform PEM -pubout`
        - private.pem이라는 이름의 "비공개 키" 에 대해 public.pem 이라는 이름의 "공개 키"를 생성한다.
        - 이 공개키를 자신에게 정보를 제공할 사람에게 전송하면 된다.
    - `echo 'coding everybody' > file.txt`
        - 이 파일은 평문이기 때문에 암호화가 필요하다.
    - `openssl rsautl -encrypt -inkey public.pem -pubin -in file.txt -out file.ssl`
        - rsa방식으로 public key 를 이용해 암호화를 한 결과가 file.ssl이다. 이 file.ssl을 private key를 가지고있는 다른 사용자에게 전송했다고 가정해 보자.
    - `openssl rsqutl -decrypt -inkey private.pem -in file.ssl -out decrypted.txt`
        - 전송된 file.ssl을 비공개키를 이용해 평문으로 복호화하는 코드이다.
        
</details>

## SSL 인증서

- ssl 인증서의 기능은 크게 두 가지이다. 이 두 가지를 이해하고 있어야 한다.
    - 클라이언트가 접속한 서버가 신뢰할 수 있는 서버임을 보장한다.
    - SSL 통신에 사용할 공개키를 클라이언트에 제공한다.

## CA

- Certificate Authority 혹은 Root Certificate라고 부른다.
- 인증서는 클라이언트가 접속한 서버가 정말로 클라이언트가 의도했던 서버인지 보장하는 역할을 수행한다.
- 이 역할을 하는 민간기업들이 있는데, 이를 CA 혹은 RC라고 부른다.
- CA는 아무 기업이나 하는 것이 아니고, 신뢰성이 엄격하게 공인된 기업들 만이 가능하다.
- SSL을 통해 암호화된 서비스를 하고싶다면 CA를 통해 인증서를 구매해야 한다.

## 사설 인증기관

- 개발이나 사적인 목적을 위해서 SSL의 암호화 기능을 이용하려 한다면, 자신이 직접 CA의 역할을 할 수 있다.
- 물론 이것은 공인된 인증서가 아니기 때문에 이러한 사설 CA의 인증서를 이용하는 경우 브라우저는 경고를 출력한다.

    ![SSL%20TLS%2088ee8f96df594b46821fc7c16aa42cb6/Untitled%203.png](SSL%20TLS%2088ee8f96df594b46821fc7c16aa42cb6/Untitled%203.png)

- 공인된 CA가 제공하는 인증서를 사용한다면, 브라우저의 주소창이 아래와 같을 것이다.

    ![SSL%20TLS%2088ee8f96df594b46821fc7c16aa42cb6/Untitled%204.png](SSL%20TLS%2088ee8f96df594b46821fc7c16aa42cb6/Untitled%204.png)

## SSL 인증서의 내용

- SSL 인증서에는 다음과 같은 정보가 포함되어 있다.
    - 서비스의 정보 (인증서를 발급한 CA, 서비스의 도메인 등)
    - 서버 측 공개키 (공개키의 내용, 공개키의 암호화 방법 등)
- 인증서의 내용은 위와 같이 크게 두 가지로 구분할 수 있다.

## 브라우저는 이미 CA에 대한 정보를 가지고 있다.

- 브라우정의 소스코드 안에 CA의 리스트가 들어있다.
- 브라우저가 미리 파악하고 있는 CA의 리스트에 포함되어야만 공인된 CA가 될 수 있다.

## SSL의 동작 방법

- 결론부터 말하자면, SSL은 암호화된 데이터를 전송하기 위하여, **공개키와 대칭키를 혼합**해서 사용한다.
- 즉, 클라이언트와 서버가 주고 받는 실제 정보는 대칭키 방식으로 암호화 하고, 대칭키 방식으로 암호화된 실제 정보를 복호화할 때 사용할 "대칭키"는 공개키 방식으로 암호화해서 클라이언트와 서버가 주고 받는다.
    - 실제 데이터 : 대칭키가 해석한다
    - 대칭키 : 공개키가 해석한다.
    - 즉, 데이터를 대칭키가 암호화하고 그 대칭키를 공개키가 암호화 한다는 말이다.
- 실제 http통신에서 SSL의 과정을 알아보자.

---

### 악수 (handshake)

- 핸드쉐이크 과정을 통해 서로 상대방이 존재하는 지
- 상대방과 데이터를 주고 받기 위한 방식을 결정한다.
- SSL 방식을 이용하여 통신할 때도 이 handshake 방식을 사용한다.
    - 이 때,SSL 인증서를 주고 받는다.
- 공개키는 이상적인 통신 방법이다. 암호화와 복호화를 할 때 사용하는 키가 서로 다르기(공개키, 비공개키) 때문에, 공개키로 정보를 암호화하고 수신자가 비공개키를 이용해 정보를 복호화 하면 되기 때문이다.
- 하지만 SSL에서는 이 방식을 사용하지 않는다.
- 그 이유는, 공개키 방식의 암호화는 매우 많은 컴퓨터 자원을 사용하기 때문이다.
- 반면, 암호화와 복호화를 할 때 사용되는 키가 같은 방식인 대칭키는 적은 컴퓨터 자원으로 암호화 및 복호화를 수행할 수 있다.
    - 그렇지만 수신측과 송신측이 동일한 키를 가지고 있어야 하기 때문에 보안의 문제가 발생한다.
- 따라서 SSL 통신을 할 때는 이 공개키와 대칭키의 장점만을 혼합한 방식을 사용한다.
1. 클라이언트가 서버에 접속한다. 이 단계를 Client Hello 라고 한다. (나 왔어)
    - 이 단계에서 주고받는 정보는 다음과 같다.
        - 클라이언트측에서 생성한 랜덤 데이터 : 3번 참조
        - 클라이언트가 지원하는 암호화 방식들
            - 클라이언트 및 서버가 서로 지원하는 암호화 방식이 다를 수 있다.
            - 따라서 협상하기 위해 먼저 클라이언트가 지원 가능한 암호화 방식을 알려주어야 한다.
        - 세션 아이디
            - 이미 handshake를 한 client와 server라면, 자원을 절약하기 위해 기존의 세션을 재활용한다.
            - 이 때 사용할 연결에 대한 식별자를 서버 측으로 전송한다.
2. 서버는 Client Hello에 대한 응답으로 Server Hello(어 왔니) 를 하게 된다. 
    - 이 단계에서 주고받는 정보는 다음과 같다.
        - 서버 측에서 생성한 랜덤 데이터 : 3번 참조
        - 서버가 선택한 클라이언트의 암호화 방식
            - 클라이언트가 전달한 암호화 방식 중에서 서버 쪽에서도 사용할 수 있는 암호화 방식을 선택해서 클라이언트에 전달한다.
        - 인증서
            - 서버가 비공개 키로 암호화 시킨 인증서.
3. 검증절차
    - 클라이언트는 서버가 준 인증서가 CA에 의해서 발급된 것인지 확인해야한다.
        - 이 작업을 클라이언트의 브라우저가 해준다.
        - 만약 CA 리스트에 인증서가 없다면 사용자에게 경고 메세지를 출력한다.

            ![SSL%20TLS%2088ee8f96df594b46821fc7c16aa42cb6/Untitled%205.png](SSL%20TLS%2088ee8f96df594b46821fc7c16aa42cb6/Untitled%205.png)

        - 인증서가 CA에 의해 발급된 것인지 확인하기 위해서는 다음과 같이 한다.
            - 클라이언트에 내장된 CA의 공개키를 이용해서 전달된 인증서를 복호화 한다.
            - 복호화에 성공했다면, CA의 비공개키(개인키)로 암호화된 문서임이 암시적으로 보증된 것이다.
    - 클라이언트는 상기 2번을 통해 받은 서버의 랜덤데이터와 자신이 생성한 랜덤데이터를 조합해서 pre master secret이라는 키를 생성한다.
        - 이 키는 뒤에서 살펴볼 세션 단계에서 데이터를 주고 받을 때 암호화하기 위해서 사용될 것이다.
        - 이 때 사용할 암호화 기법은 대칭키이기 때문에, pre master secret 값은 **제 3자에게 절대 노출되어서는 안된다.**
    - 그럼 문제는 이 pre master secret 값을 어떻게 서버에 전달할 것인가 이다.
        - 이 때 사용하는 방법이 바로 공개키 방식이다. 인증서를 전달할 때와 같다.
        - 서버(CA)의 공개키로 pre master secret 값을 암호화해서 서버로 전송하면, 서버는 자신의 비공개키(개인 키)로 안전하게 복호화 할 수 있다.
4. 서버는 클라이언트가 전송한 pre master secret 값을 자신이 가지고있는 비공개키로 복호화 한다.
    - 이로써 서버와 클라이언트가 모두 동일한 pre master secret 값을 가지게 되었다.
    - 그리고 서버와 클라이언트는 모두 일련의 과정을 거쳐 pre master secret 값을 master secret 깞으로 만든다.
    - master secret은 session key를 생성하는 데, 이 session key 값을 이용해서 서버와 클라이언트는 데이터를 대칭 키 방식으로 암호화 한 후에 주고 받는다.
        - 이렇게 해서 세션키를 클라이언트와 서버가 모두 공유할 수 있게 된다.
5. 클라이언트와 서버는 핸드쉐이크 단계의 종료를 서로에게 알린다.

### 세션 (Session)

- 세션은 실제로 서버와 클라이언트가 데이터를 주고받는 단계이다.
- 이 단계의 핵심은 정보를 상대방에게 전송하기 전에 session key 값을 이용해서 대칭키 방식으로 암호화 한다는 점이다. 암호화된 정보는 상대방에게 전송될 것이고, 상대방도 세션키 값을 알고 있기 때문에 복호화 할 수 있다.
- 그냥 공개키 방식을 사용하면 될 텐데, 대칭키와 공개키를 조합해서 사용하는 이유는 다음과 같다.
    - 공개 키 방식이 많은 컴퓨터 자원을 소모하기 때문이다.
    - 만약 공개키를 그대로 사용하면, 많은 트래픽이 몰리는 서버는 매우 큰 비용을 지불해야 할 것이다.
    - 반대로, 대칭키는 암호를 푸는 열쇠인 대칭키를 상대에게 전송해야 하는데, 암호화가 되지 않은 대칭키를 전송하는 것은 위험하다.
    - 따라서, 속도는 느리지만 데이터를 안전하게 주고받을 수 있는 공개키 방식으로 대칭키를 암호화 하고, 실제 데이터를 주고받을 때는 대칭키를 이용해서 데이터를 주고 받는다.

### 세션 종료

- 데이터의 전송이 끝나면 SSL 통신이 끝났음을 서로에게 알려준다.
- 이 때 통신에서 사용한 대칭키인, 세션키를 폐기한다.