---
layout: post
section-type: post
title: Calculate system by openCV mat class[cv::Mat] {C++}
category: KHU-KongBot
tags: [ 'c++','Linux[ubuntu]']
---

KHU-Kongbot2 Project를 진행하면서 객체지향적 요소를 왠만하면 넣지 않을라고 했다. 이유는 함께하는 기계과 친구들이 객체지향보다 절차지향적인 코드에 익숙했기 때문이다.

하지만 계속 진행하다보니 코드가 점점 꼬였다. 

실제로 각각의 요소별로 같은 코드로 묶이는 것들도 많았고 요소별로 다른 함수를 작성하고 변수를 일일이 설정해줘야해서 코드가 복잡해지다보니 알아보기도 힘들고 코드길이가 많이 길어졌다.

따라서 이를 해결하기 위해 프로그램을 절차지향에서 객체지향으로 다시 짜기로 마음 먹었다.

또한 객체지향으로 바꾸면서 기존의 로봇의 현재의 state를 계산하고 이에 따른 제어 input을 계산하는 헤더파일을 만들었다.

만들어진 헤더파일의 크기와 계산양을 보니 라즈베리파이4에서 이를 계산하고 시리얼통신을 통해 아두이노로 내려주는 것까지 하기엔 너무 리소스가 모잘라 보였다.

결국이 제어 입력과 현재의 상태를 계산하는 코드를 서버에서 돌리기로 했고  라즈베리파이4와는 소켓통신으로 연결해주기로 하였다.

리소스를 생각하기 때문에 모든 코드는 C++로 작성하는 중이었으니 이를 유지하기로 했다.

이번에 그 소켓통신 공부하면서 간단히 테스트용으로 만든 소켓통신 코드를 포스팅하기로 한다.


# Client
## client.h

```c
#pragma once
#include<iostream>
#include<string.h>
#include<thread>
#include<unistd.h>
#include<arpa/inet.h>
#include<sys/socket.h>
#include<netdb.h>
using namespace std;
#define BUFF_SIZE 8
class Client{
    private:
        int client=0; 
        struct sockaddr_in server_addr; //서버의 주소가 저장되는 구조체
        string msgReceive="";//받은 메세지 저장
        string msgSend="";//보낸 메세지
        struct hostent *he;//server가 URL주소일 경우 일차적으로 hostent 구조체의 이름으로 넣어짐.
    public:
        Client(const char *hostname, const int _port);//client 생성자 prameter로 서버주소와 포트를 넣어줌
        void runClient();//client시작 
        void receive_send();//서버와 주고 받는 쓰레드
};
```

여기서 중요한 Keypoint는 hostent 이다.

보통 서버의 주소로는 Const char* 를 사용하는데 이 경우 서버가 "127.0.0.1"과 같은 직접적인 IPv4주소를 아래와 같게 직접 써준다.
```c
server_addr.sin_addr.s_addr=inet_addr("127.0.0.1");
```

하지만 이경우 서버의 주소가 "https://snovvyowl.github.io"와 같은경우는 대처할 수가 없다 따라서 이를 hostent 구조체를 써서 해결한다.
```c
struct hostent *he; //헤더에 정의함
he=gethostbyname(hostname);//생성자에 정의함
server_addr.sin_addr.s_addr=*(long*)(he->h_addr_list[0]);//생성자에 정의함
```

이렇게 하면 문제가 되는 서버의 주소도 처리가 가능해진다.

