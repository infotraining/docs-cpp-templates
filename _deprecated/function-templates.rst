Szablony funkcji
================

**Szablon funkcji** - funkcja, której typy argumentów lub typ zwracanej wartości zostały sparametryzowane. Szablony funkcji są fundamentalnym mechanizmem programowania generycznego w C++, pozwalającym na pisanie kodu wielokrotnego użytku, który działa z różnymi typami danych przy zachowaniu bezpieczeństwa typów w czasie kompilacji.

Składnia definicji szablonu funkcji
-----------------------------------

.. code-block:: cpp

    
    template <typename T>  // lub template <class T>
    return_type function_name(parameters)
    {
        // implementacja
    }

    // Wiele parametrów szablonu
    template <typename T, typename U, typename R>
    R function_name(T arg1, U arg2)
    {
        // implementacja
    }

.. note::
   Słowa kluczowe ``typename`` i ``class`` są w tym kontekście równoważne. Zaleca się używanie ``typename`` dla lepszej czytelności.

Przykłady szablonów funkcji
---------------------------

Prosty szablon funkcji ``max_value()``
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
.. code-block:: cpp

    template<typename T>
    T max_value(T a, T b)
    {
        return b < a ? a : b;
    }

Użycie szablonu funkcji ``max_value()``
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. code-block:: cpp

    // Jawne podanie typu
    int val1 = max_value<int>(4, 9);

    // Automatyczna dedukcja typu
    double x = 4.50;
    double y = 3.14;
    double val2 = max_value(x, y);  // T dedukowane jako double

    // Działa z dowolnym typem implementującym operator<
    std::string s1 = "mathematics";
    std::string s2 = "math";
    std::cout << "max(s1, s2) = " << max_value(s1, s2) << '\n';

Zaawansowane przykłady
~~~~~~~~~~~~~~~~~~~~~

.. code-block:: cpp

    // Szablon z referencjami
    template<typename T>
    const T& max_value(const T& a, const T& b)
    {
        return b < a ? a : b;  // zwraca referencję, unika kopiowania
    }

    // Szablon z forwarding references (C++11)
    template<typename T>
    void process(T&& arg)
    {
        // perfect forwarding
        do_something(std::forward<T>(arg));
    }

    // Szablon z wieloma parametrami
    template<typename T, typename U>
    auto add(T a, U b) -> decltype(a + b)  // C++11
    {
        return a + b;
    }

    // lub w C++14
    template<typename T, typename U>
    auto add(T a, U b)
    {
        return a + b;
    }


Dedukcja typów argumentów szablonu
----------------------------------

W momencie wywołania funkcji szablonowej następuje proces **dedukcji typów argumentów szablonu** (*template argument deduction*). Typy parametrów szablonu, które nie zostały podane jawnie w ostrych nawiasach ``<>``, są dedukowane na podstawie typów argumentów przekazanych do funkcji.

Podstawowe reguły dedukcji
~~~~~~~~~~~~~~~~~~~~~~~~~~

* Każdy parametr funkcji może (ale nie musi) brać udział w procesie dedukcji parametru szablonu
* Wszystkie dedukcje są przeprowadzane niezależnie od siebie
* Na zakończenie procesu kompilator sprawdza, czy:

  - każdy parametr szablonu został wydedukowany
  - nie ma konfliktów między wydedukowanymi parametrami
  - wszystkie ograniczenia (*constraints*) są spełnione

* Parametry funkcji biorące udział w dedukcji muszą **ściśle pasować** do typu argumentu - **niejawne konwersje są zabronione**

Przykłady dedukcji
~~~~~~~~~~~~~~~~~

.. code-block:: c++

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
        
        // Konflikt typów - ERROR
        g(1, 2u);     // error: no matching function for call to g(int, unsigned int)
        g(1, 2.5);    // error: no matching function for call to g(int, double)
        
        // Dedukcja ze wskaźników
        int value = 42;
        h(&value);    // void h(T*) [T = int]
    }

Rozwiązywanie konfliktów dedukcji
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

W przypadku, gdy w procesie dedukcji wykryte zostaną konflikty:

.. code-block:: c++

    short f();

    auto val = max_value(f(), 42); // ERROR - conflicting types: short vs int

istnieją trzy główne rozwiązania:

.. code-block:: c++

    // Rozwiązanie #1: Jawne rzutowanie argumentu
    auto val1 = max_value(static_cast<int>(f()), 42); // OK

    // Rozwiązanie #2: Jawne podanie typu szablonu
    auto val2 = max_value<int>(f(), 42); // OK - niejawna konwersja short -> int

    // Rozwiązanie #3: Użycie szablonu z różnymi typami
    template<typename T1, typename T2>
    auto max(T1 a, T2 b)
    {
        return b < a ? a : b;
    }
    
    auto val3 = max_value(f(), 42); // OK - T1=short, T2=int

Dedukcja z const i referencji
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. code-block:: c++

    template<typename T>
    void f1(T arg);      // przekazywanie przez wartość
    
    template<typename T>
    void f2(T& arg);     // przekazywanie przez referencję
    
    template<typename T>
    void f3(const T& arg);  // przekazywanie przez const referencję

    int x = 42;
    const int cx = 42;
    const int& rx = x;

    f1(x);   // T = int (const i & są ignorowane)
    f1(cx);  // T = int
    f1(rx);  // T = int

    f2(x);   // T = int
    f2(cx);  // T = const int
    f2(rx);  // T = const int

    f3(x);   // T = int (const w parametrze, nie w T)
    f3(cx);  // T = int
    f3(rx);  // T = int

Tworzenie instancji szablonu
----------------------------

**Instancjacja szablonu** (*template instantiation*) to proces, w którym kompilator generuje konkretny kod funkcji na podstawie definicji szablonu i dostarczonych argumentów typu.

Model kompilacji szablonów
~~~~~~~~~~~~~~~~~~~~~~~~~~

Szablony stosują specyficzny model kompilacji, który różni się od zwykłych funkcji:

* Cały kod szablonu musi być dostępny w miejscu użycia (zazwyczaj w pliku nagłówkowym)
* Kompilator generuje kod dla każdej unikalnej kombinacji parametrów szablonu
* Proces ten nazywany jest **kompilacją "inclusion model"**

.. code-block:: cpp

    // max.hpp
    #ifndef MAX_HPP
    #define MAX_HPP

    template<typename T>
    T max(T a, T b)
    {
        return b < a ? a : b;
    }

    #endif

    // main.cpp
    #include "max.hpp"

    int main()
    {
        max(1, 2);      // instancjacja dla int
        max(1.5, 2.5);  // instancjacja dla double
    }

Rodzaje instancjacji
~~~~~~~~~~~~~~~~~~~

Instancjacja niejawna (implicit instantiation)
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Kompilator automatycznie generuje kod szablonu w miejscu pierwszego użycia:

.. code-block:: cpp

    template<typename T>
    T add(T a, T b) { return a + b; }

    int x = add(1, 2);        // niejawna instancjacja dla int
    double y = add(1.5, 2.5); // niejawna instancjacja dla double

Instancjacja jawna (explicit instantiation)
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Programista może wymusić wygenerowanie konkretnej instancji szablonu:

.. code-block:: cpp

    // Deklaracja jawnej instancjacji (w pliku nagłówkowym)
    extern template int max<int>(int, int);

    // Definicja jawnej instancjacji (w pliku .cpp)
    template int max<int>(int, int);

Zalety jawnej instancjacji:

* Skrócenie czasu kompilacji (kod generowany raz)
* Zmniejszenie rozmiaru plików obiektowych
* Kontrola nad tym, gdzie kod jest generowany

Wymagania wobec typów
~~~~~~~~~~~~~~~~~~~~~

Utworzenie instancji szablonu jest możliwe tylko wtedy, gdy dla typu podanego jako argument szablonu zdefiniowane są wszystkie operacje używane przez szablon.

.. code-block:: cpp

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

Fazy kompilacji szablonu
~~~~~~~~~~~~~~~~~~~~~~~~

