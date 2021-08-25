---
layout: post
section-type: post
title: Socket Communication between Linux Computer and Raspberry Pi {C++}
category: KHU-KongBot
tags: [ 'Socket Communication','RaspberryPI']
---

KHU-Kongbot2 Project를 진행하면서 객체지향적 요소를 왠만하면 넣지 않을라고 했다. 이유는 함께하는 기계과 친구들이 객체지향보다 절차지향적인 코드에 익숙했기 때문이다.

하지만 계속 진행하다보니 코드가 점점 꼬였다. 

실제로 각각의 요소별로 같은 코드로 묶이는 것들도 많았고 요소별로 다른 함수를 작성하고 변수를 일일이 설정해줘야해서 코드가 복잡해지다보니 알아보기도 힘들고 코드길이가 많이 길어졌다.

따라서 이를 해결하기 위해 프로그램을 절차지향에서 객체지향으로 다시 짜기로 마음 먹었다.

또한 객체지향으로 바꾸면서 기존의 로봇의 현재의 state를 계산하고 이에 따른 제어 input을 계산하는 헤더파일을 만들었다.

만들어진 헤더파일의 크기와 계산양을 보니 라즈베리파이4에서 이를 계산하고 시리얼통신을 통해 아두이노로 내려주는 것까지 하기엔 너무 리소스가 모잘라 보였다.

결국이 제어 입력과 현재의 상태를 계산하는 코드를 서버에서 돌리기로 했고  라즈베리파이4와는 소켓통신으로 연결해주기로 하였다.

리소스를 생각하기 때문에 모든 코드는 C++로 작성하는 중이었으니 이를 유지하기로 했다.

이번에 그 소켓통신 공부하면서 간단히 테스트용으로 만든 소켓통신 코드를 포스팅하기로한다.

# Client.h

<pre><code data-trim class="yml">
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
        void quitClient();//client 종료
        void receive_send();//서버와 주고 받는 쓰레드
};
</code></pre>

# client.cpp
<pre><code data-trim class="yml">
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
        cerr<< "\n Socket creation error \n";
        msgReceive="quit";
	    exit(1);
    }
    if (connect(client, (struct sockaddr *)&server_addr, sizeof(server_addr)) < 0){    
        cerr<<"\nConnection Failed \n"; 
        msgReceive="quit";
        exit(1);
    }
    runClient();
}
       
void Client::runClient(){
    thread sockReceive([&](){receive_send();});
    int i=0;
    sockReceive.detach();
    do{ 
        msgSend=to_string(i)+"\n";
        send(client,msgSend.c_str(),msgSend.size(),0); 
        i++;
    } while (msgReceive[0] != 'q');
    //cout << "quit" << endl;  
    quitClient();
}
void Client::quitClient(){
    cout<<"Close.."<<endl;
    close(client); 
    cout<<"OFF"<<endl;
    exit(1);
}
void Client::receive_send(){
    //Socket INPUT and SEND Nano [THREAD 3]
    char buffer[BUFF_SIZE]={0};
    do {
        read(client,buffer,BUFF_SIZE);
        msgReceive=buffer;
        //serialPuts(Nano,CMD.c_str());
        buffer[0]={0,};
    } while ( msgReceive != "q");
}
</code></pre>
