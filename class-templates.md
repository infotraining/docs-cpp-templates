# Szablony klas

Podobnie do funkcji, klasy oraz struktury też mogą być być szablonami sparametryzowanymi typami.

Szablony klas mogą być wykorzystane do implementacji kontenerów, które mogą przechowywać dane typów, które będą definiowane później.

W terminologii obiektowej szablony klas nazywane są *klasami parametryzowanymi*.

W przypadku użycia szablonów klas generowany jest kod tylko dla tych funkcji składowych, które rzeczywiście są wywoływane.

```cpp
template <typename T>
class Vector 
{
    size_t size_; 
    T* items_;
    
public: 
    explicit Vector(size_t size);
    ~Vector() { delete [] items_; }
    
    //...
    
    const T& operator[](size_t index) const
    {
         return items_[index];
    }

    T& operator[](size_t index)
    {
         return items_[index];
    }

    const T& at(size_t index) const;
    T& at(size_t index);
    
    size_t size() const
    {
        return size_;
    }
};
```

Aby utworzyć zmienne typów szablonowych musimy określić parametry szablonu klasy:

```cpp
Vector<int> integral_numbers(100);
Vector<double> real_numbers(200);
Vector<std::string> words(665);
Vector<Vector<int>> matrix(10);
```

## Implementacja funkcji składowych

Definiując funkcję składową szablonu klasy należy określić jej przynależność do szablonu.

```cpp
template <typename T>
Vector<T>::Vector(size_t size) 
    : size_{size}, items_{new T[size]}
{        
}

template <typename T>
T& Vector<T>::at(size_t index)
{        
    if (index >= size_)
        throw std::out_of_range("Vector::operator[]");

    return items_[index];
}

template <typename T>
const T& Vector<T>::at(size_t index) const
{        
    if (index >= size_)
        throw std::out_of_range("Vector::operator[]");

    return items_[index];
}
```

## Parametry szablonów klas

Każdy parametr szablonu może być:

1. Typem (wbudowanym lub zdefiniowanym przez użytkownika).
2. Stałą znaną w chwili kompilacji (liczby całkowite, wskaźniki i referencje danych statycznych).
3. Innym szablonem.

### Parametry szablonów niebędące typami

Można używać parametrów nie będących typami, o ile są to wartości znane na etapie kompilacji.

```cpp
template <class T, size_t N>
struct Array 
{
    using value_type = T;        

    T items_[N];
    
    constexpr size_t size()
    {
        return N;
    }
}; 

Array<int, 1024> buffer;
```

Parametrom szablonu klasy można przypisać argumenty domyślne (od C++11 jest to możliwe również dla szablonów funkcji).

```cpp
template <class T, size_t N = 1024>
struct Array 
{
public:
    using value_type = T;        

    constexpr size_t size()
    {
        return N;
    }

    T items_[N];
}; 

template <typename T, typename Container = std::vector<T>>
class Stack
{
public:
    Stack();
    void push(const T& elem);
    void push(T&& elem);
    void pop();
    T& top() const;
private:
    Container items_;
};

// creating objects
Stack<int> stack_one; // Stack<int, vector<int>>
Stack<int, std::list<int>> stack_two;
```

### Szablony jako parametry szablonów

Jeżeli w kodzie jako parametr ma być użyty inny szablon, to kompilator powinien zostać o tym poinformowany.

```cpp
template <typename T, template<typename, size_t> class Container, size_t N = 1024>
class Stack 
{
private:
    Container<T, N> elems_;
public:
    Stack();
    void push(const T& elem);
    void push(T&& elem);
    void pop();
    T& top() const;
};

template <typename T, template<typename, size_t> class Container, size_t N>
Stack<T, Container, N>::Stack()
{
}

// creating objects
Stack<int, Array> stack_three;
Stack<int, Array, 512> stack_four;
```

## Specjalizacja szablonów klas

Specjalizacja szablonów klas:

* polega na osobnej implementacji szablonów dla wybranych typów argumentów.
* umożliwia optymalizację implementacji dla wybranych typów lub uniknięcie niepożądanego zachowania na skutek utworzenia instancji szablonu dla określonego typu.

```cpp
template <typename T> class Pair { /* ... */ };     // szablon ogólny
template<typename T> class Pair<T*> { /* ... */ };  // częściowa specjalizacja
template<> class Pair<const char*>{ /* ... */ };    // pełna specjalizacja
```

### Pełna specjalizacja szablonu klasy

Deklaracja pełnej specjalizacji wymaga podania:

```cpp
template <>
class Vector<bool>
{
    //...
};
```

Jeśli specjalizujemy szablon klasy, to musimy zapewnić wyspecjalizowaną implementację dla wszystkich funkcji składowych.

