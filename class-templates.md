# Szablony klas

**Szablony klas** (*class templates*) to mechanizm pozwalający na tworzenie klas, które są parametryzowane. Podobnie jak szablony funkcji, szablony klas są fundamentalnym elementem programowania generycznego w C++.

Szablony klas charakteryzują się następującymi właściwościami:

* **Parametryzacja typu** - klasa może działać z różnymi typami danych określonymi w momencie utworzenia obiektu
* **Generowanie kodu na żądanie** - kompilator tworzy kod tylko dla tych funkcji składowych, które rzeczywiście są wywoływane (*lazy instantiation*)
* **Bezpieczeństwo typów** - sprawdzanie typów odbywa się w czasie kompilacji
* **Brak narzutu wydajnościowego** - kod jest generowany w czasie kompilacji, więc nie ma narzutu w czasie działania programu

Szablony klas są szeroko wykorzystywane do:

* **Implementacji kontenerów** - `std::vector`, `std::list`, `std::map` itp.
* **Tworzenia smart pointerów** - `std::unique_ptr`, `std::shared_ptr`
* **Wrapper'ów i adapterów** - `std::optional`, `std::variant`, `std::any`
* **Metaprogramowania** - obliczenia w czasie kompilacji
* **Policy-based design** - elastyczne konfigurowanie zachowania klas

## Podstawowa składnia

### Definicja szablonu klasy

Szablon klasy definiujemy poprzedzając definicję klasy deklaracją `template` z listą parametrów:

```cpp
template <typename T>
class Vector 
{
    size_t size_; 
    T* items_;
    
public: 
    explicit Vector(size_t size);
    ~Vector() noexcept { delete [] items_; }
    
    // Operacje kopiowania
    Vector(const Vector& other);
    Vector& operator=(const Vector& other);
    
    // Dostęp do elementów
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
    
    size_t size() const noexcept
    {
        return size_;
    }

    bool empty() const noexcept
    {
        return size_ == 0;
    }

    void swap(Vector& other) noexcept
    {
        std::swap(size_, other.size_);
        std::swap(items_, other.items_);
    }
    
    // Iteratory (uproszczone)
    T* begin() noexcept { return items_; }
    T* end() noexcept { return items_ + size_; }
    const T* begin() const noexcept { return items_; }
    const T* end() const noexcept { return items_ + size_; }
};
```

### Implementacja funkcji składowych poza ciałem klasy

Definiując funkcję składową szablonu klasy **poza ciałem klasy**, należy:

1. Poprzedzić definicję deklaracją `template` z parametrami
2. Użyć pełnej nazwy klasy z parametrami szablonu (`Vector<T>`)
3. Zachować spójność parametrów szablonu

```cpp
// Konstruktor
template <typename T>
Vector<T>::Vector(size_t size) 
    : size_{size}, items_{new T[size]}
{
    std::fill_n(items_, size_, T{});
}

// Konstruktor kopiujący
template <typename T>
Vector<T>::Vector(const Vector& other)
    : size_{other.size_}, items_{new T[size_]}
{
    for (size_t i = 0; i < size_; ++i)
        items_[i] = other.items_[i];
}

// Operator przypisania
template <typename T>
Vector<T>& Vector<T>::operator=(const Vector& other)
{
    if (this != &other) {
        Vector<T> temp(other); // copy-and-swap idiom
        swap(temp);
    }
    return *this;
}

// Metoda at() z sprawdzaniem zakresu
template <typename T>
T& Vector<T>::at(size_t index)
{        
    if (index >= size_)
        throw std::out_of_range("Vector::at() - index out of range");

    return items_[index];
}

template <typename T>
const T& Vector<T>::at(size_t index) const
{        
    if (index >= size_)
        throw std::out_of_range("Vector::at() - index out of range");

    return items_[index];
}
```

### Tworzenie instancji szablonu klasy

Aby utworzyć obiekt na podstawie szablonu klasy, musimy **jawnie określić parametry szablonu** w nawiasach ostrych `<>`:

```cpp
Vector<int> integral_numbers(10); // Vector przechowujący wartości typu int
Vector<double> real_numbers(20);  // Vector przechowujący wartości typu double
Vector<std::string> words(42);    // Vector przechowujący wartości typu string
```

```{note}
W przeciwieństwie do szablonów funkcji, dla szablonów klas (przed C++17) **nie ma automatycznej dedukcji typów**. Musimy zawsze podać jawnie typ, który jest parametrem szablonu.

Od C++17 nie jest to wymagane dla szablonów klas, które wspierają mechanizm CTAD - *Class Template Argument Deduction*.
```

### Przykład użycia

```cpp
// Tworzenie wektora liczb całkowitych
Vector<int> numbers(5);
for (size_t i = 0; i < numbers.size(); ++i)
    numbers[i] = i * 10;

// Iteracja używając range-based for (dzięki begin/end)
for (auto num : numbers)
    std::cout << num << " ";  // Wypisze: 0 10 20 30 40

std::cout << "\n";

// Tworzenie wektora stringów
Vector<std::string> words(3);
words[0] = "Hello";
words[1] = "World";
words[2] = "!";

for (const auto& word : words)
    std::cout << word << " ";  // Wypisze: Hello World !
std::cout << "\n";
```

## Parametry szablonów klas

Szablony klas mogą mieć różne rodzaje parametrów, co daje dużą elastyczność w projektowaniu.

Każdy parametr szablonu może być:

1. **Parametrem typu** (`typename` lub `class`) - dowolny typ
2. **Parametrem niebędącym typem** (*non-type parameter - NTTP*) - stała wartość znana w czasie kompilacji
3. **Parametrem będącym szablonem** (*template template parameter*) - szablon jako parametr

