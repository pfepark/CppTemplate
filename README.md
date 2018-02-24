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

### typename keyword
```cpp
class Test
{
public:
    enum { value1 = 1 };
    static int value2;

    typedef int INT;
    using SHORT = short;

    class innerClass {};
}

int Test::value2 = 1;

int main()
{
    int n1 = Test::value1;
    int n2 = Test::value2;

    Test::INT a;
    Test::SHORT b;
    Test::innerClass c;
}
```
1. "클래스 이름:"으로 접근 가능한 요소들
- 값 (enum 상수, static 맴버 변수)
- 타입 : typedef, using

```cpp
int p = 0;

class Test
{
public:
    //....
};

template<typename T>
int foo(T t) // T는 Test로 결정..
{
    // 아래 한줄을 해석하시오.
    T::DWORD * p;   // 값으로 해석 가능 : 곱하기 p
                    // 타입으로 해석 가능 : 포인터 지역변수 선언

    T::DWORD * p;   // 컴파일러는 T::DWORD는 값으로 해석
    typename T::DWORD * p // 컴파일러는 T::DWORD는 타입으로 해석한다.
    
    return 0;
}

int main()
{
    Test t;
    foo(t);
}
```

### value_type
```cpp
#include <iostream>
#include <vector>
#include <list>

using namespace std;

void print_first_element(vector<int>& v)
{
    int n = v.front();
    cout << n << endl;
}

// vector의 모든 타입을 처리하는 함수
template<typename T>
void print_first_element(vector<T>& v)
{
    T n = v.front();
    cout << n << endl;
}

// 모든 컨테이너를 처리하는 함수.
template<typename T>
void print_first_element(T& v)
{
    // 예 : T는 list<double>
    // double이 필요하다.
    // T::value_type => list<double>::value_type, STL에서는 typedef를 해두었다.
    typename T::value_type n = v.front();
    auto n = v.front(); // C++11
    cout << n << endl;
}

int main()
{
    vector<int> v = { 1, 2, 3 };
    print_first_element(v);
}
```
### class template type deduction using value_type
```cpp
#include <list>
using namespace std;

template<typename T> class Vector
{
    T* buff;
    int size;
public:
    Vector(int sz, T value) {}

    template<typename C> Vector(C c) {}
};

// 아래 생성자를 사용하게 되면 -> 이후 타입으로 추론해달라.
template<typename C>
Vector(C c) {} -> Vector<typename C::value_type>;

int main()
{
    Vector<int> v(10, 3);
    Vector v1(10, 3);    //C++17에서는 두번째 인자를 보고 추론가능
    list<int> s = {1, 2, 3};

    Vector v2(s);   // 생성자 추가하고 deduction guide 작성 필요

    // 도전과제 : 다른 컨테이너의 반복자로 초기화한 Vector
    Vector v3(s.begin(), s.end());
}
```
### template parameter
```cpp
// 템플릿 인자로 올 수 있는 것.
// 1. type
// 2. 값 (non-type)
// 3. template
template<typename T, int N> struct Stack
{
	T buff[N];
}

int main()
{
	Stack<int, 10> s;
	return 0;
}

// type parameter
template<typename T> class List
{

};

int main()
{
	list<int> s1;
}

// non-type (value) parameter
// 정수형 상수 (실수 안됨)
template<int N> class Test1 {};

// 2. enum 상수
enum Color { red = 1, green = 2 };
template<Color> class Test2 {}:

// 3. 포인터 : 지역변수의 주소는 안된다. 전역변수 주소는 가능.
// no linkage를 가지는 변수 주소는 안됨.
template<int*> class Test3 {};

int x = 0;

// 4. 함수 포인터
template<int(*)(void) class Test4 {};

int main()
{
	int n = 10;

	Test1<10> t1;	// ok
	//Test1<n> t2;	// error
	Test2<red> t3;	// ok

	//Test3<&n> t4;	// n은 여기서 스택의 주소인데 알 수 없다.
	Test3<&x>	t5;	// ok

	Test4<&main> t6;	// ok
}

// non-type(값) parameter
// 정수형 상수, enum, 포인터, 함수 포인터, 맴버 함수 포인터
// C++17 : auto

template<auto N> struct Test
{
	Test()
	{
		cout << typeid(N).name() << endl;	// type 확인
	}
};

int x = 0;

int main()
{
	Test<10> t1;	// N : int
	Test<&x> t2;	// N : int*
	Test<&main> t3;
}

// 3. tempalte parameter

template<typename T> class list {};

// 첫번째 인자는 타입, 두번째는 템플릿인데 인자가 하나인 템플릿.
template<typename T, template<typename> class C> class stack
{
	C c;	// error, list c
	C<T> c;	// ok. list<int> c
};

int main()
{
	list		s1;	// T를 결정할 수 없어서 error, list는 타입은 아니고 템플릿
	list<int>	s2; // ok. list<int>는 타입

	stack<int, list> s3;	// ok
}

// default parameter
template<typename T = int, int N = 10> struct Stack
{

}

int main()
{
	Stack<int, 10>	s1;
	Stack<int>		s2;
	Stack<>			s3;	// 모든 인자 디폴트 값 사용
}

// C++11 variadic template
template<typename ... T> class Test {};
tempalte<int ... N> class Test {};
```

