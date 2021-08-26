---
layout: post
section-type: post
title: Method of Calculating a Governing  Equation by Lagrangian Dynamics Using Python
category: dynamics and control
tags: [ 'python']
use_math: true
---

로봇에 대해 공부하다보면 정형화된 6축 매뉴퓰레이터가 아닌 다양한 로봇을 설계할 때가 있다. 

프로젝트로 진행하고 있는 공모양 로봇이 그중 하나이다. 

이러한 로봇을 제어하는 로직을 짜거나 칼만필터를 사용하기 위해서는 로봇을 동역학(Dynamics)적으로 해석할 필요가 있다.

로봇은 여러게의 요소가 연결되어 있어서 다물체 동역학(Multibody Dynamics)이다.

이러한 다물체로 이루어진 시스템을 해석하는 방법으로는 가장 기본적으로 쓰이는 뉴턴방식, 가상일 (Virtual Works), 라그랑주 역학(Lagrangian Dynamics)의 방식이 있다.

뉴턴방식은 간단한 시스템에서 주로 쓰이고 로봇과 같은 복잡한 시스템은 가상일과 라그랑주 역학으로 해석을 하게 된다.

그중 라그랑주 역학은 내력을 구할 필요가 없는 장점이 있다.

우리의 시스템도 라그랑주 역학으로 해석하기로 했다.

# Lagrange Equation
라그랑주 역학에 대해서 간략히 설명하겠다.

$$ \frac{d}{dt} \cfrac{\partial L }{\partial \dot{q}}-\cfrac{\partial L }{\partial q}=Q  $$

그렇다면 우리의 시스템에 요소에 해당하는 에너지항을 구해보자 


## Energy Term
![RobotSystem](/img/lagrangeanalysis.jpg )
본 그림은 로봇의 시스템을 간략하게 그린 그림이다.

## Code
```python
from sympy import diff, Function, symbols, cos, sin, latex, simplify
from sympy.physics.mechanics import *

# variable define
ms, mi, mp, Rs, Js,  Jp, t, g, lp, rpm = symbols("m_{s} m_{i} m_{p} R_{s} J_{s} J_{p} t g l_{p} r_{pm}")
"""
    m_{s}= mass of Shell
    m_{i}= mass of IDU
    m_{p}= mass of pendulums
    R_{s}= Radius of Shell (Scala)
    J_{s}= inertia of Shell
    J_{i}= inertia of IDU
    J_{p}= inertia of pendulums (rotational axis)
    g = gravity accelation (scala)
    l_{p}= length of pendulums C.M to rotational Axis
    \\theta_{s}= angle of Shell (Global Frame)
    \\theta_{im}= angle of IDU motor (Shell Frame)
    \\theta_{pm}=angle of pendulum motors (IDU Frame)

    \\theta_{i}=\\theta_{im}+\\theta_{s}                        (Global Frame)
    \\theta_{p}=\\theta_{im}+\\theta_{s}+\\theta_{pm}           (Global Frame)
"""

th_s,  th_p = dynamicsymbols("\\theta_{s} \\theta_{p}")
# Energy Equation
Ts = 1 / 2 * ms * (Rs * th_s.diff()) ** 2 + 1 / 2 * Js * th_s.diff() ** 2
Ti = 1 / 2 * mi * (Rs * th_s.diff()) ** 2
Tp = 1 / 2 * mp * (Rs * th_s.diff()) ** 2 + 1 / 2 * Jp * th_p.diff() ** 2
T = Ts + Ti + Tp
U = -mp * g * lp * cos(th_p)-mp*g*rpm
L = T - U
LM = LagrangesMethod(L, [th_s, th_p])
print(latex(simplify(LM.form_lagranges_equations())))
```