# SFINAE - enable_if

Jednym z podstawowych narzędzi w meta-programowaniu w C++ jest szablon
**enable_if** wykorzystujący mechanizm **SFINAE**.

Motywacją dla stosowania `enable_if` jest chęć dostarczenia dla
klientów uniwersalnego interfejsu, bez zmuszania ich do podejmowania
decyzji o wyborze określonych implementacji. Decyzje takie są
podejmowane przez implementorów bibliotek. Dobór optymalnych
implementacji w zależności od typów danych dokonywany jest na etapie
kompilacji.

Chcemy uniknąć kodu, który często wygląda tak:

```cpp
// Don't even dare to pass an array of complex objects to this function!!!
template <typename T>
T* copy_it(const T* src, size_t n)
{
    // impl
}
```

## SFINAE

**Substitition Failure Is Not An Error (SFINAE)** jest mechanizmem
wykorzystywanym przez kompilator w trakcie dedukcji typów dla argumentów
szablonu. Szablony, dla których nie udaje się podstawić określonego typu
jako typu argumentu szablonu są ignorowane i nie powodują błędów
kompilacji.

W praktyce jeśli dedukcja typów argumentów dla szablonu funkcji znajdzie
przynajmniej jedno dopasowanie, błędne podstawienia są usuwane z listy
funkcji kandydujących do wywołania (*overload resolution*) i nie
zgłaszają błędów kompilacji.

Załóżmy następującą implementację szablonów funkcji:

```cpp
template <typename T> void foo(T arg)
{}

template<typename T> void foo(T* arg)
{}
```

Wywołanie `foo(42)` powoduje utworzenie instancji szablonu funkcji
`foo<int>(int)`. Ponieważ kompilatorowi nie udaje się podstawić
prawidłowo typu `int` dla drugiej implementacji działa SFINAE -
nieudane podstawienie nie generuje błędów kompilacji a implementacja
znika z listy funkcji kandydujących do wywołania. Jeśli brakowałoby
pierwszej implementacji funkcji (tej prawidłowo dopasowanej) kompilator
zgłosiłby błąd.

## Szablon enable_if

Rozważmy dwie implementacje funkcji z identycznymi interfejsami, ale
różnymi wymaganiami odnośnie typu danych:

```cpp
template <typename T>
void do_stuff(T t) // for small objects
{
    //~~~
}

template <typename T>
void do_stuff(const T& t) // for large objects
{
    //~~~
}
```

Sygnatury tych funkcji są zbyt podobne, aby można było zastosować
klasyczne przeciążenie funkcji. Rozwiązaniem tego problemu jest
zastosowanie szablonu `enable_if` i mechanizmu SFINAE.

Typowa implementacja szablonu `enable_if` wygląda następująco:

```cpp
template <bool Condition, typename T = void>
struct enable_if
{
    using type = T;
};

template <typename T>
struct enable_if<false, T>
{};
```

Szablon ogólny jest sparametryzowany wartością logiczną (zwykle
wyrażenie logiczne, które jest ewaluowane na etapie kompilacji) oraz
typem (domyślnie `void`). Wewnątrz struktury definiowany jest typ
`type`. W wersji specjalizowanej dla wartości logicznej `false`
takiej definicji typu brakuje.

- Jeśli wyrażenie `Condition` przekazane jako pierwszy argument
  szablonu ewaluowane jest do wartości `true`:
   
  -  `enable_if<Condition>` ma składową `type`
  -  i `enable_if<Condition>::type` odnosi się do tego typu

- Jeśli wyrażenie `Condition` ewaluowane jest do `false`:

  -  `enable_if<Condition>` nie ma składowej `type`
  -  i `enable_if<Condition>::type` jest nieprawidłowym odwołaniem
  -  w rezultacie podstawienie nie udaje się (działa SFINAE)

W celu ułatwienia korzystania z `enable_if` C++14 definiuje poniższy
alias szablonu:

```cpp
template <bool Condition, typename T = void>
using enable_if_t = typename enable_if<Condition, T>::type;
```

Wykorzystanie szablonu `enable_if` do problemu wyboru implementacji
wygląda następująco:

```cpp
#include <iostream>

template <typename T>
typename enable_if<(sizeof(T) < 8)>::type do_stuff(T t) // for small objects
{
    std::cout << "do_stuff(Small Object)\n";
}

template <typename T>
enable_if_t<(sizeof(T) > 8)> do_stuff(T const& t) // for large objects
{
    std::cout << "do_stuff(Large Object)\n";
}
```

```cpp
#include <string>

do_stuff('A'); // OK, small object
do_stuff(std::string("abc")); // OK, large object
```

```
do_stuff(Small Object)
do_stuff(Large Object)
```

W zależności od rozmiaru typu wydedukowanego w procesie tworzenia
instancji szablonu jest tworzona i później wywoływana odpowiednia
instancja szablonu funkcji `do_stuff()`.

Próba utworzenia instancji dla typu `double` zgłaszany jest błąd
kompilacji:

```cpp
do_stuff(3.14);
```

```
error: no matching function for call to do_stuff
    note: candidate template ignored: disabled by enable_if [with T = double]
```

Kompilator nie jest w stanie wygenerować żadnej instancji szablonu dla
typu o rozmiarze 8 bajtów.

Mechanizm SFINAE zapewnia dwufazowe dopasowanie funkcji do wywołania
(*two-phase overload resolution*): 

