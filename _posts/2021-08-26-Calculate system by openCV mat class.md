---
layout: post
section-type: post
title: Calculate system by openCV mat class[cv::Mat]
category: KHU-KongBot
tags: [ 'c++','ubuntu']
---

KHU-KongBot2 프로젝트를 진행하다보니 센서의 데이터와 실제 로봇의 모델을 계산해서 현재의 로봇의 상태를 추정하는 칼만필터를 사용할 필요가 생겼다.

하지만 c++ 에 기본 라이브러리에서는 Matrix 클래스는 존재 하지 않으니 실재로 구현을 해보았는데, 직접 짜고 보니 굉장히 무거워 보였다.

[직접 작성한 Matrix 클래스](https://github.com/SnovvyOwl/SampleCode/blob/master/Matrix.h) 수치해석적인 inv() 메서드도 열심히 구현해 보았지만 조금 더 최적화 잘된 라이브러리를 찾아보았다.

그러다가 머리를 스친게 하나 있었다. 

요즘 컴퓨터비전쪽으로 공부를 하다보니 생각이 난 것이데 이미지 파일이 python에서는 2차원(또는 3차원) numpy array 로 구성되어 있다는 것이다.

따라서 이미지 처리하는  C++ 라이브러리인 OpenCV가 떠올랐다. 이미 나의 프로젝트에서는 OpenCV를 이용하고 있었으므로 이를 그냥 사용만 하면 되는 것이다.

이번에 사용하면서 주로 쓴 OpenCV Mat class를 포스팅하겠다.

# 전체 예제 코드
```c
#include<iostream>
#include<opencv2/opencv.hpp>
using namespace std;
using namespace cv;
int main(){
    //Mat 기본적인 매트릭스 클래스
    Mat X = (Mat_<double>(3, 3) << 1, 2 ,3, 8, 7, 3, 3, 4, 5);
    Mat Z;
    //Matx 
    //작은 크기 크기가 정해저버림
    //각각의 요소에 접근할수 없음
    Matx31d Y(9, 8, 7); 
    Matx<double,1,3> W(1, 2, 3);
    Mat I = cv::Mat::eye(3, 3, CV_64FC1);//단위 행렬 만들기
    cout<<I<<endl;
    cout<<X.inv()<<endl; //X의 역행렬 계산
    cout<<X.t()<<endl; //X의 전치 행렬
    cout<<X.diag()<<endl; // 대각 행렬
    
    cout<<X;//현재의 X
    cout<<X*Y;//X*Y
    X.mul(Y); //X*Y를 X에 저장
    cout <<X<<endl;//바뀐 X
    
    Z.copySize(X);//Z에 X크기 집어넣기 이렇게 안하면 가끔 사이즈가 안맞다고 계산이 안되는 경우가 있음
    Z=X;//Z에 X대입
    
    cout<<Y<<endl;
    cout <<Z<<endl;
    cout<<Z.at<double>(1)<<endl;//Z에 1번 인덱스 2번째 행에 있는 값출력
}
```

## 생성자
```c
//Mat 기본적인 매트릭스 클래스
Mat X = (Mat_<double>(3, 3) << 1, 2 ,3, 8, 7, 3, 3, 4, 5);
Mat Z;
//Matx 
//작은 크기 크기가 정해저버림
//각각의 요소에 접근할수 없음
Matx31d Y(9, 8, 7); 
Matx<double,1,3> W(1, 2, 3);
```
Mat과 Matx는 다른데 Mat은 사이즈가 바뀔 수 있고 Matx는 아니라는 점이다.

또한 Mat은 at()을 이용하여 각각의 요소에 접근할수 있는 데 Matx는 할수없다.

하지만 Matx를 쓰는 이유는 코드가 더가벼워 질수 있기 때문이다.

Matx는 크기가 정해지고 안에 있는 값을 바꿀 수없을때 쓸수 있고 코드가 가벼워지길 원한다면 이런 조건일때는 Matx를 적극적으로 사용하는 것이 좋을 것 같다.

## 수치해석적 메서드
```c
Mat I = cv::Mat::eye(3, 3, CV_64FC1);//단위 행렬 만들기
cout<<I<<endl;
cout<<X.inv()<<endl; //X의 역행렬 계산
cout<<X.t()<<endl; //X의 전치 행렬
cout<<X.diag()<<endl; // 대각 행렬
```
Mat 행렬은 단위 행렬도 생성해주는 메서드와 역행렬을 계산해주는 매서드 inv(), 전치 행렬을 계산해주는 메서드 t() , 대각 행렬을 계산해주는 매서드 diag()가 있다.

이외에도 다양한 메서드가 있지만 크게 쓰이는 건 이 정도이다.

```c
cout<<X;//현재의 X
cout<<X*Y;//X*Y
X.mul(Y); //X*Y를 X에 저장
cout <<X<<endl;//바뀐 X
```
또한 mul()이라는 메서드가 있는데 이것을 쓰면 현재 객체에 파라미터로 넣어준 객체를 곱해준 값을 현재 객체에 넣어주는 것이다.

이것은 일반적인 행렬곱과는 다른데 X*Y를 하면 이것을 계산한 값을 리턴 해주는 방면 X.mul(Y)는 X=X*Y와 같은 것이 된다.

```c
Z.copySize(X);//Z에 X크기 집어넣기 이렇게 안하면 가끔 사이즈가 안맞다고 계산이 안되는 경우가 있음
Z=X;//Z에 X대입
```
Z는 위에서 그냥 Mat Z로 Mat 클래스로 Z를 정의만 해줬다.

이를 그냥 수치 계산으로 넘기면 크기가 정해지지 않았다고 에러를 뱉어준다. 따라서 이미 정의된 Mat클래스중에 원하는 크기의 사이즈를 Z에 입력해줘야하는데, 이를 해주는 것이 copySize() 이다.

## 요소 접근
```c
cout<<Z.at<double>(1)<<endl;//Z에 1번 인덱스 2번째 행에 있는 값출력
```
마지막으로는 at() 메서드인데 이는 파라미터로 넣어주는 인덱스에 있는 행렬 요소를 출력해준다. 

# 후기
내가 짠코드 보다 훨씬 더 간결하게 매트릭스를 선언할 수 있었다.

기계과 출신이라 그런지 아직도 최적화쪽은 공부가 좀 필요한 것같았다.

결과적으로 코드가 많이 가벼워진 것 같았다.

그래도 오랜만에 수치해석적으로 Matrix.h를 짤 수 있어서 좋았다. 