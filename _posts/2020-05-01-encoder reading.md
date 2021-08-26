---
layout: post
section-type: post
title: Encoder sensing Code with Raspberry Pi
category: KHU-KongBot
tags: [ 'RaspberryPi' ,'c++']
---

[C/C++] 라즈베리파이로 엔코더의 각도측정 및 각속도 계산  Raspberry Pi

임베디드 소프트웨어와 하드웨어를 공부하다보면 종종 모터에 의한 링크와 바디의 각도 또는 각속도를 측정할 필요가 있다.

이를 위해서 주로 쓰는 센서로는 엔코더와 자이로센서를 사용한다.



그리고 지난번에 포스팅한 AHRS(https://blog.naver.com/owl0614/221854976614)를 사용하기도 한다.


[C/C++] E2BOX EBIMU 9DOF v4에서 Raspberry Pi로 읽어보자 (Serial통...
오늘은 E2BOX의 EBIMU 9DOF V4라는 AHRS센서를 라즈베리파이와 연결하여 데이터를 읽어보...
blog.naver.com

﻿이번엔 그중 엔코더 센서를 사용해보려고한다.

엔코더는 크게 인크리멘탈 엔코더와 앱솔루트 엔코더로 나뉜다.



인크리멘탈은 말그대로 처음 시작점이 존재하지 않고 측정을 시작한 지점부터 얼마만큼 돌았는가를 측정한다.

따라서 측정을 오래 진행하게되면 에러가 누적될 수는 있지만 앱솔루트 엔코더에 비해 저렴한 편이라 많이 사용한다



앱솔루트 엔코더는 처음시작점이 존재한다. 그 시작점으로 부터 얼마나 돌았는지를 측정하는 엔코더로 가격이 비싼 대신 시작점이 존재하기에 에러가 쌓여도 이를 보정할 수 있다는 장점이 있다.



이번 프로젝트에서도 로봇의 외피에 대한 각속도를 측정할 필요가 있어 엔코더를 구입했고 여러가지 조건에 부합되는 엔코더는 인크리멘탈 엔코더였다.

따라서, 이 글에서는 인크리멘탈 엔코더만을 다루겠다.



이번에 사용한 엔코더는 오토노믹스 사의 P/R이 3600인 엔코더이다.

Autonomics E50S8-3600-3-T-5



﻿2개의 상을 읽어서 회전방향까지 체크하기 위해 A상 핀을 21번 B상 핀을 22번으로 설정했다.
<pre><code data-trim class="yml">
#define PhaseA 21

#define PhaseB 22
</code></pre>


그리고 각속도를 구하기 위해서 20ms에 한번씩 상의 변화량을 구해서 이를 나눠줬다.
<pre><code data-trim class="yml">
uint32_t controlPeriod = 20
</code></pre>


중요한것은 엔코더를 쓰기 위해 인터럽트를 사용해야한다. 

인터럽트는 신호가 1에서 0으로 떨어질때를 체크하는 INT_EDGE_FALLING,

신호가 0에서 1로 올라갈때를 체크하는  INT_EDGE_RISING,

둘다 발생한것을 체크하는 INT_EDGE_BOTH​﻿ 가 있다.


<pre><code data-trim class="yml">
이번에는 INT_EDGE_BOTH를 이용하여 둘다 측정하는 것으로 하였다.

wiringPiISR(PhaseA, INT_EDGE_BOTH, &Interrupt_A);

wiringPiISR(PhaseB, INT_EDGE_BOTH, &Interrupt_B);
</code></pre>

<pre><code data-trim class="yml">
void Interrupt_A();

void Interrupt_B();
</code></pre>
는 인터럽트가 발생하면 구동되는 함수이다. 상의 변화에 따라 회전각을 측정한다.



20ms 동안 발생한 회전각을 이용해 각속도를 구하는 부분이다.

<pre><code data-trim class="yml">
 now = past = millis();
 float anglePast = angle;
 float angleNow = angle;
 while(1){
  if((now-past)>controlPeriod){
   angleNow =angle;
   vel = (angleNow-anglePast)/(now-past) * 1000;
   now = past=millis();
   anglePast=angle;
  }
  now=millis();
  cout << "Encoder Pos : " << encoder_pos << "\tAngle : " << angle << "\t Vel : " << vel << "\n";
 }
</code></pre>

따라서 코드를 정리하면 이렇게 된다.




```c
#include<iostream>
#include<wiringPi.h>
#define PhaseA 21
#define PhaseB 22
using namespace std;
 
uint32_t past;
uint32_t now;
uint32_t controlPeriod = 20; //20ms
double encoder_pulse = 0;
double angle = 0;
int encoder_pos = 0;
 
bool State_A = 0;
bool State_B = 0;
float vel=0;
void Interrupt_A();
void Interrupt_B(); 
 
int main(){
    wiringPiSetup();
    pinMode(PhaseA, INPUT);
    pinMode(PhaseB, INPUT);
    wiringPiISR(PhaseA, INT_EDGE_BOTH, &Interrupt_A);
    wiringPiISR(PhaseB, INT_EDGE_BOTH, &Interrupt_B);
    encoder_pulse = (float)360 / (3600 * 4) * 1000; //3600 PPR    
    now = past = millis();
    float anglePast = angle;
    float angleNow = angle;
    while(1){
        if((now-past)>controlPeriod){
            angleNow =angle;
            vel = (angleNow-anglePast)/(now-past);
            now = past=millis();
            anglePast=angle;
        }
        now=millis();
        cout << "Encoder Pos : " << encoder_pos << "\tAngle : " << angle << "\t Vel : " << vel << "\n";
    }
}
 
void Interrupt_A() {
    State_A = digitalRead(PhaseA);
    State_B = digitalRead(PhaseB);
    if (State_A == 1) {
        if (State_B == 0) {
            encoder_pos++;                    // 회전방향 : CW
            angle +=  encoder_pulse;
        }
        else {
            encoder_pos--;                    // 회전방향 : CCW
            angle -= encoder_pulse;
        }
    }
    else {
        if (State_B == 1) {
            encoder_pos++;                    // 회전방향 : CW
            angle += encoder_pulse;
        }
        else {
            encoder_pos--;                    // 회전방향 : CCW
            angle -= encoder_pulse;
        }
    }
}
 
void Interrupt_B() {
    State_A = digitalRead(PhaseA);
    State_B = digitalRead(PhaseB);
    if (State_B == 1) {
 
        if (State_A == 1) {
            encoder_pos++;                    // 회전방향 : CW
            angle += encoder_pulse;
        }
        else {
            encoder_pos--;                    // 회전방향 : CCW
            angle -= encoder_pulse;
        }
    }
    else {
        if (State_A == 0) {
            encoder_pos++;                    // 회전방향 : CW
            angle +=  encoder_pulse;
        }
        else {
            encoder_pos--;                    // 회전방향 : CCW
            angle -=  encoder_pulse;
        }
    }
}
```




라즈베리파이에서 컴파일 할때 

g++ -o Encoder Encoder.cpp -lwiringPi   로 컴파일하면 된다. 