Kompilacja szablonów przebiega w **dwóch fazach** (*two-phase lookup*):

**Faza 1: Analiza definicji szablonu**

Na etapie definicji szablonu (przed tworzeniem jakiejkolwiek instancji) sprawdzana jest:

* poprawność składniowa
* użycie nazw **niezależnych od parametrów szablonu** (*non-dependent names*)
* statyczne asercje, które nie zależą od parametru szablonu

**Faza 2: Analiza instancji szablonu**

Podczas tworzenia instancji szablonu kod jest ponownie sprawdzany, ze szczególnym uwzględnieniem:

* nazw **zależnych od parametrów szablonu** (*dependent names*)
* wymagań dotyczących operacji na typach parametrów
* statycznych asercji zależnych od parametrów

.. code-block:: c++

    template<typename T>
    void foo(T t)
    {
        undeclared();   // Błąd fazy 1: jeśli undeclared() jest nieznane
        undeclared(t);  // Błąd fazy 2: jeśli undeclared(T) jest nieznane dla danego T
        
        static_assert(sizeof(int) > 10, "int too small");  // Faza 1: zawsze błąd jeśli warunek nie jest spełniony
        static_assert(sizeof(T) > 10, "T too small");      // Faza 2: błąd tylko dla T o rozmiarze <= 10
    }

Przykład dwufazowej kompilacji:

.. code-block:: cpp

    void helper(int) { }  // (1)

    template<typename T>
    void process(T value)
    {
        helper(value);    // niezależne od T - szukane w fazie 1
        value.method();   // zależne od T - szukane w fazie 2
    }

    void helper(double) { }  // (2) - za późno dla process<int>

    int main()
    {
        process(42);      // używa helper(1), nie helper(2)
        process(3.14);    // używa helper(2)
    }

Specjalizacja funkcji szablonowych
----------------------------------

**Specjalizacja szablonu funkcji** pozwala na dostarczenie alternatywnej implementacji dla konkretnych typów lub zestawów typów.

Pełna specjalizacja (full specialization)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Zdefiniujmy najpierw szablon główny:

.. code-block:: cpp
    
    template <typename T>
    bool is_greater(T a, T b)
    {
        return a > b;
    }

Wywołanie tego szablonu dla literałów znakowych `"abc"` i `"def"` utworzy instancję, która porówna adresy wskaźników zamiast zawartości c-łańcuchów. Aby zapewnić prawidłowe porównanie tekstów, dostarczamy specjalizowaną wersję:
  
.. code-block:: cpp

    // Pełna specjalizacja dla const char*
    template <>
    bool is_greater<const char*>(const char* a, const char* b)
    {
        return std::strcmp(a, b) > 0;
    }

    // Użycie
    is_greater(4, 5);      // wywołanie szablonu głównego
    is_greater("abc", "def"); // wywołanie wersji specjalizowanej

Ponieważ podawanie typu w nawiasach ostrych jest redundantne (kompilator może go wydedukować), specjalizację można zapisać krócej:

.. code-block:: cpp

    template <>
    bool is_greater(const char* a, const char* b)  // bez <const char*>
    {
        return std::strcmp(a, b) > 0;
    }

.. warning::
   Specjalizacje muszą być zadeklarowane **po** szablonie głównym i **przed** jego pierwszym użyciem.

Częściowa specjalizacja - NIE dla funkcji!
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

W przeciwieństwie do szablonów klas, **szablony funkcji nie obsługują częściowej specjalizacji**. Próba utworzenia częściowej specjalizacji zakończy się błędem kompilacji:

.. code-block:: cpp

    template<typename T>
    void process(T value) { /* ... */ }

    // BŁĄD: częściowa specjalizacja niedozwolona dla funkcji
    // template<typename T>
    // void process<T*>(T* ptr) { /* ... */ }

Zamiast częściowej specjalizacji należy użyć **przeciążenia**.

Przeciążanie vs. specjalizacja
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

W praktyce **zaleca się używanie przeciążenia** zamiast jawnej specjalizacji szablonów funkcji:

.. code-block:: cpp

    // Szablon główny
    template <typename T>
    bool is_greater(T a, T b)
    {
        return a > b;
    }

    // Przeciążenie (preferowane nad specjalizacją!)
    bool is_greater(const char* a, const char* b)
    {
        return std::strcmp(a, b) > 0;
    }

**Zalety przeciążenia:**

* Prostsze w użyciu i bardziej intuicyjne
* Uczestniczy w rozwiązywaniu przeciążeń w bardziej przewidywalny sposób
* Może być częściowo specjalizowane (np. dla wskaźników)
* Lepiej współpracuje z innymi przeciążeniami

**Kolejność rozwiązywania:**

1. Funkcje nieszablonowe (zwykłe przeciążenia)
2. Szablony funkcji
3. Specjalizacje szablonów funkcji

.. code-block:: cpp

    template<typename T>
    void foo(T) { std::cout << "szablon\n"; }

    template<>
    void foo<int>(int) { std::cout << "specjalizacja\n"; }

    void foo(int) { std::cout << "przeciążenie\n"; }

    foo(42);  // wywoła "przeciążenie" (najwyższy priorytet)

Połączenie przeciążania szablonów, specjalizacji i przeciążania funkcji
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Kompletny przykład ilustrujący różne techniki:

.. code-block:: cpp

    #include <complex>
    #include <iostream>

    // Szablon główny
    template <typename T> 
    T sqrt(T value)
    {
        std::cout << "szablon główny sqrt<T>\n";
        return value;  // uproszczona implementacja
    }

    // Pełna specjalizacja szablonu
    template <> 
    float sqrt(float value)
    {
        std::cout << "specjalizacja sqrt<float>\n";
        return std::sqrt(value);
    }

    // Przeciążenie szablonu dla std::complex
    template <typename T> 
    std::complex<T> sqrt(std::complex<T> value)
    {
        std::cout << "przeciążony szablon sqrt dla complex<T>\n";
        return std::sqrt(value);
    }

    // Zwykłe przeciążenie (nieszablonowe)
    double sqrt(double value)
    {
        std::cout << "przeciążona funkcja sqrt(double)\n";
        return std::sqrt(value);
    }
    
    void f(std::complex<double> z)
    {
        sqrt(2);        // sqrt<int>(int) - szablon główny
        sqrt(2.0);      // sqrt(double) - przeciążenie nieszablonowe
        sqrt(z);        // sqrt<double>(complex<double>) - przeciążony szablon
        sqrt(3.14f);    // sqrt<float>(float) - specjalizacja
    }

Reguły wyboru przeciążenia
^^^^^^^^^^^^^^^^^^^^^^^^^^

Kompilator stosuje następującą hierarchię dopasowania:

1. **Dokładne dopasowanie z funkcją nieszablonową** - najwyższy priorytet
2. **Dokładne dopasowanie z funkcją szablonową**
3. **Dopasowanie z konwersją dla funkcji nieszablonowej**
4. Błąd kompilacji, jeśli nie można znaleźć dopasowania

Przeciążanie szablonów
----------------------

W programie może współistnieć (mając tę samą nazwę):

* **Kilka szablonów funkcji** – o różnych sygnaturach parametrów
* **Funkcje specjalizowane** – pełne specjalizacje szablonów
* **Zwykłe funkcje** – przeciążenia nieszablonowe

Przykłady przeciążania
~~~~~~~~~~~~~~~~~~~~~~

.. code-block:: cpp

    // Szablon główny
    template<typename T>
    void print(const T& value)
    {
        std::cout << "szablon: " << value << '\n';
    }

    // Przeciążenie szablonu dla wskaźników
    template<typename T>
    void print(T* ptr)
    {
        if (ptr)
            std::cout << "wskaźnik: " << *ptr << '\n';
        else
            std::cout << "nullptr\n";
    }

    // Przeciążenie szablonu dla std::vector
    template<typename T>
    void print(const std::vector<T>& vec)
    {
        std::cout << "vector: ";
        for (const auto& elem : vec)
            std::cout << elem << ' ';
        std::cout << '\n';
    }

    // Zwykłe przeciążenie
    void print(const char* str)
    {
        std::cout << "c-string: " << str << '\n';
    }

    int main()
    {
        int x = 42;
        print(x);           // szablon główny
        print(&x);          // szablon dla wskaźników
        print("hello");     // zwykłe przeciążenie
        print(std::vector{1, 2, 3});  // szablon dla vector
    }

