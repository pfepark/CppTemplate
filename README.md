# CppTemplate
modern cpp template

### Object Generator Idioms
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

### identity
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