Możliwe jest natomiast rozszerzenie interfejsu klasy o dodatkowe składowe (np. `std::vector<bool>` definiuje dodatkowe metody `flip()`)

### Specjalizacja częściowa szablonu klasy

Dla szablonów klas (w odróżnieniu od szablonów funkcji) możliwe jest tworzenie częściowych specjalizacji szablonów.

Dla szablonu klasy:

```cpp
template <class T1, class T2>
class MyClass 
{
  //...
};
```

możemy utworzyć następujące specjalizacje częściowe:

```cpp
template <typename T>
class MyClass<T, T> 
{ 
  // specjalizacja częściowa: drugim typem jest T
};
```

```cpp
template <typename T>
class MyClass<T, int> 
{
    // specjalizacja częściowa: drugim typem jest int 
};
```

```cpp
template <typename T1, typename T2>
class MyClass<T1*, T2*> 
{
    // oba parametry są wskaźnikami 
};
```

Poniższe przykłady pokazują, które wersje szablonu klasy zostaną utworzone:

```cpp
MyClass<int, float> mif;      // uses MyClass<T1,T2>
MyClass<float, float> mff;    // uses MyClass<T,T>
MyClass<float, int> mfi;      // uses MyClass<T,int>
MyClass<int*, float*> mp;     // uses MyClass<T1*,T2*>
```

W przypadku, gdy więcej niż jedna specjalizacja pasuje wystarczająco dobrze zgłaszany jest błąd dwuznaczności:

```cpp
MyClass<int, int> me1; // ERROR: matches MyClass<T, T> & MyClass<T, int>

MyClass<int*, int*> me2; // ERROR: matches MyClass<T, T> & MyClass<T1*, T1*>
```

## Składowe jako szablony

Składowe klas mogą być szablonami. Dotyczy to:

* wewnętrznych klas pomocniczych,
* funkcji składowych.

```cpp
template <typename T>
class Stack 
{
    std::deque<T> items_;
public:
    void push(const T&);
    void pop();
    //...
    // przypisanie stosu o elementach typu T2
    template <typename T2>
    Stack<T>& operator=(const Stack<T2>&);
}; 

template <typename T>
    template <typename T2>
Stack<T>& Stack<T>::operator=(const Stack<T2>& source)
{
    //...
}
```

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
```

## Dedukcja argumentów szablonu dla klas

C++17 wprowadza mechanizm dedukcji argumentów szablonu klasy (*Class Template Argument Deduction*).
Typy parametrów szablonu klasy mogą być dedukowane na podstawie argumentów przekazanych do konstruktora tworzonego obiektu.

```cpp
template <typename T>
class complex
{
    T re_, img_;
public:
    complex(T re, T img) : re_{re}, img_{img}
    {}
};

auto c1 = complex<int>{5, 2}; // OK - all versions of C++ standard

auto c2 = complex{5, 3}; // OK since C++17 - compiler deduces complex<int>

auto c3 = complex(5.1, 6.5); // OK in C++17 - compiler deduces complex<double>

auto c4 = complex{5, 4.1}; // ERROR - args don't have the same type
auto c5 = complex(5, 4.1); // ERROR - args don't have the same type
```

```{warning}
Nie można częściowo dedukować argumentów szablonu klasy. Należy wyspecifikować lub wydedukować wszystkie parametry z wyjątkiem parametrów domyślnych.
```

Praktyczny przykład dedukcji argumentów szablonu klasy:

```cpp
std::mutex mtx;

std::lock_guard lk{mtx}; // deduces std::lock_guard<std::mutex>
```

### Podpowiedzi dedukcyjne (*deduction guides*)

C++17 umożliwia tworzenie podpowiedzi dla kompilatora, jak powinny być dedukowane typy parametrów szablonu klasy na podstawie wywołania odpowiedniego konstruktora.

Daje to możliwość poprawy/modyfikacji domyślnego procesu dedukcji.

Dla szablonu:

```cpp
template <typename T>
class S
{
private:
    T value;
public:
    S(T v) : value(v)
    {}
};
```

Podpowiedź dedukcyjna musi zostać umieszczona w tym samym zakresie (przestrzeni nazw) i może mieć postać:

```cpp
template <typename T> S(T) -> S<T>; // deduction guide
```

gdzie:

* `S<T>` to tzw. typ zalecany (*guided type*)
* nazwa podpowiedzi dedukcyjnej musi być niekwalifikowaną nazwą klasy szablonowej zadeklarowanej wcześniej w tym samym zakresie
* typ zalecany podpowiedzi musi odwoływać się do identyfikatora szablonu (*template-id*), do którego odnosi się podpowiedź

Użycie podpowiedzi:

```cpp
S x{12}; // OK -> S<int> x{12};
S y(12); // OK -> S<int> y(12);
auto z = S{12}; // OK -> auto z = S<int>{12};
S s1(1), s2{2}; // OK -> S<int> s1(1), s2{2};
S s3(42), s4{3.14}; // ERROR
```

W deklaracji `S x{12};` specyfikator `S` jest nazywany symbolem zastępczym dla klasy (*placeholder class type*).

W przypadku użycia symbolu zastępczego dla klasy, nazwa zmiennej musi zostać podana jako następny element składni.
W rezultacie poniższa deklaracja jest błędem składniowym:

```cpp
S* p = &x; // ERROR - syntax not permitted
```

Dany szablon klasy może mieć wiele konstruktorów oraz wiele podpowiedzi dedukcyjnych:

```cpp
template <typename T>
struct Data
{
    T value;