### 1. Parametry typu

Najpowszechniejszy rodzaj parametrów:

```cpp
template <typename T>
class Box
{
    T value_;
public:
    explicit Box(T v) : value_(v) {}
    const T& get() const { return value_; }
};

// Użycie
Box<int> int_box(42);
Box<std::string> str_box("Hello");
```

Możemy mieć wiele parametrów, które są typami:

```cpp
template <typename Key, typename Value>
class KeyValuePair
{
    Key key_;
    Value value_;
public:
    KeyValuePair(Key k, Value v) : key_(k), value_(v) {}
    Key key() const { return key_; }
    Value value() const { return value_; }
};

// Użycie
KeyValuePair<std::string, int> pair("age", 25);
```

### 2. Parametry niebędące typami - NTTP

Można używać **parametrów nie będących typami**, o ile są to wartości znane na etapie kompilacji:

```cpp
template <typename T, size_t N>
struct Array 
{
    using value_type = T;
    using iterator = T*;
    using const_iterator = const T*;

    T items_[N];
    
    constexpr size_t size() const
    {
        return N;
    }
    
    T& operator[](size_t index)
    {
        return items_[index];
    }
    
    const T& operator[](size_t index) const
    {
        return items_[index];
    }

    iterator begin() noexcept { return items_; }
    iterator end() noexcept { return items_ + N; }
    const_iterator begin() const noexcept { return items_; }
    const_iterator end() const noexcept { return items_ + N; }
}; 

// Użycie
Array<int, 10> small_buffer;
Array<double, 1024> large_buffer;
Array<char, 256> char_buffer;

// Rozmiar jest częścią typu!
static_assert(sizeof(Array<int, 10>) != sizeof(Array<int, 20>));

// Nie można przypisać tablic różnych rozmiarów
// Array<int, 10> a;
// Array<int, 20> b;
// a = b;  // BŁĄD - różne typy!
```

#### Dopuszczalne parametry NTTP

* Typy całkowite (`int`, `long`, `size_t`, etc.)
* Typy wyliczeniowe
* Wskaźniki i referencje do obiektów/funkcji (C++11)
* `std::nullptr_t` (C++11)
* Wskaźniki do składowych (C++11)
* Typy zmiennoprzecinkowe (C++20)
* Typy strukturalne (C++20)
* Lambdy (C++20)


### 3. Szablony jako parametry szablonów

**Template template parameters** - gdy jako parametr ma być użyty inny szablon, kompilator musi zostać o tym poinformowany. Należy przy tym określić liczbę i rodzaj parametrów szablonu przekazywanego jako argument:

```cpp
template <typename T, 
          template<typename, typename> class Container, /* template template parameter */
          typename TAllocator = std::allocator<T>>
class Stack 
{
private:
    Container<T, TAllocator> items_;
    
public:
    Stack() = default;

    void push(const T& item)
    {
        items_.push_back(item);
    }

    void push(T&& item)
    {
        items_.push_back(std::move(item));
    }
    
    void pop()
    {
        if (items_.empty())
            throw std::underflow_error("Stack is empty");
        
        items_.pop_back();
    }
    
    T& top()
    {
        if (items_.empty())
            throw std::underflow_error("Stack is empty");

        return items_.back();
    }
    
    const T& top() const
    {
        if (items_.empty())
            throw std::underflow_error("Stack is empty");
        
        return items_.back();
    }
    
    bool empty() const { return items_.empty(); }

    size_t size() const { return items_.size(); }
    
private:
    Container items_;
};

// Użycie - podajemy szablon kontenera, nie konkretny typ
Stack<int, std::vector> vec_stack;
Stack<int, std::deque> deque_stack;
Stack<int, std::list> list_stack;
```

### Parametry domyślne

Parametrom szablonu klasy można przypisać **argumenty domyślne** (od C++11 możliwe również dla szablonów funkcji):

```cpp
// Domyślny rozmiar tablicy
template <typename T, size_t N = 1024>
struct Array 
{
    //...
}; 

// Użycie z domyślnym rozmiarem
Array<int> default_buffer;      // Array<int, 1024>
Array<int, 512> small_buffer;   // Array<int, 512>
Array<double> double_buffer;    // Array<double, 1024>
```

Często jako parametr szablonu klasy podaje się kontener (np. `std::vector`, `std::list`, etc.) lub komparator (np. `std::less`, `std::greater`).

```cpp
template <typename T, typename Compare = std::less<T>, typename Container = std::vector<T>>
class PriorityQueue
{
    Container items_;
    [[no_unique_address]] Compare comp_;    
    
public:
    PriorityQueue() : items_(), comp_() {}

    void insert(const T& item)
    {
        auto it = std::lower_bound(items_.begin(), items_.end(), item, comp_);
        items_.insert(it, item);
    }
    
    const T& front() const { return items_.front(); }

    //...
};

// Użycie z domyślnym komparatorem
PriorityQueue<int> numbers_ascending;  // std::less<int>

// Użycie z własnym komparatorem
PriorityQueue<int, std::greater<int>> numbers_descending;

// Użycie z własnym komparatorem i kontenerem
PriorityQueue<int, std::greater<int>, std::deque<int>> numbers_descending_deque;    
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
    // przypisanie stosu o elementach typu U
    template <typename U>
    Stack<T>& operator=(const Stack<U>&);
}; 

template <typename T>
    template <typename U>
Stack<T>& Stack<T>::operator=(const Stack<U>& source)
{
    //...
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
