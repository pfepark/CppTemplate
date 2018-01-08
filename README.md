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