### tempalte alias
```cpp
typedef int DWORD;
typedef void(*F)(int);

// C++11
using DWORD = int;
using F = void(*F)(int);

int main()
{
    DWORD n;
    
    F f;
}

 template<typename T, typename U> struct Pair
 {
	 T v1;
	 U v2;
 };

 typedef Pair Point;	// error, 타입의 별칭이 아니라 탬플릿의 별칭을 시도..

 template<typename T, typename U>
 using Point = Pair<T, U>;
 
 template<typename T>
 using Point2 = Pair<T, T>;
 
 template<typename T>
 using Point3 = Pair<T, short>;

 int main()
 {
	Point<int, double> p;
	Point2<int> p2; // Pair<int, int>
	Point3<int> p3; // Pair<int, short>
 }
```

### variable template (C++14)
```cpp
//#define PI 3.14

//constexpr double PI = 3.14;
//constexpr float PI = 3.14;

template<typename T>
constexpr T PI = 3.14;

template<typename T> void foo(T a, T b)
{

}

int main()
{
	float f = 3.3;
	foo(f, PI<float>);

	double d = PI<double>;
}
```

### specialization
```cpp
#include <iostream>
using namespace std;

// primary template
template<typename T> class Stack
{
public:
	void push(T a) { cout << "T" << endl; }
};

// partial specialization(부분 특수화, 전문화)
template<typename T> class Stack<T*>
{
public:
	void push(T* a) { cout << "T*" << endl; }
};

// char*라고 완벽히 타입이 결정되었으므로 typename T 필요없다.
// specialization(특수화, 전문화)
template<> class Stack<char*>
{
public:
	void push(char* a) { cout << "char*" << endl; }
};

int main()
{
	Stack<int> s1; s1.push(0);
	Stack<int*> s2; s2.push(0);
	Stack<char*> s3; s3.push(0);
}
```
특수화를 사용하면 소스코드는 좀 더 커진다. template이 많아서.. 하지만 기계어코드는 유사함.
### specialization example
```cpp
template<typename T, typename U> struct Test
{
	static void foo() { cout << "T, U" << endl; }
}

template<typename T, typename U> struct Test<T*, U>
{
	static void foo() { cout << "T*, U" << endl; }
}

template<typename T, typename U> struct Test<T*, U*>
{
	static void foo() { cout << "T*, U*" << endl; }
}

// 핵심
template<typename T> struct Test<T, T>
{
	static void foo() { cout << "T, T" << endl; }
}

template<typename T> struct Test<int, T>
{
	static void foo() { cout << "int, T" << endl; }
}

template<> struct Test<int, int>
{
	static void foo() { cout << "int, int" << endl; }
}

template<> struct Test<int, short>
{
	static void foo() { cout << "int, short" << endl; }
}

template<typename T, typename U, typename V>
struct Test<T, Test<U, V>>
{
	static void foo() { cout << "T, Test<U, V>" << endl; }
}

int main()
{
	Test<int, double>::foo();	// T, U
	Test<int*, double>::foo();	// T*, U
	Test<int*, double*>::foo();	// T, U

	Test<int, int>::foo();		// T, T -> int, int 템플릿 충돌로 추가.
	Test<int, char>::foo();		// int, U

	Test<int, short>::foo();	// int, short
	Test<int, Test<char, short>>::foo();	// T, Test<U,V>
}
```
### Partial Ordering
```cpp
#include <iostream>
using namespace std;

// primary template
template<typename T, typename U> struct order
{
	static void foo() { cout << "T, U" << endl; }
}

// partial specialization
template<typename T> struct order<T, T>
{
	static void foo() { cout << "T, T" << endl; }
};

template<typename T> struct order<T*, T*>
{
	static void foo() { cout << "T*, T*" << endl; }
};

int main()
{
	order<int*, int*>::foo();
}
```
template instantiation을 위해 어떤 템플릿을 사용할 것인가?
- specialization > partial specialization > primary template
2개 이상의 partial specialization 버전이 사용 가능 할 때
- The most specialized specialization is used

