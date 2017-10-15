---
title: EMC_Chatper25
date: 2017-10-15 15:55:59
categories:
- Programming
tags: [c++, EMC, std::move, std::forward]
---

### 항목 25: 오른값 참조에는 std::move를, 보편 참조에는 std::forward를 사용하라.
오른값 참조는 이동할 수 있는 객체에만 묶이고, 어떤 매개변수가 오른값 참조라면, 그 참조에 묶인 객체를 이동할 수 있음이 확실하다.

``` cpp
class Widget {
	Widget(Widget&& rhs); //rhs 는 이동이 가능한 객체를 참조함이 확실함.
	...
};
```

그런데 만약에 그런 이동 가능한 객체를 다른 함수에 넘겨주되, 그 함수의 오른값 성질을 활용할 수 있도록 넘겨주어야 하는 경우에는 어떻게 해야할까?
그러한 경웅우는 매개변수를 오른값으로 캐스팅 해야한다. 이는 std::move가 하는 일이다.

``` cpp
class Widget {
	Widget(Widget&& rhs)
	: name(std::move(rhs.name)),
	  p(std::move(rhs.p))
	{
			...
	}; //rhs 는 이동이 가능한 객체이미므로 이 성질을 활용하여 std::move 활용.
	
	std::string name;
	std::shared_ptr<SomeDataStructure> p;	
};
```

반면 보편 참조는 이동에 적합한 객체에 묶일 수도 아닐 수도 있다.
보편 참조는 오른값으로 초기화되는 경우에만 오른값으로 캐스팅 되어야 한다.

#### 오른값 참조를 다른 함수로 전달할 때에는 오른값으로의 무조건 캐스팅 적용(std::move). 하지만 보편 참조를 다른 함수를 전달할 때에는 오른값으로의 조건부 캐스팅 적용(std::forward)

오른값 참조에 std::forward를 사용하면 안되는 이유?
- 소스 코드가 장황하고, 실수의 여지가 있으며 관용구에서 벗어남.

보편 참조에 std::move를 사용하면 안되는 이유?
- 왼값이 의도치 않게 수정되는 수정될 수 있기 때문.


``` cpp
class Widget
{
public:
    // 컴파일 되지만 아주 아주 나쁘다!
    template<typename T>
    void setName(T&& newName)
    { name = std::move(newName); }
    ...

private:
    std::string name;
    ...
};

std::string getWidgetName(); //팩터리 함수

Widget w;

auto n = getWidgetName(); //n 은 지역 변수

w.setName(n); // n을 w로 이동시킨다.

//이제 n의 값에 대해서는 알 수 없다...(미지정 값을 가짐)
}
```

만약 이러한 문제점 때문에 매개변수의 오른값과 왼값에 대해서 오버로딩을 통한 각각 함수를 만들어 주면 어떨까?

``` cpp
class Widget
{
public:
    //const 왼값으로 name을 설정
    void setName(const std::string& newName)
    { name = newName; }

    //오른값으로 name을 설정
    void setName(std::string&& newName)
    { name = std::move(newName); }
};
```

하나의 해결책이 될 수는 있지만 몇 가지 단점이 있다.
- 작성하고 유지 보수해야할 코드가 늘어남.
- 효율 성이 떨얻짐

``` cpp
w.setName("Adela Novak")
```

위의 호출부를 보면 처음의 template 에서는 T&& 형식의 보편 참조이기때문에, T는 const char*로 연역되어 임시 객체 std::string을 만들지 않는다.
하지만 오버로딩을 통했을때는 T가 아닌 std::string 타입으로 고정되어있기 때문에 매개변수에 먼저 literal을 적용시켜야 해서 임시 std::string 객체가 생성된다.
그리고 그 임시 std::string이 w의 자료 멤버로 이동한다. 즉 임시 객체 생성을하고, std::move가 끝난후 임시 객체를 파괴하기 위해 소멸자도 불리우기 때문에 효율이 떨어진다.
또한 매개변수의 값이 늘어날수록 오버로딩하여 관리해야할 함수의 갯수는 2^n(매개변수 갯수) 이기 때문에 거의 유지보수가 불가능에 가깝게 된다.

``` cpp
template<typename T>
void setSignText(T&& text)
{
    //text를 사용하지만 수정하지는 않는다.
    sign.setText(text); 

    auto now = std::chrono::system_clock::now();

	//text 를 오른값으로 조건부 캐스팅
    signHistroy.add(now, std::forward<T>(text));
}
```

만약 sign.setText에서 text값을 변경시켜버리면 dsignHistory.add에서 text를 사용하기 때문에 문제가 될 수 있다.
즉 std::forward를 보편 참조를 마지막으로 사용하는 지점에서만 적용해야 한다. 즉, 마지막 사용되기 전에 오른값참조나 보편 참조에 묶인 객체를 다른 객체로 이동시켜버리면, 해당 객체를 혹시라도 나중에 사용하는 경우에 문제가 될 수 있다.
따라서 미리 이동시키면 안되고, 마지막에 가서 이동을 시켜야 한다. 이는 std::move, std::forward 둘다에 해당한다.

### return 값 최적화

