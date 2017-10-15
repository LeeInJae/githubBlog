---
title: EMC_Chapter26
date: 2017-10-15 18:21:52
categories: 
- Programming
tags: [c++, EMC, std::move, std::forward]
---

### 26. 보편참조에 대한 중복적재를 피하라

사람 이름 하나를 매개변수로 받고, 현재 날짜와 시간을 기록 후 전역 자료구조에 추가하는 함수.
``` cpp
std::multiset<std::string> names; //전역 자료구조

void logAndAdd(const std::string& name)
{
    auto now = 
        std::chrono::system_clock::now();

    log(now, "logAndAdd");

    names.emplace(name); //이름을 전역 자료구조에 추가;
}
```

이 코드는 기능상에는 문제가 없지만 효율성에서 문제가 있다.
다음과 같이 호출 형태에 따라서 살펴보자.

``` cpp
std::string petName("Darla");

//왼값 std::string 넘기는 경우
logAndAdd(petName); 

//오른값 std::string 넘기는 경우
logAndAdd(std::string("Persephone"));

//문자열 리터럴을 넘기는 경우
logAndAdd("Patty Dog");
```

첫 호출에서 매개변수 name은 변수 petName에 묶인다. 그리고 names.emplace로 전달 된다. name은 왼값이므로 emplace는 그것을 names에 복사한다.
logAndAdd에 전달된 값이 왼값이므로 복사는 불가피하다.
둘째 호출에서는 매개변수 nabme이 오른값에 묶인다. name자체는 왼값이므로 names에 복사된다. 그러나 원칙상 name을 names로 이동하는 것이 가능하다. 즉 복사1회 비용을 이동 작업으로 대체가능하다.
셋째 호출에서는 매개변수 name은 오른값에 묶이지만 암묵적으로 "Patty Dog"에 대한 stjd::string 객체가 생성되고, name은 names에 복사된다. 만약 문자열 리터럴을 emplace에 직접 전달했다면 직접 std::multiset안에서 std::string 객체를 생성할수 있는데도 말이다. 즉 std::string 임시 객체도 필요없고 복사와 이동도 필요없는 경우이다.
이를 해결하기 위해서 logAndAdd가 보편 참조를 받게하면 std::forward를 이용해서 emplace에 전달시킬 수 있고, 둘째, 셋째 호출의 비효율성을 제거할 수 있다.

``` cpp
template<typename T>
void logAnddAdd(T&& name)
{
    auto now = std::chrono::system_clock::now();
    log(now, "logAndAdd");
    names.emplace(std::forward<T>(name));
}

std::string petName("Darla");

//multiset으로 복사
logAnddAdd(petName);

//오른값 이동.
logAndAdd(std::string("Persephone"));

//multiset안에서 std::string을 생성.
logAndAdd("Patty Dog");
```

그런데 만약 이름이 아니라 index를 통하여 이름을 전역 자료구조에 저장해야되는 경우도 생겼다고해보자.
이를 overloading을 통해서 구현한 예이다.

``` cpp
//idx에 해당하는 이름을 돌려준다.
std::string nameFromIdx(int idx);

void logAndAdd(int idx) //오버로딩
{
    auto now = std::chrono::system_click::now();
    log(now, "logAndAdd");
    names.emplace(nameFromIdx(idx));
}

std::string petName("Darla"); //이전과 동일

//모두 T&&를 받는 오버로딩 함수 호출
logAndAdd(petName);
logAndAdd(std::string("Persephone"));
logAndAdd("Patty Dog");

//int버전에 대한 오버로딩 함수 호출
logAndAdd(22);
```

위의 예에서는 기대한 대로 호출을 한다. 하지만 다음의 경우는 어떻게 될까?

``` cpp
short nameIdx;
...
logAndAdd(name[idx]); //오류
```

다음의 2가지 경우로 나뉘게 될 것이다.
- short& 로 연역이 되어, T&& 의 오버로딩 함수로 호출을 한다.
- short를 int로 승격시켜 int 타입의 오버로딩 함수를 호출한다.
그런데 이때 T&& 형태의 오버로딩 함수는 short를 int로 승격시키는 과정이 필요없이 호출 인자에 부합하므로 보편 참조 오버로딩 함수가 호출된다. 그렇게되면 매개변수 name은 short형태에 묶이게 되고,
name 이 std::forward를 통해서 names의 멤버함수 emplace에 전달된다. 그런데 std::string의 생성자 중에 short를 받는 버전은 없으므로 이는 실패하게된다.
따라서 보편참조와 오버로딩을 결합하는 것은 거의 항상 나쁘다.