### Specialization 주의사항
```cpp
#include <iostream>
using namespace std;

template<typename T> struct Test
{
	static void foo() { cout << typeid(T).name() << endl; }
};

template<typename T> struct Test<T*>
{
	static void foo() { cout << typeid(T).name() << endl; }
};

int main()
{
	Test<int>::foo();	// T:int
	Test<int*>::foo();	// T:int
}
```
```cpp
template<typename T, int N = 10> struct Stack
{
	T buff[N];
};

// 부분 전문화버전은 default값은 primary template에 선언된 것을 사용한다.
template<typename T, int N> struct Stack
{
	T buff[N];
};

int main()
{
	Stack<int, 10> 	s1;
	Stack<int> 		s2;
	Stack<int*> 	s3;
}
```
```cpp
template<typename T> class Stack
{
public:
	T pop()	{}
	void push(T a);
}


template<typename T> void Stack<T>::push(T a)
{
	cout << "T" << endl;
}
// 맴버함수만 특수화
template<> void Stack<char*>::push(char* a)
{
	cout << "char*" << endl;
}
```
### IfThenElse
```cpp
#include <iostream>
#include <typeinfo>
using namespace std;

template<bool b, typename T, typename F> struct IfThenElse
{
	typedef T type;
};

template<typename T, typename F>
struct IfThenElse<false, T, F>
{
	typedef F type;
};

int main()
{
	IfThenElse<true, int, double>::type t0;		// int
	IfThenElse<false, int, double>::type t1;	// double 부분특수화로 인한..

	cout << typeid(t0).name() << endl;
	cout << typeid(t1).name() << endl;
}
// 컴파일 시간에 bool 값에 따라 type을 선택하는 도구

// 비트 관리 및 보관을 위한 클래스
template<size_t N>
struct Bit
{
	//int bitmap;	// 32bit 관리
	using type = typename IfThenElse<(N <= 8), char, int>::type;

	using type1 = typename conditional<(N <= 8), char, int>::type;

	type bitmap;
};

int main()
{
	Bit<32> b1;
	Bit<8> b2;

	Bit<16> b3;

	cout << sizeof(b1) << endl;	// 4
	cout << sizeof(b2) << endl;	// 1
	cout << sizeof(b3) << endl;	// 1
}
```
- IfThenElse, IF, Select 라는 이름으로 알려져 있다.
- C++ 표준에는 confitional이라는 이름으로 제공 <type_traits> 헤더.