Reguły przeciążania szablonów
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

1. **Przeciążenia muszą się różnić sygnaturą** - nie można przeciążyć tylko przez zmianę typu zwracanego
2. **Szablon bardziej wyspecjalizowany ma pierwszeństwo** - kompilator wybiera najbardziej konkretne dopasowanie
3. **Funkcja nieszablonowa ma pierwszeństwo** przed szablonem przy równym dopasowaniu
4. **Dedukcja musi się udać** - jeśli kompilator nie może wydedukować typów, szablon nie jest brany pod uwagę

.. code-block:: cpp

    template<typename T>
    void foo(T) { std::cout << "szablon 1\n"; }

    template<typename T>
    void foo(T*) { std::cout << "szablon 2 (bardziej wyspecjalizowany)\n"; }

    void foo(int*) { std::cout << "funkcja nieszablonowa\n"; }

    int x = 42;
    foo(x);    // szablon 1
    foo(&x);   // funkcja nieszablonowa (priorytet nad szablonem 2)
    
    double y = 3.14;
    foo(&y);   // szablon 2 (brak nieszablonowej dla double*)

Wskaźniki do funkcji szablonowych
---------------------------------

Możliwe jest pobranie adresu funkcji wygenerowanej na podstawie szablonu.

Podstawy
~~~~~~~~

.. code-block:: cpp

    template <typename T> 
    void process(T* ptr)
    {
        std::cout << "przetwarzanie wskaźnika typu T*\n";
    }

    void execute(void (*pf)(int*))
    {
        int value = 42;
        pf(&value);
    }

    int main() 
    {
        // Jawne podanie typu - kompilator generuje process<int>
        execute(&process<int>);
        
        // Można też przypisać do zmiennej
        void (*ptr)(int*) = &process<int>;
        ptr(&some_int);
    }

Automatyczna dedukcja typu
~~~~~~~~~~~~~~~~~~~~~~~~~~

W niektórych kontekstach kompilator może wydedukować typ automatycznie:

.. code-block:: cpp

    template<typename T>
    T add(T a, T b) { return a + b; }

    int main()
    {
        // Dedukcja z typu zmiennej wskaźnikowej
        int (*func_ptr)(int, int) = add;  // dedukuje add<int>
        
        auto result = func_ptr(3, 4);  // = 7
        
        // Dedukcja przy przekazywaniu do funkcji
        using IntBinaryOp = int(*)(int, int);
        
        auto apply = [](IntBinaryOp op, int x, int y) {
            return op(x, y);
        };
        
        apply(add, 10, 20);  // dedukuje add<int>
    }

Wskaźniki do przeciążonych szablonów
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Przy przeciążonych szablonach typ wskaźnika determinuje, która wersja jest wybierana:

.. code-block:: cpp

    template<typename T>
    void foo(T value) { std::cout << "foo(T)\n"; }

    template<typename T>
    void foo(T* ptr) { std::cout << "foo(T*)\n"; }

    int main()
    {
        void (*p1)(int) = foo;   // wybiera foo<int>(int)
        void (*p2)(int*) = foo;  // wybiera foo<int>(int*)
        
        int x = 42;
        p1(x);    // foo(T)
        p2(&x);   // foo(T*)
    }

Parametry szablonu dla wartości zwracanych przez funkcję
--------------------------------------------------------

Gdy funkcja szablonowa ma zwrócić typ inny niż typy jej argumentów, mamy kilka możliwości rozwiązania tego problemu.

1. Jawny parametr typu zwracanego
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Dodajemy parametr szablonu określający zwracany typ:

