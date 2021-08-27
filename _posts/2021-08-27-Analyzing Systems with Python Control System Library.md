---
layout: post
section-type: post
title: Analyzing Systems with Python Control System Library
category: dynamics and control
tags: [ 'python' ,'ubuntu']
use_math: true
---

우리 학교 기계공학과에서 각종 프로젝트를 진행하게 되면 각종 수치 계산을 포함해서 간단한 프로그래밍 전부 MATLAB에 의존하는 경우가 많다.
물론 MATLAB이 사용하기 쉽고 편하다는 사실은 부정할 수 없다. 하지만 MATLAB에겐 치명적인 단점이 있는데 이는 MATLAB이 유료라는 사실이다.\
각 종 라이브러리도 전부다 사야하고 필요한 툴박스를 구매하기 위한 비용을 생각 하지 않을 수 없다.\
결국 MATLAB이라는 유료 플랫폼을 거칠 수 없는 환경이 결국 왔을 때 이 한계는 크게 느껴질 것이라고 생각한다.

그래서 MATLAB을 사용하지 못하게 될때를 대비하기 위해 MATLAB의 대체제를 찾고 있는데 나는 이것을 Python이라고 생각한다.\
실제로 MATLAB의 Symbolic toolbox는 Python의 sympy로 대체 가능하고 그외 자주 사용하는 여러 MATLAB의 toolbox를 Python에서도 지원한다.\
나는 로봇의 움직임을 제어하는 것을 공부하는 기계공학과 학생이므로 MATLAB의 Control System Tool Box 를 대체할 Python libary를 찾았다.

그것을 바로 Python Control system library 이다.

# [Python Control Systems Library](https://python-control.readthedocs.io/en/0.9.0/)
## Install
```dotnetcli
pip3 install slycot   # optional
pip3 install control
```
control 만 설치해도 기본적인 기능은 돌아가지만 모든 기능을 하기 위해서는 slycot을 필수로 설치해주는 것을 권장한다.\
slycot은 control libaray를 wrapping 해주는 library인데 설치 할때는 Fortran compiler 가 필요하다.


```dotnetcli
sudo apt install gfortran   #ubuntu
```
ubuntu 환경에서 fortran Compiler를 아래와 같이 입력해주면 설치가 된다.


```dotnetcli
conda install numpy scipy matplotlib    # if not yet installed
conda install -c conda-forge control
```
Conda 환경이면 이렇게 install 하면 된다.


```python
import control
```
사용하는 방법은 control을 import 하면 된다.


## System creation
자주 쓰는 Method를 간략하게 포스팅하겠다.\
먼저 State Space represention 위한 ```control.ss()``` 이다

연속시간계인 $x$에서\
$\dot{x}=Ax+Bu$\
$y=Cx+Du$

```python
import numpy as np
A=np.array([[1., -2], [3.,-4]])
B=np.array([[5],[7]])
C=np.array([6,8])
D=np.array([9])
sys=control.ss(A,B,C,D)
```
불연속 시간계에서는\
$ x[k+1]=Ax[k]+Bu[k]$\
$y[k]=Cx[k]+Du[k]$

```python
dt=0.1
sys=control.ss(A,B,C,D,dt) #dt는 sampling Time
```
이고 이 메서드의 반환 값은 contorl.ss 의 형태로 반환된다.

두번째는 Transfer Function에 해당하는 ```tf()```이다.
만약 $\cfrac{s+1}{s^2+s+1}$ 에 해당하는 시스템이 있다고 하자\
그럼 여기서 numerator는 s+1이고 denominator는 $s^2+s+1$이다.\
이것을 파이썬으로 옮긴다면 아래와 같다.

```python
num=[1,1]
den=[1,1,1]
dt=0.1
sys_tf_continous=control.tf(num,den) #연속시간계
sys_tf_discrete=control.tf(num,den,dt) #불연속 시간계에서 dt는 sampling Time
```
이건 다른 사용법이하나 더있는데 아래와 같은 방법으로도 쓸수 있다.

```python
s=control.tf('s')
G=(s+1)/(s**2+s+1)
```
또한 공부를 하다보면 State Space 표현을 Transfer Function 표현으로 바꿔야 할때와 그 반대 상황으로 바꿔야 할때가 있다.
```python
st=control.ss2tf(sys) #State Space에서 Transfer Function으로 바꿈
ss=control.tf2ss(sys_tf_continuous) #Transfer Function에서 State Space로 바꿈
```
그때 쓸수 있는 메서드이다.




## Control system analysis
시스템을 해석하기 위해 다양한 선도를 그릴 필요가 있는데 대표적으로 root locus, Bode선도, Nyquist 선도등이 있다.


### Root Locus
```python
import matplotlib.pyplot as plt
rlist ,klist=control.root_locus(sys_tf_continous)
plt.plot(rlist,klist)
plt.show()
```
루트로커스 선도 그리는 방법이다.


### Bode 선도
```pyhton
sys = control.ss([[1, -2], [3, -4]], [[5], [7]], [[6, 8]], [[9]])
mag, phase, omega = control.bode(sys)
```
Bode 선도 그리는 방법이다.


### Nyquist 선도
```python
sys = control.ss([[1, -2], [3, -4]], [[5], [7]], [[6, 8]], [[9]])
count = control.nyquist_plot(sys)
```
State Space 에서이 시스템이 제어가능한지 관측가능한지 확인하기 위해 가제어성 행렬과 가관측성 행렬을 뽑을 필요가 있다.\
이때 사용가능한 명령어는 다음과 같다

### Controllabilty and Observability
```python
C = ctrb(A, B) # Controllabilty matrix
O = obsv(A, C) # Observability matrix
```
```C```와 ```O```의 rank가 각각 Full 일때 제어가능하고 관측 가능하다라고 말한다.


```python
control.lyap(A, Q, C=None, E=None) #Lyapunov equation
```
이 외에도 Lyapunov equation를 푸는 메서드가 있다.


## Control System Synthesis
state space에서 Control Gain K를 얻는 방법으로 Ankermann Method와 LQR Method가 있다.
```python
K = control.acker(A, B, poles) # Ackermann method
K, S, E = lqr(A, B, Q, R, [N]) # Linear quadratic regulator design
```
이것 역시 코드로 쉽게 계산할 수 있다.


## Input Output System
Step Input과 Impulse Input에 대한 반응은 tf 일때와 ss일때 둘 다 변환 없이 출력이 가능하다. 그리고 입력이 일정할경우에도 출력가능하다.
```python
T, yout = step_response(sys, T, X0) #step Responce
T, yout = impulse_response(sys, T, X0) # Impulse Responce
T, yout, xout = forced_response(sys, T, u, X0)# Const U
```
여러 Input을 주었을 때 Output이 출력해야 할때 도 있다.\
그렇다면 기존의 ```control.tf```와 ```control.ss``` 를 ```control.InputOutputsystem```으로 변환해줄 필요가 있다.

```python
tfiosys=control.tf2io(sys_tf_continuous) # Transfer Function을 Input Output System으로 변환
ssiossy=control.ss2io(sys) # State Space를 Input Output System으로 변환
```

이렇게 변환이 됬다면 Input ```U```를 Array 형태에 데이터가 일정하지 않고 그것에 따른 시간 ```T``` array가 있으면 이를 바탕으로 직접 입력에 따른 출력값을 얻을 수 있다.

```python
Y = control.input_output_response(tfiosys, t, u)
``` 

이렇게 간략하게 Python Control System Library를 정리해보는 것으로 포스팅을 마감한다.