### Couple Template
```cpp
#include <iostream>
using namespace std;

template<typename T> void printN(const T& cp)
{
	cout << T::N << endl;
}

template<typename T, typename U> struct couple
{
	T v1;
	U v2;

	enum { N = 2 };
};

// 2번째 인자가 couple일 경우
template<typename A, typename B, typename C>
struct couple<A, couple<B, C> >
{
	A v1;
	couple<B, C> v2;
	enum { N = 1 + couple<B, C>::N}
};

int main()
{
	couple<int, double> c1;
	couple<int, couple<int, char>> c2;
	couple<int, couple<int, couple<int, char>>> c3;

	printN(c1);	// 2
	printN(c2);	// 3
	printN(c3);	// 4
}
```
- 템플릿의 인자로 자기 자신의 타인을 전달하는 코드
- 부분 특수화를 만들 때 템플릿 인자의 개수
- N을 값을 표현하는 방법
```cpp
template<typename A, typename B, typename C>
struct couple< couple<A, B>, C> >
{
	couple<A, B> v1;
	C v2;
	enum { N = couple<A, B>::N + 1 };
}

int main()
{
	// 첫번째 인자가 couple일때..
	couple<int, double> c2;
	couple<couple<int, int>, int> c3;
	couple<couple<couple<int, int>, int>, int> c4;

	printN(c2);
	printN(c3);
	printN(c4);
	
	// error 컴파일러가 어떤 템플릿을 선택할 수 없으므로 다시 부분 특수화 필요.
	couple<couple<int, int>, couple<int, int>> c5;
}

template<typename A, typename B, typename C, typename D>
struct couple< couple<A, B>, couple<C, D> >
{
	couple<A, B> v1;
	couple<C, D> v2;
	enum { N = couple<A, B>::N + couple<C, D>::N };
};
```

### tuple using couple
```cpp
#include <iostream>
using namespace std;

template<typename T, typename U> struct couple
{
	T v1;
	U v2;

	enum { N = 2 };
};

int main()
{
	couple<int, couple<int, double>> c3;
	xtuple<int, int, double> t3;	// 이렇게 된다면 편하다.
}

template<typename T, typename U> struct couple
{
	T v1;
	U v2;

	enum { N = 2 };
};

struct Null {};	// empty class

// 2개이상 5개미만의 타입 전달
template<typename P1, 
		typename P2,
		typename P3 = Null,
		typename P4 = Null,
		typename P5 = Null>
class xtuple
 : public couple<P1, xtuple<P2, P3, P4, P5, Null>>
{

};

template<typename P1, typename p2>
class xtuple<P1, P2, Null, Null, Null>
: public couple<P1, P2>
{

};

int main()
{
	//										  couple<short, double>
	//                           couple<long, xt<short, double, Null, Null>>
	//             couple<char, xt<long, short, double, Null, Null>>
	// couple<int, xt<char, long, short, double, Null>>
	xtuple<int, char, long, short, double> t5;

	cout << sizeof(t5) << endl;

	//xtuple<int, char> t3;
}
```
1. empty class Null
 - 아무 멤버도 없는 클래스
 - 크기는 항상 1이다. (sizeof(Null))
 - 아무 멤버도 없지만 "타입"이므로 함수 오버로딩이나 템플릿 인자로 활용가능
2. 상속을 사용하는 기술
3. 개수의 제한을 없앨 수 없을까?  C++11 Variadic template

### Template Meta Programming
```cpp
// 컴파일 타임에 계산하는 프로그래밍
// tempalte meta programming
template<int N> struct factorial
{
	//enum { value = N * factorial<N-1>::value };
	// c++ 11
	static constexpr int value = N * factorial<N-1>::value;
};

// 재귀의 종료를 위해 특수화 문법 사용
template<> struct factorial<1>
{
	//enum { value = 1 };
	static constexpr int value = 1;
};

int main()
{
	int n = factorial<5>::value;	// % * 4* 3* 2* 1 = 120
	//      5 * f<4>::v
	//      4 * f<3>::v
	//      3 * f<2>::v
	//      2 * f<1>::v
	//            1

	cout << n << endl;
}
```