    using type1 = T;

    Data(const T& v)
        : value(v)
    {
    }

    template <typename ItemType>
    Data(initializer_list<ItemType> il)
        : value(il)
    {
    }
};

template <typename T>
Data(T)->Data<T>;

template <typename T>
Data(initializer_list<T>)->Data<vector<T>>;

Data(const char*) -> Data<std::string>;

//...

Data d1("hello"); // OK -> Data<string>

const int tab[10] = {1, 2, 3, 4};
Data d2(tab); // OK -> Data<const int*>

Data d3 = 3; // OK -> Data<int>

Data d4{1, 2, 3, 4}; // OK -> Data<vector<int>>

Data d5 = {1, 2, 3, 4}; // OK -> Data<vector<int>>

Data d6 = {1}; // OK -> Data<vector<int>>

Data d7(d6); // OK - copy by default rule -> Data<vector<int>>

Data d8{d6, d7}; // OK -> Data<vector<Data<vector<int>>>>
```

Podpowiedzi dedukcyjne nie są szablonami funkcji - służą jedynie dedukowaniu argumentów szablonu i nie są wywoływane. 
W rezultacie nie ma znaczenia czy argumenty w deklaracjach dedukcyjnych są przekazywane przez referencje, czy nie.

```cpp
template <typename T> 
struct X
{
    //...
};

template <typename T>
struct Y
{
    Y(const X<T>&);
    Y(X<T>&&);
};

template <typename T> Y(X<T>) -> Y<T>; // deduction guide without references
```

W powyższym przykładzie podpowiedź dedukcyjna nie odpowiada dokładnie sygnaturom konstruktorów przeciążonych. Nie ma to znaczenia, ponieważ jedynym celem podpowiedzi jest umożliwienie dedukcji typu, który jest parametrem szablonu. Dopasowanie wywołania przeciążonego konstruktora odbywa się później.

### Niejawne podpowiedzi dedukcyjne

Ponieważ często podpowiedź dedukcyjna jest potrzebna dla każdego konstruktora klasy, standard C++17
wprowadza mechanizm **niejawnych podpowiedzi dedukcyjnych** (*implicit deduction guides*).
Działa on w następujący sposób:

* Lista parametrów szablonu dla podpowiedzi zawiera listę parametrów z szablonu klasy
  - w przypadku szablonowego konstruktora klasy kolejnym elementem jest lista parametrów szablonu konstruktora klasy

* Parametry "funkcyjne" podpowiedzi są kopiowane z konstruktora lub konstruktora szablonowego

* Zalecany typ w podpowiedzi jest nazwą szablonu z argumentami, które są parametrami szablonu wziętymi 
  z klasy szablonowej

Dla klasy szablonowej rozważanej powyżej:

```cpp
template <typename T>
class S
{
private:
    T value;
public:
    S(T v) : value(v)
    {}
};
```

niejawna podpowiedź dedukcyjna będzie wyglądać następująco:

```cpp
template <typename T> S(T) -> S<T>; // implicit deduction guide
```

W rezultacie programista nie musi implementować jej jawnie.

### Specjalny przypadek dedukcji argumentów klasy szablonowej

Rozważmy następujący przypadek dedukcji:

```cpp
S x{42}; // x has type S<int>

