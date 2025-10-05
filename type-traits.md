# Klasy cech typów - biblioteka <type_traits>

Biblioteka standardowa zawiera zestaw szablonów klas umożliwiających
cechowanie typów na etapie kompilacji.

Dzięki cechom typów (**type traits**) na etapie kompilacji:

* wybierana jest optymalna (wydajna) implementacja danej funkcjonalności (z wykorzystaniem SFINAE). 
* przeprowadzana jest statyczna asercja umożliwiająca wczesne wychwycenie błędów

## Meta-funkcje

Technika cechowania wykorzystuje meta-funkcje, najczęściej implementowane jako szablony klas przyjmujące jako parametr
cechowany typ i zwracające: 

* `type_trait<T>::value` - stałą statyczną wartość `constexpr` (najczęściej typu `bool`)
* `type_trait<T>::type` - nowy typ będący efektem transformacji (np. usunięcia modyfikatora `const`)

### Meta-funkcje zwracające wartość

Przykład implementacji meta-funkcji zwracającej stałą wartość ewaluowaną na etapie kompilacji:

```cpp
template <typename T, T Constant>
struct IntegralConstant
{
    static constexpr T value = Constant;
};

static_assert(IntegralConstant<int, 5>::value == 5);
static_assert(IntegralConstant<char, 'a'>::value == 'a');
static_assert(IntegralConstant<bool, true>::value == true);
```

Dla takich meta-funkcji często definiujemy szablony zmiennych, które pozwalają uniknąć konieczności deklaracji `::value`:

```cpp
template <typename T, T Constant>
inline constexpr T IntegralConstant_v = IntegralConstant<T, Constant>::value;

static_assert(IntegralConstant_v<int, 5> == 5);
static_assert(IntegralConstant_v<char, 'a'> == 'a');
static_assert(IntegralConstant_v<bool, true> == true);
````

### Meta-funkcje zwracające typy

Najprostsz przykład meta-funkcji zwracającej typ to `Identity`:

```cpp
template <typename T>
struct Identity
{
    using type = T;
};

static_assert(std::is_same<Identity<int>::type, int>::value);
static_assert(std::is_same<Identity<std::string>::type, std::string>::value);
```

Dla takich meta-funkcji często definiujemy aliasy szablonów, które pozwalają uniknąć konieczności deklaracji `typename`
przed zagnieżdżonym typem `type`:

```cpp
template <typename T>
using Identity_t = typename Identity<T>::type;

static_assert(std::is_same<Identity_t<int>, int>::value);
static_assert(std::is_same<Identity_t<std::string>, std::string>::value);
```

## Klasy cech typów - predykaty

Predykaty to meta-funkcje, które zwracają wartość `true` lub `false` w zależności od cechowanego typu.

### Klasa cech IsVoid<T>

Najprostszym przykładem predykatu jest `IsVoid`, który sprawdza, czy dany typ to `void`. Implementacja predykatu polega na zdefiniowaniu ogólnego
szablonu, który definiuje zagnieżdżoną stałą statyczną `value` o wartości `false`, oraz pełnej specjalizacji szablonu dla typu `void`, która definiuje `value` jako `true`.

```cpp
template <typename T>
struct IsVoid
{
    static constexpr bool value = false;
};

template <>
struct IsVoid<void>
{
    static constexpr bool value = true;
};

template <typename T>
inline constexpr bool IsVoid_v = IsVoid<T>::value;

static_assert(!IsVoid_v<int>); // int is not void - predicate is false
static_assert(IsVoid_v<void>); // void is void - predicate is true
```

### Klasa cech IsSame<T, U>

Predykat `IsSame` sprawdza, czy dwa typy są takie same. Implementacja predykatu polega na zdefiniowaniu ogólnego
szablonu, który definiuje zagnieżdżoną stałą statyczną `value` o wartości `false`, oraz specjalizacji częściowej szablonu dla dwóch takich samych typ
ów, która definiuje `value` jako `true`.

```cpp
template <typename T, typename U>
struct IsSame
{
    static constexpr bool value = false;
};

template <typename T>
struct IsSame<T, T>
{
    static constexpr bool value = true;
};

template <typename T, typename U>
inline constexpr bool IsSame_v = IsSame<T, U>::value;

static_assert(IsSame_v<int, int>);           // true
static_assert(!IsSame_v<int, unsigned>);     // false
static_assert(IsSame_v<std::string, std::string>); // true
static_assert(!IsSame_v<std::string, const char*>); // false
```

```{note}
Predykatową klasę cech można zaimplementować dziedzicząc po `std::true_type` i `std::false_type`, które są zdefiniowane w bibliotece standardowej w nagłówku `<type_traits>`.

