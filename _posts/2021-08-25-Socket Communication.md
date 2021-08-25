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


# Client.h
먼저 클라이언트인 라즈베리파이에서의 코드이다. 
클라이언트인 라즈베리파이는 센서에서 데이터를 직접 받는 컴퓨터이고 연결된 센서는 E2box EBIMU 9 DOF v4(이하 AHRS)와 Autonomics E50S8-3600-3-T-5(이하 Encoder[엔코더])

<pre><code data-trim class="yml">
#pragma once
#include<iostream>
#include<string.h>
#include<wiringPi.h>
#include<wiringSerial.h>
#include<sstream>
#include<thread>
#include<unistd.h>
#include<arpa/inet.h>
#include<sys/socket.h>
#include<netdb.h>
#include<raspicam/raspicam_cv.h>
#include<opencv2/imgproc.hpp>
#include<opencv2/opencv.hpp>
#include<math.h>
using namespace std;
using namespace cv;
//SENSOR
#define PhaseA 21 //Encoder A
#define PhaseB 22 //Encoder B
#define TILTPIN 26
#define  BUFF_SIZE 64

float encoder_pulse = (float)360 / (3600 * 4)*1000;;
float angle = 0;
int encoder_pos = 0;
bool State_A = 0;
bool State_B = 0;

void Interrupt_A() {
    //INTERRUPT pin A [ENCODER]
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
    //INTERRUPT pin B [ENCODER]
	State_A = digitalRead(PhaseA);
	State_B = digitalRead(PhaseB);

	if (State_B == 1) {

		if (State_A == 1) {
			encoder_pos++;                    // rotate direction : CW
			angle += encoder_pulse;
		}
		else {
			encoder_pos--;                    // rotate direction : CCW
			angle -= encoder_pulse;
		}
	}

	else {
		if (State_A == 0) {
			encoder_pos++;                    // rotate direction : CW
			angle +=  encoder_pulse;
		}
		else {
			encoder_pos--;                    // rotate direction : CCW
			angle -=  encoder_pulse;
		}
	}
}
class Client{
    private:
        int client=0;
        int AHRS=0;//Serial
        int Nano=0; //NANO
        struct sockaddr_in server_addr;//
        string msgSend="";
        struct hostent *he;
        float tiltAngle=0;
        string RPY="";
        uint32_t past = 0;
        uint32_t now = 0;
        uint32_t controlPeriod = 20; //20ms
        uint32_t time= now-past;
        float angularVel = 0;
        float anglePast = angle;
        float angleNow = angle;
        int penL=0;
        int penR=0;
        int idu=0;
        int tilt=0;
        string CMD="";
	bool recv_sock=0;

    public:
        Client(const char *hostname, const int _port)
        {   
            he=gethostbyname(hostname); // server url
            client = socket( AF_INET, SOCK_STREAM, 0);
            server_addr.sin_family = AF_INET;
            server_addr.sin_port = htons(_port);
            server_addr.sin_addr.s_addr=*(long*)(he->h_addr_list[0]);
            if(client==-1){
                cerr<< "\n Socket creation error \n";
                CMD="q";
		exit(1);
            }
            if (connect(client, (struct sockaddr *)&server_addr, sizeof(server_addr)) < 0){    
                cerr<<"\nConnection Failed \n"; 
                CMD="q";
		exit(1);
            }
        }
        void startClient(const char *_AHRSport,int _AHRSbaud, const char *_NanoPort,int _NanoBaud){
            //Client RPI Start
            pinMode(TILTPIN,INPUT);
            pinMode(PhaseA, INPUT);
	        pinMode(PhaseB, INPUT);
	        wiringPiISR(PhaseA, INT_EDGE_BOTH, &Interrupt_A);
	        wiringPiISR(PhaseB, INT_EDGE_BOTH, &Interrupt_B);
            if((AHRS = serialOpen(_AHRSport,_AHRSbaud))<0){
                cerr<<"Unable to open AHRS"<<endl;
                exit(1);
            }
            if((Nano=serialOpen(_NanoPort,_NanoBaud))<0){
                cerr<<"Unable to open Arduino"<<endl;
                exit(1);
            }
            initNano();
            runClient();
        }
        void AHRSread(){   
            //Read SENSOR
            int rawdata;
            string data;
            do{
                rawdata=serialGetchar(AHRS);
                data+=(char)rawdata;
            }while(rawdata!=10);//init ASCII 42 == "*"
            RPY=data;
	        RPY.pop_back();
	        RPY.pop_back();
        }
        