함수가 결과를 값으로 돌려준다면(return by value) 그리고 그것이 오른값 참조나 보편 참조에 묶인 객체라면 해당 참조를 돌려주는 return 문에서 std::move나 std::forward를 사용하는 것이 바람직하다.
다음의 예를 살펴보자.
``` cpp
Matrix operator+(Matrix&& lhs, const Matrix& rhs)
{
    lhs += rhs;
    return std::move(lhs); //lhs를 반환값으로 이동
}

Matrix operator+(Matrix&& lhs, const Matrix& rhs)
{
    lhs += rhs;
    return lhs; //lhs를 반환값으로 복사
}
```

첫번째 경우에는 lhs를 std::move롤 통해 오른값으로 캐스팅한 덕분에 컴파일러가 lhs를 함수의 반환값 장소로 "이동" 시킨다.
그런데 두번째 경우 처럼 std::move가 없다면 lhs를 반환값 장소로 복사시킨다. 즉, 복사 시간만큼 더 성능이 안좋아 질 수 있다.
만약 Matrix 객체가 이동 생성을 지원하지 않는다고 해도 오른값으로의 캐스팅이 해가 되지 않는다. 이동 생성이 없으면 복사와 똑같이 기능을 할 것이기 때문이다.
따라서 이동생성이 없더라도 std::move를 하는 것이 적합한 이유이다.
이는 보편참조에도 해당된다. 즉 반환해야될 것이 오른값이면 이동, 왼값이면 복사를 해서 성능의 이점을 누릴 수 있다.

``` cpp
template<typename T>
Fraction reduceAndCopy(T&& frac)
{
    frac.reduce();
    
    // frac이 오른값이면 이동, 왼값이면 복사
    return std::forward<T>(frac);
}
```

### 반환값 최적화 이용시 주의점

반환값 최적화를 활용하려고 할 때 주의해야할 점을 살펴보자.

``` cpp
Widget makeWidget()
{
    Widget w; //지역변수

    ...

    return w; //반환값으로 지역변수를 복사.
}
```

위와 같은 경우가 있을때 이를 최적화하기 위해서 다음과 같이 이동을 활용하려고 한 경우를 보자.
``` cpp
Widget makeWidget()
{
    Widget w;

    ...

    return std::move(w); //w를 반환값으로 이동한다.(잘못된 것이다!)
}
```

그런데 이렇게 하면 안되는것이 이런 경우를 대비해서 표준 위원회가 이미 수를 써놓았다.
만약 지역변수 w를 함수의 반환값을 위해 마련한 메모리 안에서 생성하려면 w의 복사는 불가피하다.
그런데 이는 이미 오래전부터 알고있던 이슈였고, 이를 해결하기위해 RVO(return value optimization)을 적용시켰다.
그 방식의 규칙은 
- 지역 객체의 형식이 함수의 반환 형식과 같다
- 그 지역 객체가 바로 함수의 반환값이다

이 2가지를 만족하면 RVO를 적용하여 최적화를 시켜준다(따로 무얼 하지 않아도)

이 시점에서 다시 이부분을 살펴보자.
``` cpp
return std::move(w);
```
여기서는 조건을 만족시키지 못해서 RVO가 적용되지 않는다.
바로 지역객체 w를 돌려주는 것이 아니라 std::move(w)의 결과를 돌려준다.
즉 w에 대한 참조를 돌려주므로 RVO가 적용되지 않는다. 
따라서 RVO를 적용할 수 있는데 적용 시키지 않아서 성능상 손해를 가져온다(불필요한 이동연산 때문)

그러면 만약 제어흐름에 의해 반환값에 대한 복사가 불가피할 경우는 어떨까?

``` cpp
Widget makeWidget() 
{
	Widget w1,w2,w3,w4;

	if(...) 
	{
		return w1;
	}
	else if(...) 
	{
		return w2;
	}
	else if(...) 
	{
		return w3;
	}
	else 
	{
		return w4;
	}
}
```

그런데 이러한 경우에도 std::move를 적용해주면 여전히 안된다.
위와 같은 이유에서 RVO의 조건은 만족하지만 제어가 복잡하여 RVO를
적용시킬 수 없을때는 컴파일러가 반환되는 객체에 암묵적으로 std::move를 적용시킨다.
즉 애초에 다음과 같이 컴파일러는 인식하는 것이다.
``` cpp
Widget makeWidget() 
{
	Widget w1,w2,w3,w4;

	if(...) 
	{
		return std::move(w1);
	}
	else if(...) 
	{
		return std::move(w2);
	}
	else if(...) 
	{
		return std::move(w3);
	}
	else 
	{
		return std::move(w4);
	}
}
```

이러한 규칙은 매개 변수에도 똑같이 적용된다.

다음과 같이 작성한 코드가 있다고 해보자.
``` cpp
Widget makeWidget(Widget w)
{
    ...
    return w; //값전달 방식으로 매개변수를 반환값으로 전달
}
```


위의 코드를 컴파일러는 다음과 같이 인식해버린다.
``` cpp
Widget makeWidget(Widget w)
{
    ...
    return std::move(w); //컴파일러는 w를 오른값으로 취급
}
```

### 정리
- 오른값 참조나 보편 참조가 마지막으로 쓰이는 지점에서, 오른값 참조에는 std::move를, 보편 참조에는 sdt::forward를 적용.
- 결과를 값 전달 방식으로 돌려주는 함수가 오른값 참조나 보편 참조를 돌려줄 때에도 std::move 혹은 std::forward를 적용.
- 반환값 최적화(RVO)의 대상이 될 수 있는 지역 객체에는 절대로 std::move 나 std::forward를 적용하지 말것.