````cpp
template <typename T, typename U>
struct IsSame : std::false_type
{};

template <typename T>
struct IsSame<T, T> : std::true_type
{};
````
```

### Klasy cech transformujące typy

Często w trakcie pisania kodu szablonowego zachodzi potrzeba transformacji typu określonego
parametru szablonu (np. wymagane jest usunięcie lub dodanie referencji, modyfikatora `const` lub `volatile`, itp.).

W takim przypadku możemy posłużyć się klasą cechy transformującej (implementowaną jako meta-funkcja):

```cpp
template <typename T>
struct RemoveReference
{
    using type = T;
};

template <typename T>
struct RemoveReference<T&>
{
    using type = T;
};

template <typename T>
struct RemoveReference<T&&>
{
    using type = T;
};
```

Od C++14 do klas cech dodane są odpowiednie aliasy szablonów, które umożliwiają uniknięcie konieczności deklaracji `typename`
przed zagnieżdżonym typem `type`:

```cpp
template <typename T>
using RemoveReference_t = typename RemoveReference<T>::type;
```

Klasa cechy `RemoveReference` może być wykorzystana w innych szablonach, np. do implementacji funkcji `move`:

```cpp
template <typename T>
RemoveReference_t<T>&& move(T&& item)
{
    return static_cast<RemoveReference_t<T>&&>(item);
}
```

## Biblioteka standardowa <type_traits>

### Cechy transformujące typy w bibliotece standardowej

Biblioteka standardowa w nagłówku `<type_traits>` definiuje zbiór klas cech transformujących:

| Cecha | Rezultat `::value` |
|-------|-------------------|
| `remove_reference` | usuwa referencję z typu (`int& -> int`) |
| `add_lvalue_reference` | dodaje lvalue referencję  (`double -> double&`) |
| `add_rvalue_reference` | dodaje rvalue referencję  (`double -> double&&`) |
| `remove_pointer` | usuwa wskaźnik z typu (`int* -> int`) |
| `add_pointer` | dodaje wskaźnik (`int -> int*`) |
| `remove_const` | usuwa modyfikator `const`  (`const int& -> int&`) |
| `remove_volatile` | usuwa modyfikator `volatile` (`volatile int -> int`) |
| `remove_cv` | usuwa modyfikatory `const` i `volatile` |
| `add_const` | dodaje modyfikator `const`  (`int -> const int`) |
| `add_volatile` | dodaje modyfikator `volatile`  (`double -> volatile double`) |

#### Cecha `std::decay`

Przydatną cechą jest zdefiniowana w bibliotece standardowej cecha `std::decay`.

Dokonuje ona transformacji odpowiadającej następującym przekształceniom:

* usuwane są referencje
* usuwane są modyfikatory `const` lub `volatile`
* tablice konwertowane są do wskaźników
* funkcje konwertowane są do wskaźników do funkcji

```cpp
template <typename T, typename U>
void check_decay()
{
    static_assert(std::is_same_v<std::decay_t<T>, U>);
}