## client.cpp
```c
#include<client.h>
using namespace std;
#define BUFF_SIZE 8
Client::Client(const char *hostname, const int _port)
{   
    he=gethostbyname(hostname); // server url
    client = socket( AF_INET, SOCK_STREAM, 0);
    server_addr.sin_family = AF_INET;
    server_addr.sin_port = htons(_port);
    server_addr.sin_addr.s_addr=*(long*)(he->h_addr_list[0]);
    if(client==-1){
        cerr<< "\n Socket creation error \n";//소켓이 만들어지지 않았을 경우 프로그램 종료
        msgReceive="quit";
	    exit(1);
    }
    if (connect(client, (struct sockaddr *)&server_addr, sizeof(server_addr)) < 0){    
        cerr<<"\nConnection Failed \n"; //서버와 연결되지 않았을 경우 프로그램 종료
        msgReceive="quit";
        exit(1);
    }
    runClient();//runClient함수 호출
}
       
void Client::runClient(){
    thread sockReceive([&](){receive_send();});//소켓통신을 통해 데이터만 주고 받는 쓰레드
    int i=0;
    sockReceive.detach();//메인 쓰레드와 생성된 쓰레드가 완전히 독립적으로 작동함.
    do{ 
        msgSend=to_string(i)+"\n";
        send(client,msgSend.c_str(),msgSend.size(),0); //서버로 i를 보냄
        cout<<msgReceive<<endl;//서버에서 보낸 메세지 출력
    } while (msgReceive!= "q");//서버에서 받은 데이터가 q로 시작될경우 종료
    close(client); //보통 서버가 먼저 꺼지면 프로그램이 자동 종료 된다.
}

void Client::receive_send(){
    //Socket receive[THREAD 2]
    char buffer[BUFF_SIZE]={0};//서버에서 보낸 데이터를 받을 버퍼
    do {
        read(client,buffer,BUFF_SIZE);//받은 데이터를 버퍼에 저장
        msgReceive=buffer;//버퍼를 string으로 변환
        buffer[0]={0,};//버퍼 초기화
    } while ( msgReceive != "q");
}
```

여기서 read()와 send()는 정의에 따라 cstring 형태로 데이터를 주고 받는다 따라서 익숙한 string 형태를 쓰기 위해 보낼때는 받은 문자열 배열을 string 멤버 변수에 저장을 한다.

그리고 보낼때는 string을 .c_str()과 같은 메서드로  변환해서 보내준다.
## main.cpp
```c
#include<client.h>
int main(){
    Client client("127.0.0.1",13000);
    return 0;
}
```

이제 server쪽을 알아보도록 하자.

# Server
## server.h
```c
#pragma once
#include <iostream>
#include <stdio.h>
#include <string>
#include <unistd.h>
#include <arpa/inet.h>
#include <sys/socket.h>
#include <thread>
#include <netdb.h>
#define BUFF_SIZE 8
using namespace std;
class Server
{
private:
    char *ip;//서버 IP주소
    int port = 13000;//포트번호
    int server = 0;//서버 소켓
    int client = 0;//클라이언트 소켓
    struct sockaddr_in client_addr;//클라이언트 소켓 주소  구조체
    struct sockaddr_in server_addr;//서버 소켓 주소 구조체
    char CMD;//키보드 입력 명령어
    socklen_t client_addr_len = sizeof(client_addr);//소켓주소 길이
    string msgReceive = "";//클라이언트에서 보낸 데이터
    string msgSend = "";//클라이언트로 보낼 데이터
public:
    Server(const char *_ip, int _port);//생성자
    void startServer();//서버시작 메서드
    void runServer();//서버 동작중 메서드
    void receiving();//서버에서 데이터 보내는 메서드
    void sending();//클라이언트에서 데이터 받는 메서드
    void keyboardInput();//키보드에서 입력 받는 함수
    void stopServer();// 서버 종료 메서드
};
```
소켓서버는 클라이언트와 계속 내용을 주고 받는다는 가정하에 서버가 강제 종료가 필요할 경우에 이를 하기 위해 키보드 입력 함수를 구현하여 다른 Thread를 이용하여 동작 하려고 한다.

또한 데이터를 보내는 것도 다른 쓰레드로 구성하고 받는 것도 다른 쓰레드로 구성하여 메인을 포함하여 4개의 쓰레드가 동시 동작을 할수 있도록 하자.


