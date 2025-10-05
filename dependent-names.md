## Nazwy zależne od typów

Gramatyka języka C++ nie jest niezależna od kontekstu. Aby sparsować np. definicję funkcji, potrzebna jest znajomość kontekstu, w którym funkcja jest definiowana.

Przykład problemu:

```cpp
template <typename T>
auto dependent_name_context1(int x)
{
    auto value = T::A(x);
    
    return value;
}
```

Standard C++ rozwiązuje problem przyjmując założenie, że dowolna nazwa, która jest zależna od parametru szablonu odnosi się do zmiennej, funkcji lub obiektu.

```cpp
struct S1
{
    static int A(int v) { return v * v; }
};

auto value2 = dependent_name_context1<S1>(10); // OK - T::A(x) was parsed as a function call
```

Słowo kluczowe `typename` umożliwia określenie, że dany symbol (identyfikator) występujący w kodzie szablonu i zależny od parametru szablonu jest typem, np. typem zagnieżdżonym – zdefiniowanym wewnątrz klasy.

```cpp
struct S2
{
    struct A
    {
        int x;
        
        A(int x) : x{x}
        {}
    };
};

template <typename T>
auto dependent_name_context2(int x)
{
    auto value = typename T::A(x); // hint for a compiler that A is a type
    
    return value;
}

auto value = dependent_name_context2<S2>(10); // OK - T::A was parsed as a nested type
```

Przykład użycia słowa kluczowego `typename` dla zagnieżdżonych typów definiowanych w kontenerach:

```cpp
template <class T> 
class Vector 
{      
public:
    using value_type = T;
    using iterator = T*;
    using const_iterator = const T*;    

    // ...
    // rest of implementation
};

template <typename Container>
typename Container::value_type sum(const Container& container)
{
    using result_type = typename Container::value_type;

    result_type result{};

    for(typename Container::const_iterator it = container.begin(); it != container.end(); ++it)
        result += *it;

    return result;
}
```

Podobne problemy dotyczą również nazw zależnych od parametrów szablonu i odwołujących się do zagnieżdżonych definicji innych szablonów:

```cpp
struct S1 { static constexpr int A = 0; }; // S1::A is an object
struct S2 { template<int N> static void A(int) {} }; // S2::A is a function template
struct S3 { template<int N> struct A {}; }; // S3::A is a class template
int x;

template<class T>
void foo() 
{
    T::A < 0 > (x); // if T::A is an object, this is a pair of comparisons;                    
                    // if T::A is a typename, this is a syntax error;                        
                    // if T::A is a function template, this is a function call;        
                    // if T::A is a class or alias template, this is a declaration.
}

foo<S1>(); // OK
```

Aby określić, że dany symbol zależny od parametru szablonu to szablon funkcji piszemy:

```cpp
template <typename T>
voi foo()
{
    T::template A<0>();
}
```

Aby określić, że dany symbol zależny od parametru szablonu to szablon klasy piszemy:

```cpp
template <typename T>
voi foo()
{
    typename T::template A<0>();
}