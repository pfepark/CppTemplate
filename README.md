# CppTemplate
modern cpp template

## Object Generator Idioms
```cpp
template<typename T, typename U> struct pair
{
    T first;
    U second;
    pair(const T& a, const U& b) : first(a), second(b) {}
};

template<typename T, typename U>
pair<T, U> make_pair(const T& a, const U& b)
{
    return pair<T, U>(a, b);
}
```
C++17이 나오기 전까지는 클래스 템플릿은 함수 인자를 통해 타입을 추론할 수 없기 때문에, 클래스 템플릿 사용지 복잡하고 불편하다.
"클래스 템플릿의 객체를 생성하는 함수 템플릿"을 사용한다.

## identity
```cpp
template<typename T> struct identity
{
    typedef T type;
}

template<typename T> void foo(T a) {}
template<typename T> void goo(typename identity<T>::type a) {}

int main()
{
    identity<int>::type n;  // int
    
    foo(3);        // ok
    foo<int>(3);   // ok
    
    goo(3);        // error 
    goo<int>(3);   // ok
```
1. 함수 템플릿 사용시 컴파일러에 의한 타입 추론을 막는 테크닉
2. 함수 템플릿 사용시 사용자가 반드시 타입을 전달하도록 하고 싶을 때.
- 컴파일레 의한 타입 추론이 원하지 않는 타입으로 추론되는 경우

## lazy instantiation
```cpp
template<typename T> class A
{
    T data;
public:
    void foo(T n) {*n = 10; }   // error
};

int main()
{
    A<int> a;   // compile ok. 
    a.foo(0);   // error
}
```
사용하지 않는 템플릿은 인스턴스화 되지 않는다.
static 맴버도 인스턴스화 되지 않는다.

## lazy instantiation, if
```cpp
template<typename T> void foo(T n)
{
    *n = 10;
}

int main()
{
    if (false)
        foo(0);    // compile error
        
    if constexpr (false)
        foo(0);    // compile ok
}
```
1. if문은 "실행시간 조건문"이므로, 컴파일 시간에 조건이 false로 결정되어도 if문에 있는 코드는 사용되는 것으로 간주 된다.
2. C++17 if constexpr은 "컴파일 시간 조건문"이므로 조건이 false로 결정되면 if문에 있는 코드는 사용되지 않는 것으로 간주 된다.

```cpp
template<typename T> void foo(T n, int)
{
    *n = 3.4;
}

template<typename T> void foo(T n, double)
{
}

int main()
{
    foo(1, 3.4);    // compile ok
}
```
동일한 이름의 함수가 여러 개 있을 때 어떤 함수를 호출할지 결정하는 것은 컴파일 시간에 이루어 진다. 선택되지 않은 함수가 템플릿이라면 인스턴스화되지 않는다.

## Template Type Deduction
```cpp
#include <iostream>
#include <boost\type_index.hpp>

using namespace boost::typeindex;

template<typename T> void foo(const T a)
{
	//std::cout << "T : " << typeid(T).name() << std::endl;		// const, reference, volatile 은 표준함수로 타입을 알 수 없다.
	//std::cout << "a : " << typeid(a).name() << std::endl;

	std::cout << "T : " << type_id_with_cvr<T>().pretty_name() << std::endl;
	std::cout << "a : " << type_id_with_cvr<decltype(a)>().pretty_name() << std::endl;
}


int main()
{
	foo(3);
	foo(3.3);

    return 0;
}
```

### Template Argument Type Deduction
```cpp
template<typename T> void foo(const T a)
{
	++a;
}

int main()
{
	int n = 0;
	int& r = n;
	const int c = n;

	foo(n);	// T : int
	foo(c); // T : const int / int (const int면 ++가 안되는데..) 그래서 int
	foo(r);	// T : int& / int (int&면 원본이 바뀌는데?) 그래서 int

    return 0;
}
```
1. 컴파일러가 함수 인자를 보고 템플릿의 타입을 결정하는 것
2. 함수 인자의 타입과 완전히 동일한 타입으로 결정되지 않는다.

```cpp
// 함수 템플릿 인자가 값 타입(T a) 일 때.
template<typename T> void foo(T a)
{
	std::cout << type_id_with_cvr<T>().pretty_name() << std::endl;
}

int main()
{
	int n = 0;
	int& r = n;
	const int c = n;
	const int& cr = c;

	foo(n);	// int
	foo(c); // int
	foo(r);	// int
	foo(cr); // int

	const char* s1 = "hello";		// 포인터가 가르키는 것이 const
	foo(s1);						// 그러므로 const char*

	const char* const s2 = "hello"; // s2값도 const이므로 제거되어 const char*
	foo(s2);
    return 0;
}
```
1. 규칙 1. 템플릿 인자가 값 타입일 때 (T a)
- 함수 인자가 가진 const, volatile, reference 속성을 제거하고 T의 타입을 결정한다.
주의. 인자가 가진 const 속성만 제거 된다.

