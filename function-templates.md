# Szablony funkcji

**Szablon funkcji** - funkcja, której typy argumentów lub typ zwracanej wartości zostały sparametryzowane. Szablony funkcji są fundamentalnym mechanizmem programowania generycznego w C++, pozwalającym na pisanie kodu wielokrotnego użytku, który działa z różnymi typami danych przy zachowaniu bezpieczeństwa typów w czasie kompilacji.

## Składnia definicji szablonu funkcji

```cpp
// Definicja szablonu funkcji
template <typename T>  // lub template <class T>
return_type_declaration function_name(T arg)
{
    // implementacja
}

// Wiele parametrów szablonu
template <typename T, typename U, typename TResult>
TResult function_name(T arg1, U arg2)
{
    // implementacja
}
```

```{note}
Słowa kluczowe `typename` i `class` są w tym kontekście równoważne. Zaleca się używanie `typename` dla lepszej czytelności.
```

## Przykłady szablonów funkcji

### Prosty szablon funkcji max_value()

```cpp
template<typename T>
T max_value(T a, T b)
{
    return b < a ? a : b;
}
```

### Użycie szablonu funkcji max_value()

```cpp
// Jawne podanie typu argumentu szablonu
int result_1 = max_value<int>(4, 9);

// Automatyczna dedukcja typu argumentu szablonu
double x = 4.50;
double y = 3.14;
double result_2 = max_value(x, y);  // T dedukowane jako double

// Działa z dowolnym typem implementującym operator<
std::string s1 = "mathematics";
std::string s2 = "math";
std::cout << "max_value(s1, s2) = " << max_value(s1, s2) << '\n';
```

## Dedukcja typów argumentów szablonu

W momencie wywołania funkcji szablonowej następuje proces **dedukcji typów argumentów szablonu** (*template argument deduction*). Typy parametrów szablonu, które nie zostały podane jawnie w ostrych nawiasach `<>`, są dedukowane na podstawie typów argumentów przekazanych do funkcji.

### Podstawowe reguły dedukcji

* Każdy parametr funkcji może (ale nie musi) brać udział w procesie dedukcji parametru szablonu
* Wszystkie dedukcje są przeprowadzane niezależnie od siebie
* Na zakończenie procesu kompilator sprawdza, czy:

  - każdy parametr szablonu został wydedukowany
  - nie ma konfliktów między wydedukowanymi parametrami
  - wszystkie ograniczenia (*constraints*) są spełnione

* Parametry funkcji biorące udział w dedukcji muszą **ściśle pasować** do typu argumentu - **niejawne konwersje są zabronione**

### Przykłady dedukcji

```cpp
template<typename T, typename U>
void f(T x, U y);

template<typename T>
void g(T x, T y);

template<typename T>
void h(T* ptr);

int main()
{
    // Różne typy dla różnych parametrów - OK
    f(1, 2);      // void f(T, U) [T = int, U = int]
    f(1, 2.5);    // void f(T, U) [T = int, U = double]
    
    // Ten sam typ dla obu parametrów - OK
    g(1, 2);      // void g(T, T) [T = int]
    
    // Konflikt typów - Error
    g(1, 2u);     // Error: no matching function for call to g(int, unsigned int)
    g(1, 2.5);    // Error: no matching function for call to g(int, double)
    
    // Dedukcja ze wskaźników
    int value = 42;
    h(&value);    // void h(T*) [T = int]
}
```

### Rozwiązywanie konfliktów dedukcji

W przypadku, gdy w procesie dedukcji wykryte zostaną konflikty:

```cpp
short short_number = 10;

auto val = max_value(short_number, 42); // ERROR - conflicting types: short vs int
```

istnieją trzy główne rozwiązania:

```cpp
// Rozwiązanie #1: Jawne rzutowanie argumentu
auto val_1 = max_value(static_cast<int>(short_number), 42); // OK

// Rozwiązanie #2: Jawne podanie typu szablonu
auto val_2 = max_value<int>(short_number, 42); // OK - niejawna konwersja short -> int

// Rozwiązanie #3: Użycie szablonu z różnymi typami
template<typename T1, typename T2>
auto max_value(T1 a, T2 b)
{
    return b < a ? a : b;
}

auto val_3 = max_value(short_number, 42); // OK - T1=short, T2=int
```

### Dedukcja arumentów

Dedukcja typów szablonu zależy od sposobu przekazywania argumentów do funkcji. Możemy wyróżnić trzy główne przypadki:

## Dedukcja przy przekazywaniu przez wartość

