---
layout: post
section-type: post
title: Using C++ for MultiThread processing 
category: KHU-KongBot
tags: [ 'RaspberryPi' ,'c++']
---
요즘 개인적으로 라즈베리파이로 학부 졸업논문으로 만들었던 공 로봇을 다시 처음부터 만들어서 제대로 만들어보려고 하고 있다,

그런데 로봇이 완전히 스스로 판단하는 것이 아닌 조종을 사람이 해줘야한다.

만약  키보드로 입력을 받아서 이를 사용하게 되는 코드를 짜서 사용하게 되면 입력이 끝나기 전까지는 다음 줄로 넘어가지 않는다. 즉

계속 백그라운드로 진행되어야 하는 로봇의 자세 제어, 센서 데이터 읽기 등이 작동을 하지 않게 되어 제어를 할수 없게된다.

따라서 이를 해결하기 위해서 제시하는 방법은 명령어만 입력받는 스레드를 만드는 것이다.



그것을 하기 앞서서 스레드를 간략히 설명하겠다.



스레드는 같은 프로세스내에서 병렬로 계산할수 있게끔 만들어주는 것이다. 스레드는 같은 프로세스 내에서 동작하기 때문에 메모리를 공유하게 된다.

따라서 스레드 내에서 바뀐 데이터는 다른 스레드도 알수 있게 된다는 것이다.

그렇다면 메인스레드가 진행하는 도중에서도 넣어준 입력을 통해 메인스레드에서 진행되는 것이 바뀌게 만들수 있다는 것이다.

즉 명령어를 입력받더라도 그외 동작들은 진행이 되게 만들수 있다는 것이다.





다음은 q가 입력되기전에 계속돌다가 입력된 문자에 따라 출력이 바뀌는 코드이다.

CMD 가 입력값이다. 이 CMD가 w a s d W S j k q 에따라 메인 스레드가 출력하는 문자열이 다르다. 

스레드를 사용하기전에는 입력이 있을때까지 출력을 하지 않았는데 이렇게 코드를 작성하면 입력을 다른 스레드로 처리해서 계속 메인이 돌고 있음을 확인할수 있다.
<pre><code data-trim class="yml">
thread inputCMD(&input, ref(CMD));//INPUT command Thread.....
inputCMD.detach();
</code></pre>

여기서 thread inputCMD(&input,ref(CMD));
를 뜯어 보자면 

inputCMD를 만드는데 이쓰레드에서 돌릴 함수는 input이라는 함수를 레퍼런스로 받고 이함수에서 리턴되는 값을 CMD의 레퍼런스로 받을 것이야 라는 뜻이고
inputCMD.detach();
는 inputCMD를 부모스레드인 메인에서 완전히 독립시킨다는 뜻이다.  따라서 inputCMD라는 스레드따로 메인 따로 코드가 굴러간다.

만약 자식쓰레드인 inputCMD가 종료될때까지 기다리는것을 하고 싶으면 inputCMD.join()을 사용하면 된다.

<pre><code>
#include&lt;thread&gt;
#include&lt;iostream&gt;
using namespace std;
void input(char &CMD) {
    do {
        cin >> CMD;
    } while (CMD != 'q');
    
}
 
int main()
{
    char CMD;
    thread inputCMD(&input, ref(CMD));//INPUT command Thread.....
    inputCMD.detach();
    do{
        switch (int(CMD)){
            case 119 :
                //CMD=w
                cout<< "go\n";
                break;
            case 115 :
                //CMD=s
                cout<<"back\n";
                break;
 
            case 87:
                //CMD=W
                cout<< "GO\n";
                break;
            
            case 83:
                //CMD=S
                cout<< "BACK\n";
                break;
            
            case 97:
                //CMD=a
                cout<< "chage roll - direction \n";
                break;
            case 100:
                //CMD=d
                cout<< "chage roll + direction \n";
                break;
            case 106:
                //CMD=j
                cout<< "chage yaw  -15 degree direction \n";
                break;
            case 107:
                //CMD=j
                cout<< "chage yaw  +15 degree direction \n";
                break;
 
        }
 
    } while (CMD != 'q');
    cout << "quit" << endl;
//방법 1
//join()을 실행시키면 t가 종료되기 전까지 기다린다.
    return 0;
}

</code></pre>