```cpp
template<typename T> void foo(T& a)
{
	std::cout << type_id_with_cvr<T>().pretty_name() << std::endl;
	std::cout << type_id_with_cvr<decltype(a)>().pretty_name() << std::endl;
}

template<typename T> void goo(const T& a)
{
	std::cout << type_id_with_cvr<T>().pretty_name() << std::endl;
	std::cout << type_id_with_cvr<decltype(a)>().pretty_name() << std::endl;
}

int main()
{
	int n = 0;
	int& r = n;
	const int c = n;
	const int& cr = c;

	foo(n);	// T : int,			a : int&
	foo(c); // T : const int,	a : const int& 가르키고 있으므로 원본 속성을 유지해야함
	foo(r);	// T : int,			a : int&	(T의 &는 제거해도 a가 참조가 유지됨)
	foo(cr); // T : const int	a : const int&

	goo(n);	// T : int,			a : const int&
	goo(c); // T : int,			a : const int&
	goo(r);	// T : int,			a : const int&
	goo(cr); // T : int			a : const int&

    return 0;
}
```
1. 규칙 2. 템플릿 인자가 값 타입일 때 (T& a)
- 함수 인자가 가진 reference 속성만 제거하고 T의 타입을 결정한다.
- const와 volatile 속성은 유지한다.

```cpp
template<typename T> void foo(T&& a) {}

int x[3] = { 1, 2, 3 };
foo(x);
```
1. 규칙 3. 템플릿 인자가 forwarding 래퍼런스 일 때 (T&& a)
- lvalue와 rvalue를 모두 전달 받는다. (perfect forwarding 참조)

1. 규칙 4. 배열을 전달 받을 때 - argument decay 발생

### Class template
```cpp
template<typename T> class Complex
{
    T re, im;
public:
    void(Complex c) // -> void foo(Complex<T> c)
    {
        Complex c2; // -> Complex<T> c2;
    }
}

void foo(Complex c)	// error
{
}

int main()
{
    Complex      c1;    // error, need c++17 deduction guide
    Complex<int> c2;
}
```
1. template & type
- Complex : template
- Complex<T> : type
2. 맴버 함수 안에서는 Complex<T> 대신 Complex를 사용할 수 있다.

```cpp
template<typename T> class Complex
{
    T re, im;
public:
    Complex(T a = {}, T b = {}) : re(a), im(b) {}
    T getReal() const;
    static int cnt;
    
    // 클래스 템플릿의 맴버함수 템플릿
    template<typename U> T func(const U& c);
};

template<typename T> template<typename U>
T Complex<T>::func(const U& c)
{
}

tempalte<typename T>
int Complex<T>::cnt = 0;

tempalte<typename T> T Complex<T>::getReal() const
{
    return re;
}

int main()
{
}
```
1. 디폴트 값 표기
- int a = 0;
- T a = T();    // C++98/03
- T a = {};     // C++11
2. 맴버 함수를 외부에 구현하는 모양
3. static member data의 외부 선언 
4. 클래스 템플릿의 맴버 함수 템플릿

### Generic copy constructor
```cpp
template<typename T> class Complex
{
    T re, im;
public:
    Complex(T a = {}, T b = {}) : re(a), im(b) {}
    
    // 일반적인 복사 생성자
    //Complex(const Complex<T>& c) {}
    
    //Complex(const Complex<int>& c) {} // Complex<int>만 받을 수 있다.
    
    template<typename U>
    Complex<const Complex<U>& c) {}
    
    template<typename>
    friend class Complex; // 모든 종류의 Complex Template class와 friend관계다.
};

// 일반적인 복사 생성자 구현
template<typename T> template<typename U>
Complex<T>::Complex(const Complex<U>& c) : re(c.re), im(c.im)	// c는 private으로 컴파일되지 않아 friend선언이 필요.
{
}

int main()
{
    Complex<int> c1(1, 1);   // ok
    Complex<int> c2 = c1;    // ok
    
    Complex<double> c3 = c1;    // Complex<int>와 Complex<double>은 다른 타입이다.
```

```cpp
#include <memory>
using namespace std;

class Animal {};
class Dog : public Animal {};

int main()
{
    shared_ptr<Dog> p1(new Dog);
    shared_ptr<Animal> p2 = p1; // template타입이 다른데 상속관계로 인해 컴파일이 되어야 한다. shared_ptr는 복사생성자, 대입연산자 등 다 구현되어 있다.
    
    p2 = p1;
    
    if (p2 == p1) {}
    if (p2 != p1) {}
    
    return 0;
}
```
### template && friend function
```cpp
#include <iostream>
using namespace std;

template<typename T> void foo(T a)
{
    cout << "T" << endl;
}

void foo(int a)
{
    cout << "int" << endl;
}

int main()
{
    foo(3); // template이 있다고 하더라고 정확한 타입 함수가 우선.

    return 0;
}
```
1. 함수 탬플릿 보다는 일반 함수가 우선해서 선택된다.(exactly matching)
2. 함수 탬플릿 있어도 동일한 타입의 인자를 가지는 일반 함수의 선언만 있으면 link에러가 발생한다.

```cpp
#include <iostream>
using namespace std;

template<typename T>
class Point
{
    T x, y;
public:
    Point(T a = 0, T b = 0) : x(a), y(b) {}
    // 클래스 탬플릿 선언임.
    friend ostream& operator<<(ostream& os, const Point& p); // 맴버변수 접근을 위해서..
    {
         return os << p.x << ", " << p.y;     
    }
}

template<typename T>
ostream& operator<<(ostream& os, const Point<T>& p)
{
    return os << p.x << ", " << p.y;
}

int main()
{
    Point<int> p(1, 2);	// 특정타입에 대한 선언이 되어 Point클래스의 함수탬플릿이 특정 타입에 대한 선언이 됨. link애러 발생

    cout << p << endl;  // 추출 연산자 재정의 필요
}
```
클래스 탬플릿 안에 friend 함수를 선언하는 방법
1. friend 함수 선언시에 함수 자체를 탬플릿 모양으로 선언
2. friend 함수를 일반 함수로 구현하고, 구현을 클래스 탬플릿 내부에 포함한다.
