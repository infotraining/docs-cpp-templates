# Szablony zmiennych

W C++14 zmienne mogą być parametryzowane przy pomocy typu. 
Takie zmienne nazywamy **zmiennymi szablonowymi** (*variable templates*). Umożliwia to definiowanie stałych, które są sparametryzowane typem.

```cpp
template<typename T>
constexpr T pi = 3.141592653589793238462643383279502884;
```

Aby użyć zmiennej szablonowej, należy podać jej typ:

```cpp
std::cout << pi<double> << '\n';
std::cout << pi<float> << '\n';
```

Inny przykład zmiennej szablonowej:

```cpp
template<typename T>
constexpr int size = sizeof(T);

std::cout << size<int> << '\n';       // prints 4
std::cout << size<double> << '\n';    // prints 8
std::cout << size<std::string> << '\n'; // prints 32 (on 64-bit system)
```

## Zmienne szablonowe a jednostki translacji

Zmienne szablonowe mogą być używane w wielu jednostkach translacji (plikach źródłowych `.cpp`). Aby uniknąć problemów z wielokrotną definicją, deklarację zmiennej szablonowej w w pliku nagłówkowym (`.hpp`) poprzedzamy słowem kluczowym `inline`. Taka deklaracja zapewnia tzw. *external linkage* i zgodność z zasadą ODR (*One Definition Rule*). 

* plik - `math_constants.hpp`

```cpp
template<typename T>
inline constexpr T pi = 3.141592653589793238462643383279502884;
```

* plik - "unit1.cpp"

  ```cpp
  #include "math_constants.hpp"

  int main()
  {
      std::cout << pi<double> << '\n'; // OK: prints 3.14159...
  }
  ```

* plik - "unit2.cpp"

  ```cpp
  #include "math_constants.hpp"
  
  void print()
  {
      std::cout << pi<float> << '\n'; // OK: prints 3.14159...
  }
  ```