```cpp
template<typename T>
void deduce_1(T arg){}    // przekazywanie przez wartość

int x = 42;
const int cx = 42;
const int& rx = x;
int buffer[5] = {};
void foo(int);

deduce_1(x);      // T = int (const i & są ignorowane)
deduce_1(cx);     // T = int
deduce_1(rx);     // T = int
deduce_1(buffer); // T = int* (tablice rozpadają się do wskaźników)
deduce_1(foo);    // T = void(*)(int) (funkcje rozpadają się do wskaźników)
```

```{important}
Przy przekazywaniu przez wartość **const** i referencje są ignorowane (usuwane) podczas dedukcji typu szablonu. 

Tablice i funkcje rozpadają się do wskaźników (*decay to pointer*).
```

## Dedukcja przy przekazywaniu przez referencję

```cpp
template<typename T>
void deduce_2(T& arg){}     // przekazywanie przez referencję

deduce_2(x);   // T = int; typ argumentu funcji = int&
deduce_2(cx);  // T = const int; typ argumentu funcji = const int&
deduce_2(rx);  // T = const int; typ argumentu funcji = const int&
deduce_2(buffer); // T = int[5]; typ argumentu funcji = int(&)[5]
deduce_2(foo);    // T - void(int); typ argumentu funcji = void(&)(int)

template<typename T>
void deduce_3(const T& arg){}  // przekazywanie przez const referencję

deduce_3(x);   // T = int; typ argumentu funcji = const int&
deduce_3(cx);  // T = int; typ argumentu funcji = const int&
deduce_3(rx);  // T = int; typ argumentu funcji = const int&
deduce_3(buffer); // T = int[5]; typ argumentu funcji = const int(&)[5]
deduce_3(foo);    // T = void(int); typ argumentu funcji = void(&)(int)
```

```{important}
Przy przekazywaniu parametru funkcji przez referencję **const** jest zachowywane w dedukowanym typie szablonu. Tablice i funkcje **nie rozpadają się** do wskaźników.
```

## Dedukcja przy przekazywaniu przez referencję do r-value

Przekazanie referencji do *r-value* (`T&&`) w kontekście dedukcji typu nazwywane jest przekazaniem **uniwersalnej referencji**.
Wynika to z faktu, że w procesie dedukcji `T&&` dopasuje się zarówno do *l-value* jak i *r-value*.

```cpp
template<typename T>
void deduce_4(T&& arg);    // przekazywanie przez referencję do r-value
{
}

int x = 42;
const int cx = 42;

// passing l-value
deduce_4(x);      // T = int&; typ argumentu funcji = int&
deduce_4(cx);     // T = const int&; typ argumentu funcji = const int&

// passing r-value
deduce_4(42);     // T = int; typ argumentu funcji = int&&
```

```{important}
Dedukcja typu parametrus szablonu oraz typu argumentu funkcji zależy od tego, czy przekazywany argument jest *l-value* czy *r-value*.

1. Przy przekazaniu *l-value* typ szablonu jest dedukowany jako referencja do *l-value* (T& lub const T&), a referencja `T&&` staje się zwykłą referencją do *l-value* (`T&` lub const `T&`). Zamiana rodzaju referencji na *l-value* nazywana jest **kolapsem referencji** (*reference collapsing*).
2. Przy przekazaniu *r-value* typ szablonu jest dedukowany jako zwykły typ (T lub const T), a referencja `T&&` pozostaje referencją do *r-value* (`T&&` lub const `T&&`).
```

## Specjalizacja funkcji szablonowych

**Specjalizacja szablonu funkcji** pozwala na dostarczenie alternatywnej implementacji dla konkretnych typów lub zestawów typów.

```{warning}
Dla szablonów funkcji **nie jest możliwa częściowa specjalizacja**. Możliwe jest jedynie pełne wyspecjalizowanie szablonu dla konkretnych typów.

Zamiast częściowej specjalizacji należy użyć **przeciążenia**.
```

### Pełna specjalizacja (full specialization)

Zdefiniujmy najpierw szablon główny:

```cpp
template <typename T>
bool is_greater(T a, T b)
{
    return a > b;
}
```

Wywołanie tego szablonu dla literałów znakowych `"abc"` i `"def"` utworzy instancję, która porówna adresy wskaźników zamiast zawartości c-łańcuchów. Aby zapewnić prawidłowe porównanie tekstów, dostarczamy specjalizowaną wersję:

```cpp
// Pełna specjalizacja dla const char*
template <>
bool is_greater<const char*>(const char* a, const char* b)
{
    return std::strcmp(a, b) > 0;
}

// Użycie
is_greater(4, 5);      // wywołanie szablonu głównego
is_greater("abc", "def"); // wywołanie wersji specjalizowanej
```