S y{x};
S z(x);
```

W obu przypadkach dedukowany typ zmiennych `y` i `z` to `S<int>`. Mechanizm dedukcji argumentów klasy szablonowej
dedukuje typ taki sam jak typ oryginalnego obiektu a następnie wywoływany jest konstruktor kopiujący.

* dla deklaracji `S<T> x;`
  `S{x}` dedukuje typ: `S<T>{x}` zamiast `S<S<T>>{x}`

W niektórych przypadkach może być to zaskakujące i kontrowersyjne:

```cpp
std::vector v{1, 2, 3}; // vector<int>
std::vector data1{v, v}; // vector<vector<int>>
std::vector data2{v}; // vector<int>!
```

W powyższym kodzie dedukcja argumentów szablonu `vector` zależy od ilości argumentów przekazanych do konstruktora!

### Agregaty a dedukcja argumentów

Jeśli szablon klasy jest agregatem, to mechanizm automatycznej dedukcji argumentów szablonu wymaga napisania jawnej podpowiedzi dedukcyjnej.

Bez podpowiedzi dedukcyjnej dedukcja dla agregatów nie działa:

```cpp
template <typename T>
struct Aggregate1
{
    T value;
};

Aggregate1 agg1{8}; // ERROR
Aggregate1 agg2{"eight"}; // ERROR
Aggregate1 agg3 = 3.14; // ERROR
```

Gdy napiszemy dla agregatu podpowiedź, to możemy zacząć korzystać z mechanizmu dedukcji:

```cpp
template <typename T>
struct Aggregate2
{
    T value;
};

template <typename T>
Aggregate2(T) -> Aggregate2<T>;

Aggregate2 agg1{8}; // OK -> Aggregate2<int>
Aggregate2 agg2{"eight"}; // OK -> Aggregate2<const char*>
Aggregate2 agg3 = { 3.14 }; // OK -> Aggregate2<double>
```

### Podpowiedzi dedukcyjne w bibliotece standardowej

Dla wielu klas szablonowych z biblioteki standardowej dodano podpowiedzi dedukcyjne w celu ułatwienia tworzenia instancji tych klas.

#### std::pair<T>

Dla pary STL dodana w standardzie podpowiedź to:

```cpp
template<class T1, class T2>
pair(T1, T2) -> pair<T1, T2>;

pair p1(1, 3.14); // -> pair<int, double>

pair p2{3.14f, "text"s}; // -> pair<float, string>

pair p3{3.14f, "text"}; // -> pair<float, const char*>

int tab[3] = { 1, 2, 3 };
pair p4{1, tab}; // -> pair<int, int*>
```

#### std::tuple<T...>

Szablon `std::tuple` jest traktowany podobnie jak `std::pair`:

```cpp
template<class... UTypes>
tuple(UTypes...) -> tuple<UTypes...>;

template<class T1, class T2>
tuple(pair<T1, T2>) -> tuple<T1, T2>;

//... other deduction guides working with allocators

int x = 10;
const int& cref_x = x;

tuple t1{x, &x, cref_x, "hello", "world"s}; -> tuple<int, int*, int, const char*, string>
```

#### std::optional<T>

Klasa `std::optional` jest traktowana podobnie do pary i krotki.

```cpp
template<class T> optional(T) -> optional<T>;

optional o1(3); // -> optional<int>
optional o2 = o1; // -> optional<int>
```

#### Inteligentne wskaźniki

Dedukcja dla argumentów konstruktora będących wskaźnikami jest zablokowana:

```cpp
int* ptr = new int{5};
unique_ptr uptr{ip}; // ERROR - ill-formed (due to array type clash)
```

Wspierana jest dedukcja przy konwersjach:

* z `weak_ptr`/`unique_ptr` do `shared_ptr`:

  ```cpp
  template <class T> shared_ptr(weak_ptr<T>) ->  shared_ptr<T>;
  template <class T, class D> shared_ptr(unique_ptr<T, D>) ->  shared_ptr<T>;
  ```

* z `shared_ptr` do `weak_ptr`

  ```cpp
  template<class T> weak_ptr(shared_ptr<T>) -> weak_ptr<T>;
  ```

```cpp
unique_ptr<int> uptr = make_unique<int>(3);

shared_ptr sptr = move(uptr); -> shared_ptr<int>
    
weak_ptr wptr = sptr; // -> weak_prt<int>

shared_ptr sptr2{wptr}; // -> shared_ptr<int>
```

#### std::function

Dozwolone jest dedukowanie sygnatur funkcji dla `std::function`:

```cpp
int add(int x, int y)
{
    return x + y;
}

function f1 = &add;
assert(f1(4, 5) == 9);

function f2 = [](const string& txt) { cout << txt << " from lambda!" << endl; };
f2("Hello");
```

#### Kontenery i sekwencje

Dla kontenerów standardowych dozwolona jest dedukcja typu kontenera dla konstruktora akceptującego parę iteratorów:

```cpp
vector<int> vec{ 1, 2, 3 };
list lst(vec.begin(), vec.end()); // -> list<int>
```

Dla `std::array` dozwolona jest dedukcja z sekwencji:

```cpp
std::array arr1 { 1, 2, 3 }; // -> std::array<int, 3>
```
