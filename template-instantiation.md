# Proces tworzenia instancji szablonu

**Tworzenie instancji szablonu** (*template instantiation*) to proces, w którym kompilator generuje konkretny kod funkcji na podstawie definicji szablonu i dostarczonych (lub wydedukowanych) typów argumentów.

## Model kompilacji szablonów

Szablony stosują specyficzny model kompilacji, który różni się od zwykłych funkcji:

* Cały kod szablonu musi być dostępny w miejscu użycia (zazwyczaj w pliku nagłówkowym)
* Kompilator generuje kod dla każdej unikalnej kombinacji parametrów szablonu

```cpp
// max_value.hpp
#ifndef MAX_VALUE_HPP
#define MAX_VALUE_HPP

template<typename T>
T max_value(T a, T b)
{
    return b < a ? a : b;
}

#endif // MAX_VALUE_HPP
```

```cpp

// main.cpp
#include "max_value.hpp"

int main()
{
    max_value(1, 2);      // instancja szablonu: max_value<int>(int, int)
    max_value(1.5, 2.5);  // instancja szablonu: max_value<double>(double, double)
}
```

## Rodzaje procesu tworzenia instancj szablonu

### Niejawne tworzenie instancji szablonu (implicit instantiation)

Kompilator automatycznie generuje kod szablonu w miejscu pierwszego użycia:

```cpp
// translation_unit.cpp
#include "max_value.hpp"

int x = max_value(1, 2);  // niejawnie utworzona instancja dla max_value<int>(int, int)
double y = max_value(1.5, 2.5);  // niejawnie utworzona instancja dla max_value<double>(double, double)
```

```{warning}
Niejawne tworzenie instancji może prowadzić do wielokrotnego generowania tego samego kodu w różnych jednostkach translacji, co zwiększa rozmiar plików obiektowych i czas kompilacji.
```

### Jawne tworzenie instancji szablonu (explicit instantiation)

Programista może wymusić wygenerowanie konkretnych instancji szablonu w jednostce translacji. Daje to kontrolę nad tym, gdzie kod jest generowany i może pomóc w optymalizacji czasu kompilacji oraz rozmiaru plików obiektowych.

```cpp
// math.hpp
#ifndef MATH_HPP
#define MATH_HPP

template <typename T>
T square(T value);

#endif // MATH_HPP
```

```cpp
// math.cpp
#include "math.hpp"
#include <complex>

template <typename T>
T square(T value)
{
    return value * value;
}

template int square<int>(int);
template double square<double>(double);
template std::complex<double> square<std::complex<double>>(std::complex<double>);
```

```{warning}
Proces tworzenia jawnej instancji blokuje możliwość tworzenia niejawnych instancji dla innych typów. Dostępne będą tylko te instancje, które zostały jawnie zadeklarowane.
```

```{note}
Dla szablonów klas składnia tworzenia jawnej instancji jest nieco inna:

````cpp
template class MyClass<int>;  // jawna instancja szablonu klasy 
````

```

## Wymagania wobec typów

Utworzenie instancji szablonu jest możliwe tylko wtedy, gdy dla typu podanego jako argument szablonu zdefiniowane są wszystkie operacje używane przez szablon.

```cpp
template<typename T>
T max(T a, T b)
{
    return b < a ? a : b;  // wymaga operatora< oraz konstruktora kopiującego
}

struct Point 
{
    int x, y;
    // brak operatora< - nie można użyć max()
};

struct Value
{
    int data;
    bool operator<(const Value& other) const 
    {
        return data < other.data;
    }
};

Point p1{1, 2}, p2{3, 4};
// max(p1, p2);  // ERROR - no operator< for Point

Value v1{10}, v2{20};
max(v1, v2);     // OK - operator< jest zdefiniowany
```

## Fazy kompilacji szablonu

Kompilacja szablonów przebiega w **dwóch fazach** (*two-phase lookup*):

1. **Faza 1: Analiza definicji szablonu**

    Na etapie definicji szablonu (przed tworzeniem jakiejkolwiek instancji) sprawdzana jest:

    * poprawność składniowa
    * użycie nazw **niezależnych od parametrów szablonu** (*non-dependent names*)
    * statyczne asercje, które nie zależą od parametru szablonu

2. **Faza 2: Analiza instancji szablonu**

    Podczas tworzenia instancji szablonu kod jest ponownie sprawdzany, ze szczególnym uwzględnieniem:

    * nazw **zależnych od parametrów szablonu** (*dependent names*)
    * wymagań dotyczących operacji na typach parametrów
    * statycznych asercji zależnych od parametrów

```cpp
template<typename T>
void foo(T t)
{
    undeclared();   // Błąd fazy 1: jeśli undeclared() jest nieznane
    undeclared(t);  // Błąd fazy 2: jeśli undeclared(T) jest nieznane dla danego T
    t.unknown();   // Błąd fazy 2: jeśli T nie ma metody unknown()
    
    static_assert(sizeof(int) > 10, "int too small");  // Faza 1: zawsze błąd jeśli warunek nie jest spełniony
    static_assert(sizeof(T) > 10, "T too small");      // Faza 2: błąd tylko dla T o rozmiarze <= 10
}
```