Ponieważ podawanie typu w nawiasach ostrych jest redundantne (kompilator może go wydedukować), specjalizację można zapisać krócej:

```cpp
template <>
bool is_greater(const char* a, const char* b)  // bez <const char*>
{
    return std::strcmp(a, b) > 0;
}
```

```{warning}
Specjalizacje muszą być zadeklarowane **po** szablonie głównym i **przed** jego pierwszym użyciem.
```

### Przeciążanie

W praktyce **zaleca się używanie techniki przeciążenia** zamiast jawnej specjalizacji szablonów funkcji:

```cpp
// Szablon główny
#include <cstring>
#include <iostream>

// Szablon główny
template <typename T>
bool is_greater(T a, T b)
{
    std::cout << "Generic version called\n";
    return a > b;
}

// Przeciążenie szablonu dla wskaźników
template <typename T>
bool is_greater(T* a, T* b)
{
    std::cout << "Pointer version called\n";
    return *a > *b;
}

// Zwykłe przeciążenie dla C-string(nieszablonowe)
bool is_greater(const char* a, const char* b)
{
    std::cout << "C-string version called\n";
    return std::string(a) > std::string(b);
}

int main()
{
    std::cout << is_greater(4, 5) << '\n';               // Generic version
    std::cout << is_greater(4.5, 3.14) << '\n';          // Generic version
    
    std::cout << is_greater("apple", "banana") << '\n';  // C-string version
    
    int x = 10, y = 20;
    std::cout << is_greater(&x, &y) << '\n';             // Pointer version
}
```

### Połączenie przeciążania szablonów, specjalizacji i przeciążania funkcji

W programie może współistnieć (mając tę samą nazwę):

* **Kilka szablonów funkcji** – o różnych sygnaturach parametrów
* **W pełni specjalizowane szablony funkcji**
* **Zwykłe funkcje** – przeciążenia nieszablonowe

Kompletny przykład ilustrujący różne techniki:

```cpp
#include <complex>
#include <iostream>

// Szablon główny
template <typename T> 
T sqrt(T value)
{
    std::cout << "Generic template sqrt<T>\n";
    return value;  // uproszczona implementacja
}

// Pełna specjalizacja szablonu
template <> 
float sqrt(float value)
{
    std::cout << "Specialization sqrt<float>\n";
    return std::sqrt(value);
}

// Przeciążenie szablonu dla std::complex
template <typename T> 
std::complex<T> sqrt(std::complex<T> value)
{
    std::cout << "Overloaded template sqrt for complex<T>\n";
    return std::sqrt(value);
}

// Zwykłe przeciążenie (nieszablonowe)
double sqrt(double value)
{
    std::cout << "Overloaded function sqrt(double)\n";
    return std::sqrt(value);
}

void f(std::complex<double> z)
{
    sqrt(2);        // sqrt<int>(int) - szablon główny
    sqrt(2.0);      // sqrt(double) - przeciążenie nieszablonowe
    sqrt(z);        // sqrt<double>(complex<double>) - przeciążony szablon
    sqrt(3.14f);    // sqrt<float>(float) - specjalizacja
}
```

### Reguły wyboru przeciążenia

Kompilator stosuje następującą hierarchię dopasowania:

1. **Dokładne dopasowanie z funkcją nieszablonową** - najwyższy priorytet
2. **Dokładne dopasowanie z funkcją szablonową** 
3. **Dopasowanie z konwersją dla funkcji nieszablonowej**
4. Błąd kompilacji, jeśli nie można znaleźć dopasowania

## Wskaźniki do funkcji szablonowych

Możliwe jest pobranie adresu funkcji wygenerowanej na podstawie szablonu.

```cpp
template <typename T> 
void process(T value)
{
    std::cout << "Processing value: " << value << "\n";
}

void execute(void (*ptr_fun)(int))
{
    int value = 42;
    ptr_fun(value);
}


// Jawne podanie typu - kompilator generuje process<int>
execute(&process<int>);

// Można też przypisać do zmiennej
void (*ptr_fun)(int) = &process<int>;
ptr_fun(42);
```

### Automatyczna dedukcja typu

W niektórych kontekstach kompilator może wydedukować typ automatycznie:

```cpp
template<typename T>
T add(T a, T b) { return a + b; }

// Dedukcja z typu zmiennej wskaźnikowej
int (*ptr_fun)(int, int) = add;  // dedukuje add<int>

auto result = ptr_fun(3, 4);  // = 7

// Dedukcja przy przekazywaniu do funkcji
using IntBinaryFunc = int(*)(int, int);

auto apply = [](IntBinaryFunc op, int x, int y) {
    return op(x, y);
};
    
apply(add, 10, 20);  // dedukuje add<int>
```