그런데 만약에 생성자에서 이러한 보편참조를 사용해야될 경우는 어떻게 해야할까? 생성자는 특성상 메서드 이름을 달리할 수 없으므로 필연적으로 오버로딩이 생길수밖에 없다.
다음의 예를 살펴보자.
``` cpp
class Person
{
public:
    template<typename T>
    explicit Person(T&& n) //완벽 전달 생성자
    : name(std::forward<T>(n)) {}

    explicit Person(int idx) //int를 받는 생성자.
    : name(nameFromIdx(idx)) { }

private:
    std::string name;
};
```

앞선 경우와 마찬가지로 int 이외의 정수형식을 넘겨주면 int를 받는 생성자 대신 보편 참조를 받는 생성자가 호출되므로 컴파일이 실패한다.
그런데 앞선 경우보다 더 심각한 문제가 있는데 이는 컴파일러가 자동으로 만들어 내는 복사 생성자와 이동 생성자 때문이다. 
그래서 이 클래스는 실제로는 이렇게 된다.

``` cpp
class Person
{
public:
    template<typename T>
    explicit Person(T&& n)
    : name(std::forward<T>(n)) {}

    explicit Person(int idx)
    : name(nameFromIdx(idx)) { }

    //복사 생성자와 이동생성자를 컴파일러가 만들어버림
    Person(const Person& rhs);
    Person(Person&& rhs);

private:
    std::string name;
};
```

그리고 이렇게 호출하면 컴파일이 안된다.
``` cpp
Person p("nancy");

auto cloneOfP(p); //p로부터 새 Person 생성. 컴파일에 실패한다.
```
우리는 피상적으로는 이 호출이 복사생성자를 호출할 것이라고 예상하지만 한번 따져보자.
p의 타입은 현재 Person이고 복사생성자의 매개변수 타입은 const Person& rhs이다. 즉 복사생성자를 호출하려면 Person 타입을 const Person 타입으로 한번 더 암시적 변환이 필요하다.
그런데 보편 참조 형태는 Person& 형태로 바로 정합시켜서 호출이 가능하기때문에 복사생성자가 아닌 보편참조형태의 생성자가 호출되는 것이다.
그래서 의도한대로 호출을 하려면 다음과 같이 해야한다.

``` cpp
const Person p("nancy"); // const를 붙여야 복사생성자를 호출한다.

auto cloneOfP(p); //의도한 대로 복사생성자 호출
```
물론 여기서 T&&형태의 템플릿 생성자도 const Person& 형태로 인스턴스화 될수 있지 않느냐고 반문할 수 있다.
하지만 규칙에 의거하여 템플릿함수와 일반함수의 정합성이 동일하다면 일반함수가 우선적으로 호출이되기때문에 문제되지 않는다.

위와 같은 문제는 상속에서도 똑같이 발생한다.

``` cpp
class SpecialPerson : public Person
{
public:
    SpecialPerson(const SpecialPerson& rhs)
    : Person(rhs) { ... }

    SpecialPerson(SpecialPerson&& rhs)
    : Person(std::move(rhs)) { ... }
};
```

위와 같이 복사 및 이동 생성자에서 부모의 복사 및 이동생성자를 호춭하여 값을 세팅하려고 하는 경우를 살펴보자.
위와같이 Person(rhs), Person(std::move(rhs))에서 Person의 복사 및 이동생성자를 호출하지 않고 보편 참조 생성자를 호출하기 때문에 문제가 발생할 것이다.
(부모클래스의 보편참조 생성자에서 const SpecialPerson& 또는 SpecialPerson&&로 연역을 바로 할 수 있기 때문이다)

### 정리
- 보편 참조에 대한 중복적재는 거의 항상 보편 참조 중복적재 버전이 예상보다 자주 호출됨(의도하지 않게)
- 완벽 전달 생성자들은 특히 주의해야함. 비 const왼값보다 더 정합성에 부합하여 클래스의 복사 및 이동 생성자들에 대한 파생 클래스의 호출들을 가로챌 수 있음