```c
#include <server.h>
#define BUFF_SIZE 8
using namespace std;

Server::Server(const char *_ip, int _port)
{
    ip = (char *)_ip;
    server = socket(AF_INET, SOCK_STREAM, 0);//서버소켓 생성
    if (server == -1)
    {
        cerr << "\n Socket creation error \n";//소켓 생성 에러
        //sock_receive="Quit";
        exit(1);
    }
    server_addr.sin_family = AF_INET;
    server_addr.sin_port = htons(port);
    server_addr.sin_addr.s_addr = htonl(INADDR_ANY);
    startServer();//서버 시작
}
void Server::startServer()
{
    cout << "[Start Server]\n";
    cout << "Server ip -> " << ip << endl;
    cout << "Server port -> " << port << endl;
    if (bind(server, (struct sockaddr *)&server_addr, sizeof(server_addr)) < 0)//소켓 서버 바인딩
    {
        cerr << "Bind ERROR" << endl;//연결이 부정확할 경우 종료
        //sock_receive="Quit";
        exit(1);
    }
    if (listen(server, 1) < 0)
    {
        cerr << "Listen ERROR" << endl;// 소켓연결을 할수 있도록 열음 
        //sock_receive="Quit";
        exit(1);
    }
    cout << "Wait...\n";
    client_addr_len = sizeof(client_addr);
    client = accept(server, (struct sockaddr *)&client_addr, &client_addr_len);//클라이언트가 연결될때까지 기다힘
    printf("Connection from: %s\n", inet_ntoa(client_addr.sin_addr));
    char buffer[BUFF_SIZE] = {0};
    runServer();//서버 시작
}
void Server::runServer()
{
    thread inputCMD([&](){ keyboardInput(); });//키보드 입력쓰레드
    thread sockReceive([&](){ receiving(); });//클라이언트에서 받는 쓰레드
    thread sockSend([&](){ sending(); });//클라이언트로 보내는 쓰레드
    //각자 메인쓰레드와 따로 놈
    sockReceive.detach();
    inputCMD.detach();
    sockSend.detach();
    do
    {
        cout << msgReceive;//받은 메세지 출력
        switch (int(CMD))
        {
        // 키보드 입력에 따라 다르게 출력되는 값을 보여주기 위해서 만든 스위치 케이스문이다.
        case 119:
            //CMD=w
            cout << "go\n";

            break;

        case 115:
            //CMD=s
            cout << "back\n";

            break;

        case 87:
            //CMD=W
            cout << "GO\n";

            break;

        case 83:
            //CMD=S
            cout << "BACK\n";

            break;

        case 97:
            //CMD=a
            cout << "chage roll - direction \n";

            break;

        case 100:
            //CMD=d
            cout << "chage roll + direction \n";

            break;

        case 106:
            //CMD=j
            cout << "chage yaw  +15 degree direction \n";
            break;

        case 107:
            //CMD=k
            cout << "chage yaw  -15 degree direction \n";
            break;

        case 113:
            cout<<"stop"<<endl;
            break;

        default:
            //cout<< "Wrong OR EMPTY CMD\n";
            break;
        }

    } while (CMD != 'q');//키보드에서 입력값이 q일경우 서버를 종료한다.
    stopServer();
}

void Server::receiving()//소켓에서 데이터를 받는 함수 
{
    char buffer[BUFF_SIZE] = {0};
    // stringstream ss;
    do
    {
        read(client, buffer, BUFF_SIZE);
        msgReceive = buffer;
        if (msgReceive.size())
        {
            buffer[0] = {0,};
    } while (CMD != 'q');
    exit(1);
}

void Server::sending()//서버에서 보내는 함수
{
    do
    {
        //penR,penL,IDU,tilt
        msgSend = to_string(CMD)+"\n";
        send(client, msgSend.c_str(), msgSend.size(), 0);
    } while (CMD != 'q');
    exit(1);
}

void Server::keyboardInput()//키보드 인풋에 대한 함수 q를 입력받으면 종료
{
    //KEBOARD INPUT
    do
    {
        cin >> CMD;
    } while (CMD != 'q');
    exit(1);
}
void Server::stopServer()//종료 함수
{
    msgSend = "q";
    send(client, msgSend.c_str(), msgSend.size(), 0);
    cout << "[stop server]\n";
    close(client);
    close(server);
    cout << "OFF";
    exit(1);
}
```

지금은 클라이언트에서 0만 보내고 서버에서는 키보드 입력값만 보내고 있는 데 필요하다면 이를 수정하여 다른 방향을 사용 가능하다.
## main.cpp
```c
#include<server.h>
int main(){
    Server ser("127.0.0.1",13000);
    return 0;
}
```

지금 내가 진행하는 프로젝트에서는 thread를 이용하여 키보드입력과 소켓 통신이 별도로 구현되야 할 필요가 있고 이를 토대로 통신과 명령전달을 동시에 진행하고 있다.