        void initNano(){
            //CONNECT ARDUINO [INIT..]
            int rawdata;
            string data="";
            do{
                rawdata=serialGetchar(Nano);
                data+=(char)rawdata;
            }while(rawdata!=42);
            cout<<data<<endl; //"Arduino is Ready*"
            //init IDU Stable...
            //####################################
            CMD="1500";

            serialPuts(Nano,CMD.c_str());//fake CMD
            //#######################################
            data="";
            do{
                rawdata=serialGetchar(Nano);
                data+=(char)rawdata;
            }while(rawdata!=64);
            cout<<data<<endl; //"KHU KongBot2 is Ready@"
        }
        void CAM(){
            //CAMERA RECORD [THREAD 2]
            raspicam::RaspiCam_Cv Camera;
            cv::Mat image;
            Camera.set( CV_CAP_PROP_FORMAT, CV_8UC3);
            Camera.set( CV_CAP_PROP_FRAME_WIDTH, 320 );
            Camera.set( CV_CAP_PROP_FRAME_HEIGHT, 240);
            if (!Camera.open()) {
                cerr<<"Error opening the camera"<<endl; // Connect Camera..
                return;
            }
            cv::VideoWriter outputVideo;
            cv::Size frameSize(320,240);
            int fps = 25;
            outputVideo.open("output.avi", cv::VideoWriter::fourcc('X','V','I','D'),fps, frameSize, true);
            if (!outputVideo.isOpened())
            {
                cout  << "Could not open the output video for write: " <<"output.avi" << endl;
                return;
            }
            do{
                Camera.grab();
                Camera.retrieve ( image);
                cv::cvtColor(image, image, cv::COLOR_BGR2RGB);
                cv::imshow( "test", image );
                outputVideo.write(image);

            } while(CMD!="q");
            Camera.release();
        }
        void runClient(){
	        msgSend="Complte";
	        send(client,msgSend.c_str(),msgSend.size(),0);
            now = past = millis();
	        anglePast = angle;
	        angleNow = angle;
            thread sockReceive([&](){receive_send();});
            thread camera([&](){CAM();});
            camera.detach();
            sockReceive.detach();
            do{ 
                //fout<<roll<<"\t"<<pitch<<"\t"<<yaw<<angle<<angleVel<<endl
                tiltRead();
                AHRSread();
                if((time)>controlPeriod){
		            angleNow = angle;
		            angularVel = (angleNow-anglePast)/(time) * 1000;
		            past=millis();
                    time= now-past;
		            anglePast=angle;
	        	}
		        now=millis();//FUNCTION SENSOR NEED
                //cout << "Encoder Pos : " << encoder_pos << "\tAngle : " << angle << "\t Vel : " << angularVel << "\n";
                msgSend=RPY+","+to_string(angle)+","+to_string(tiltAngle)+"\n";
                send(client,msgSend.c_str(),msgSend.size(),0);
		        //cout<<msgSend<<endl;
		        if(recv_sock){
		            cout<<CMD<<"time: "<<now<<endl;
		            recv_sock=0;
		        }
		
            } while (CMD!= "q");
            //cout << "quit" << endl;  
            quitClient();
        }
        void quitClient(){
            cout<<"Close.."<<endl;
            serialClose(Nano);
            serialClose(AHRS);
            close(client);
            cout<<"OFF"<<endl;
            exit(1);

        }
        void receive_send(){
            //Socket INPUT and SEND Nano [THREAD 3]
            char buffer[BUFF_SIZE]={0};
            do {
                read(client,buffer,BUFF_SIZE);
                CMD=buffer;
		        if(CMD.size()){
		            CMD=CMD.substr(0,CMD.find('\n'));
		            serialPuts(Nano,CMD.c_str());
		            buffer[0]={0,};
                    
		            recv_sock=1;
		        }
            } while (CMD != "q");
        }
        void tiltRead(){
            int rawdata;
            string data="";
            do{
                rawdata=serialGetchar(Nano);
                data+=(char)rawdata;
            }while(rawdata!=42);
            tiltAngle=stof(data);
        }
};

</code></pre>