.. code-block:: c++

    template<typename TReturn, typename T1, typename T2>
    TReturn max(T1 a, T2 b)
    {
        return b < a ? a : b;
    }

    // Użycie - typ zwracany musi być podany jawnie
    auto result = max<double>(4, 7.2);      // TReturn=double, T1=int, T2=double
    auto result = max<long long>(10, 20);   // TReturn=long long, T1=int, T2=int

**Zalety:**

* Pełna kontrola nad typem zwracanym
* Możliwość wybrania dowolnego typu

**Wady:**

* Wymaga jawnego podania typu przy każdym wywołaniu
* Mniej wygodne w użyciu

2. Automatyczna dedukcja typu zwracanego (C++14)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Najprostrze rozwiązanie - pozwalamy kompilatorowi wydedukować typ zwracany:

.. code-block:: c++

    template<typename T1, typename T2>
    auto max(T1 a, T2 b)
    {
        return b < a ? a : b;  // typ dedukowany z wyrażenia return
    }

    auto result = max(4, 7.2);  // zwraca double (dzięki konwersji 4 -> 4.0)

**Zalety:**

* Najprostsze w użyciu
* Automatyczne określanie typu
* Działa dobrze dla prostych przypadków

**Wady:**

* Typ może nie być oczywisty
* Różne instrukcje return muszą zwracać ten sam typ
* Może prowadzić do niespodziewanych konwersji

3. Trailing return type z decltype (C++11)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Jawne określenie typu zwracanego na podstawie wyrażenia:

.. code-block:: c++

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

**Zalety:**

* Precyzyjna kontrola nad typem
* Zachowuje kwalifikatory (const, &, &&)

**Wady:**

* Bardziej skomplikowana składnia
* ``decltype`` może produkować referencje

4. Użycie type traits
~~~~~~~~~~~~~~~~~~~~~

Wykorzystanie biblioteki standardowej do określenia wspólnego typu:

.. code-block:: c++

    #include <type_traits>

    template<typename T1, typename T2>
    std::common_type_t<T1, T2> max(T1 a, T2 b)
    {
        return b < a ? a : b;
    }

    auto result1 = max(4, 7.2);      // zwraca double
    auto result2 = max(4L, 7);       // zwraca long
    auto result3 = max(4.0f, 7.0);   // zwraca double

**Zalety:**

* Używa standardowych reguł konwersji typów
* Przewidywalne zachowanie
* Zgodne z semantyką języka

**Wady:**

* Wymaga ``#include <type_traits>``
* Może nie działać dla typów użytkownika (wymaga specjalizacji ``std::common_type``)

Zaawansowany przykład z wieloma opcjami
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. code-block:: c++

    #include <type_traits>

    // Wersja z common_type
    template<typename T1, typename T2>
    std::common_type_t<T1, T2> max_v1(T1 a, T2 b)
    {
        return b < a ? a : b;
    }

    // Wersja z auto (C++14)
    template<typename T1, typename T2>
    auto max_v2(T1 a, T2 b)
    {
        return b < a ? a : b;
    }

    // Wersja z jawnym typem zwracanym
    template<typename RT, typename T1, typename T2>
    RT max_v3(T1 a, T2 b)
    {
        return b < a ? static_cast<RT>(a) : static_cast<RT>(b);
    }

    int main()
    {
        auto r1 = max_v1(4, 7.2);           // double
        auto r2 = max_v2(4, 7.2);           // double
        auto r3 = max_v3<float>(4, 7.2);    // float (jawna konwersja)
    }

Domyślne parametry szablonu
---------------------------

Definiują parametry szablonu, możemy określić dla nich wartości domyślne. Mogą one odwoływać się do wcześniej zdefiniowanych parametrów szablonu.

.. code-block:: c++

    #include <type_traits>

    template<typename T1, typename T2, typename RT = std::common_type_t<T1,T2>>
    RT max (T1 a, T2 b)
    {
        return b < a ? a : b;
    }

Wywołując szablon funkcji możemy pominąć argumenty z domyślnymi wartościami:

.. code-block:: c++

    auto val_1 = max_value(1, 3.14); // max_value<int, double, double>

lub jawnie podać odpowiedni argument:

.. code-block:: c++

    auto val_2 = max_value<int, short, double>(1, 4); 