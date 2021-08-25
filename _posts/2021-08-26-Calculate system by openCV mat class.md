---
layout: post
section-type: post
title: Calculate system by openCV mat class[cv::Mat] {C++}
category: KHU-KongBot
tags: [ 'c++','Linux[ubuntu]']
---

KHU-KongBot2 프로젝트를 진행하다보니 센서의 데이터와 실제 로봇의 모델을 계산해서 현재의 로봇의 상태를 추정하는 칼만필터를 사용할 필요가 생겼다.

하지만 c++ 에 기본 라이브러리에서는 Matrix 클래스는 존재 하지 않으니 실재로 구현을 해보았는데, 직접 짜고 보니 굉장히 무거워 보였다.

[직접 작성한 Matrix 클래스](https://github.com/SnovvyOwl/SampleCode/blob/master/Matrix.h)