### Wskaźniki do przeciążonych szablonów funckcji

Przy przeciążonych szablonach typ wskaźnika determinuje, która wersja jest wybierana:

```cpp
template<typename T>
void foo(T value) { std::cout << "foo(T)\n"; }

template<typename T>
void foo(T* ptr) { std::cout << "foo(T*)\n"; }

void (*p1)(int) = foo;   // wybiera foo<int>(int)
void (*p2)(int*) = foo;  // wybiera foo<int>(int*)

int x = 42;
p1(x);    // foo(T)
p2(&x);   // foo(T*)
```

## Parametry szablonu dla wartości zwracanych przez funkcję

Gdy funkcja szablonowa ma zwrócić typ inny niż typy jej argumentów, mamy kilka możliwości rozwiązania tego problemu.

### 1. Jawny parametr typu zwracanego

Dodajemy parametr szablonu określający zwracany typ:

```cpp
template<typename TReturn, typename T1, typename T2>
TReturn max_value(T1 a, T2 b)
{
    return b < a ? a : b;
}

// Użycie - typ zwracany musi być podany jawnie
auto result = max_value<double>(4, 7.2);      // TReturn=double, T1=int, T2=double
auto result = max_value<long long>(10, 20);   // TReturn=long long, T1=int, T2=int
```

**Zalety:**

* Pełna kontrola nad typem zwracanym
* Możliwość wybrania dowolnego typu

**Wady:**

* Wymaga jawnego podania typu przy każdym wywołaniu
* Mniej wygodne w użyciu

### 2. Automatyczna dedukcja typu zwracanego (C++14)

Najprostrze rozwiązanie - pozwalamy kompilatorowi wydedukować typ zwracany:

```cpp
template<typename T1, typename T2>
auto max(T1 a, T2 b)
{
    return b < a ? a : b;    // typ dedukowany z wyrażenia return
}

auto result = max(4, 7.2);  // zwraca double (dzięki konwersji 4 -> 4.0)
```

**Zalety:**

* Najprostsze w użyciu
* Automatyczne określanie typu
* Działa dobrze dla prostych przypadków

**Wady:**

* Typ może nie być oczywisty
* Różne instrukcje return muszą zwracać ten sam typ
* Może prowadzić do niespodziewanych konwersji

### 3. Trailing return decltype (C++11)

Jawne określenie typu zwracanego na podstawie wyrażenia:

```cpp
template<typename T1, typename T2>
auto max(T1 a, T2 b) -> decltype(b < a ? a : b)
{
    return b < a ? a : b;
}

// lub używając decltype(auto) w C++14
template<typename T1, typename T2>
decltype(auto) max(T1 a, T2 b)
{
    return b < a ? a : b;
}
```

**Zalety:**

* Precyzyjna kontrola nad typem
* Zachowuje kwalifikatory (const, &, &&)

**Wady:**

* Bardziej skomplikowana składnia
* `decltype` może produkować referencje - należy być ostrożnym gdy zwracamy lokalne zmienne lub tymczasowe obiekty!!!

### 4. Użycie type traits

Klasy cech mogą definiować typ zwracany na podstawie typów argumentów.

Poniższy przykład wykorzystuje cechę `std::common_type` z biblioteki standardowej do określenia typy zwracanego, jako wspólnego typu, do którego można bezpiecznie przekonwertować oba argumenty:

```cpp
#include <type_traits>

template<typename T1, typename T2>
std::common_type_t<T1, T2> max(T1 a, T2 b)
{
    return b < a ? a : b;
}

auto result1 = max(4, 7.2);      // zwraca double
auto result2 = max(4L, 7);       // zwraca long
auto result3 = max(4.0f, 7.0);   // zwraca double
```

**Zalety:**

* Przewidywalne zachowanie
* Precyzyjna kontrola nad typem zwracanym

**Wady:**

* Może nie działać dla typów użytkownika (technika wymaga specjalizacji szablonu cechy)


## Domyślne parametry szablonu

Definiując parametry szablonu, możemy określić dla nich wartości domyślne. Mogą one odwoływać się do wcześniej zdefiniowanych parametrów szablonu.

```cpp
#include <type_traits>

template<typename T1, typename T2, typename RT = std::common_type_t<T1,T2>>
RT max (T1 a, T2 b)
{
    return b < a ? a : b;
}
```

Wywołując szablon funkcji możemy pominąć argumenty z domyślnymi wartościami:

```cpp
auto val_1 = max_value(1, 3.14); // max_value<int, double, double>
```

lub jawnie podać odpowiedni argument:

```cpp
auto val_2 = max_value<int, short, double>(1, 4);
```