check_decay<int&, int>();
check_decay<const int&, int>();
check_decay<int&&, int>();
check_decay<int(int), int(*)(int)>();
check_decay<int[20], int*>();
check_decay<const int[20], const int*>();
```

### Standardowe cechy typów - predykaty

Biblioteka standardowa definiuje szeroki zbiór meta-funkcji, które
umożliwiają odpytanie na etapie kompilacji, czy dany typ posiada
odpowiednie cechy.

#### Cechy podstawowe

| Cecha podstawowa | Rezultat `::value` |
|-----------------|-------------------|
| `is_array<T>` | `true` jeśli `T` jest typem tablicowym |
| `is_class<T>` | `true` jeśli `T` jest klasą |
| `is_enum<T>` | `true` jeśli `T` jest typem wyliczeniowym |
| `is_floating_point<T>` | `true` jeśli `T` jest typem zmiennoprzecinkowym |
| `is_function<T>` | `true` jeśli `T` jest funkcją |
| `is_integral<T>` | `true` jeśli `T` jest typem całkowitym |
| `is_member_object_pointer<T>` | `true` jeśli `T` jest wskaźnikiem do składowej |
| `is_member_function_pointer<T>` | `true` jeśli `T` jest wskaźnikiem do funkcji  składowej |
| `is_pointer<T>` | `true` jeśli `T` jest typem wskaźnikowym (ale nie wskaźnikiem do składowej) |
| `is_lvalue_reference<T>` | `true` jeśli `T` jest referencją do l-value |
| `is_rvalue_reference<T>` | `true` jeśli `T` jest referencją do r-value |
| `is_union<T>` | `true` jeśli `T` jest unią (bez wsparcia kompilatora zawsze zwraca `false` |
| `is_void<T>` | `true` jeśli `T` jest typu void |
| `is_null_pointer<T>` | `true` jeśli `T` jest typu `std::nullptr_t` |

#### Cechy kompozytowe

Cechy kompozytowe są kompozycją najczęściej kilku cech podstawowych.

| Cecha grupowana | Rezultat `::value` |
|----------------|-------------------|
| `is_arithmetic<T>` | `is_integral<T>::value \|\|` `is_floating_point<T>::value` |
| `is_fundamental<T>` | `is_arithmetic<T>::value \|\| is_void<T>::value \|\| is_null_pointer<T>::value` |
| `is_compound<T>` | `!is_fundamental<T>::value` |
| `is_object<T>` | `is_scalar<T>::value \|\| is_array<T>::value  \|\| is_union<T>::value  \|\|` `is_class<T>::value` |
| `is_reference<T>` | `is_lvalue_reference<T> \|\| is_rvalue_reference<T>` |
| `is_member_pointer<T>` | `is_member_object_pointer<T> \|\| is_member_function_pointer<T>` |
| `is_scalar<T>` | `is_arithmetic<T>::value \|\| is_enum<T>::value \|\| is_null_pointer<T>::value` `is_pointer<T>::value \|\| is_member_pointer<T>::value` |

#### Właściwości typów

Standard używa terminu **właściwość typu** w celu zdefiniowania cechy opisującej wybrane atrybuty typu.

Wybrane właściwości typu:

| Właściwość typu | Rezultat `::value` |
|----------------|-------------------|
| `is_const<T>` | `true` jeśli `T` jest typem `const` |
| `is_volatile<T>` | `true` jeśli `T` jest typem ulotnym |
| `is_polymorphic<T>` | `true` jeśli `T` posiada przynajmniej jedną funkcję wirtualną |
| `is_trivial<T>` | `true` jeśli `T` jest typem trywialnym |
| `is_trivially_copyable<T>` | `true` jeśli `T` jest trywialnie kopiowalne |
| `is_standard_layout<T>` | `true` jeśli `T` jest typem o standardowym layout'cie |
| `is_pod<T>` | `true` jeśli `T` jest typem POD |
| `is_abstract<T>` | `true` jeśli `T` jest typem abstrakcyjnym |
| `is_unsigned<T>` | `true` jeśli `T` jest typem całkowitym bez znaku lub typem wyliczeniowym zdefiniowanym przy pomocy typu `unsigned` |
| `is_signed<T>` | `true` jeśli `T` jest typem całkowitym ze znakiem lub typem wyliczeniowym zdefiniowanym przy pomocy typu `signed` |

### Cechy typów i statyczne asercje

Jednym z podstawowych zastosowań cech typów jest wykorzystanie ich do
statycznych asercji w kodzie. Takie asercje nie pozwalają wygenerować
błędnego kodu i jednocześnie podnoszą czytelność komunikatów o błędach w
szablonach.

Przykład:

```cpp
template <class T>
void swap(T& a, T& b)
{
    static_assert(std::is_copy_constructible<T>::value,
                 "Swap requires copying");

    static_assert(noexcept(std::is_nothrow_move_constructible<T>::value
                        && std::is_nothrow_move_assignable<T>::value),
                  "Swap may throw");

    auto c = b;
    b = a;
    a = c;
}
```

## Przykłady użycia cech typów

Załóżmy, że chcemy napisać szablon funkcji `process`, który będzie przetwarzał elementy kontenera. Aby to zrobić, musimy znać typ elementów przechowywanych w kontenerze. Możemy to osiągnąć za pomocą klasy cechy `ValueType`, która zwraca typ elementów kontenera.

```cpp
// generic implementation for containers
template<typename Container>
struct ValueType 
{
    using type = typename Container::value_type;
};

//partial specialization for arrays of known bounds
template<typename T, std::size_t N>
struct ValueType<T[N]> 
{          
    using type = T;     
};

//partial specialization for arrays of unknown bounds
template<typename T> 
struct ValueType<T[]> 
{          
    using type = T;
};

template <typename Container>
using ValueType_t = typename ValueType<Container>::type;
```

Meta-funkcje mogą być wykorzystane w innych szablonach:

```cpp
template <typename Container>
void process(const Container& c)
{
    ValueType_t<Container> item{};

    //...
}
```