1. W fazie pierwszej `enable_if` i SFINAE eliminują funkcje kandydujące, dla których nie można 
   zrealizować podstawienia 
2. W fazie drugiej dopasowana może zostać tylko jedna funkcja z grupy funkcji kandydujących do wywołania

## Cechy typów i enable_if

Często jako w wyrażeniu logicznym, będącym pierwszym argumentem szablonu
`enable_if`, wykorzystywane są cechy typów:

```cpp
#include <type_traits>

template <typename T>
auto store_item(T const& item) -> std::enable_if_t<std::is_trivially_copyable_v<T>>
{
    // memcpy or similar operation
    alignas(T) std::byte data[sizeof(T)];
    std::memcpy(data, &item, sizeof(T));

    auto& item_copy = *std::start_lifetime_as<T>(data);
}

template <typename T>
auto store_item(T const& item) -> std::enable_if_t<!std::is_trivially_copyable_v<T>>
{
    T item_copy = item; // use copy constructor
}
```

SFINAE może zostać użyte również do wprowadzenia ograniczeń jeśli chodzi
i zakres typów, dla których realizowane jest tworzenie instancji
szablonu:

```cpp
struct Shape
{
    //...
};

struct Rectangle : Shape
{
    //...
};
```

```cpp
template <typename T>
auto do_stuff_with_shape(const T& shape) -> enable_if_t<std::is_base_of_v<Shape, T>>
{
    //...
}
```

## Domyślne argumenty szablonów funkcji

W C++11 argumenty szablonów funkcji mogą przyjmować wartości domyślne.

Pozwala to na czytelniejszą dla programisty implementację SFINAE z
`enable_if`:

```cpp
template <
    typename T,
    typename = std::enable_if_t<std::is_base_of_v<Shape, T>>
>
void do_other_stuff_with_shape(const T& shape) 
{
    //...    
}
```

Aby podnieść jeszcze bardziej czytelność kodu możemy zdefiniować
pomocniczy alias:

```cpp
template <typename T>
using IsShape = std::enable_if_t<std::is_base_of_v<Shape, T>>;
```

i wykorzystać go do implementacji szablonu funkcji z ograniczeniem
SFINAE:

```cpp
template <
    typename T,
    typename = IsShape<T>
>
void do_other_stuff_with_shape_alt(const T& shape) 
{
    //...    
}
```

## Ograniczenia w szablonach klas

SFINAE oraz `enable_if` mogą również zostać użyte dla szablonów klas.

Możemy na przykład ograniczyć możliwość tworzenia instancji szablonów
dla typów zmiennoprzecinkowych:

```cpp
template <typename T>
using FloatingPoint = std::enable_if_t<std::is_floating_point_v<T>>;

template <
    typename T,
    typename = FloatingPoint<T>
> 
class Data
{
    //...
};
```

```cpp
Data<double> d1; // OK

Data<int> d2;    // Error: SFINAE out
```

## SFINAE i przeciążone konstruktory

SFINAE może rozwiązać problemy związane z przeciążonymi konstruktorami
klas i ich czasami zaskakującym zachowaniem.

Załóżmy, że implementujemy klasę `Heap`, która potrzebuje dwóch wersji
konstruktorów:

```cpp
#include <iostream>

template <typename T>
class Heap
{
public:
    Heap(size_t n, T const& v)
    {
        std::cout << "Heap(size_t, T)\n"; 
    }
    
    template <typename InIt>
    Heap(InIt start, InIt end)  // range init using pair of input iterators
    {
        std::cout << "Heap(InIt, InIt)" << std::endl;
    }
};
```

Obecność konstruktora definiowanego jako szablon może dać zaskakujący
rezultat:

```cpp
Heap<int> h(5, 0); // range init constructor called
```

```
Heap(InIt, InIt)
```

Powyższe kod spowoduje wywołanie konstruktora przyjmującego jako
argumenty parę iteratorów, ponieważ ta wersja konstruktora gwarantuje
lepsze dopasowanie argumentów.

Możemy uniknąć takiego zachowania wprowadzając ograniczenie dla
konstruktora wykorzystującego iteratory:

```cpp
#include <iterator>

template <typename It>
using Category = typename std::iterator_traits<It>::iterator_category;

template <typename It>
using InputIterator = std::is_base_of<std::input_iterator_tag, Category<It>>;

template <typename It>
using IsInputIterator_t = std::enable_if_t<InputIterator<It>::value>;
```

```cpp
namespace Sfinae
{
    template <typename T>
    class Heap
    {
    public:
        Heap(size_t n, T const& v)
        {
            std::cout << "Heap(size_t, T)\n"; 
        }

        template <
            typename InIt,
            typename = IsInputIterator_t<InIt>
        >
        Heap(InIt start, InIt end)  // range init using pair of input iterators
        {
            std::cout << "Heap(InIt, InIt)" << std::endl;
        }
    };
}
```

Teraz przy wywołaniu

```cpp
Sfinae::Heap<int>(5, 0);
```

```
Heap(size_t, T)
```

konstruktor z iteratorami znika z funkcji kandydujących do wywołania.

Natomiast, gdy przekażemy konstruktorowi zakres zdefiniowany przez
iteratory, to odpowiednia wersja konstruktora jest instancjonowana i
później wywołana.

```cpp
auto range = { 1, 2, 3 };

Sfinae::Heap<int>(range.begin(), range.end());
```

```
Heap(InIt, InIt)
```
