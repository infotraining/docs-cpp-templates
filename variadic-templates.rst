
Variadic templates
==================

W C++11 szablony mogą akceptować dowolną ilość (również zero) parametrów. Jest to możliwe dzięki użyciu specjalnego grupowego parametru
szablonu tzw. *parameter pack*, który reprezentuje wiele lub zero parametrów szablonu.

Parameter pack
--------------

*Parameter pack* może być

* grupą parametrów szablonu

  .. code:: cpp

      template<typename... Ts> // template parameter pack
      class tuple
      {
          //...
      };

      tuple<int, double, string&> t1; // 3 arguments: int, double, string&
      tuple<> empty_tuple; // 0 arguments

* grupą argumentów funkcji szablonowej

  .. code:: cpp

      template <typename T, typename... Args>
      shared_ptr<T> make_shared(Args&&... params)
      {
          //...
      }

      auto sptr = make_shared<Gadget>(10); // Args as template param: int
                                           // Args as function param: int&&


Rozpakowanie paczki parametrów
------------------------------

Podstawową operacją wykonywaną na grupie parametrów szablonu jest rozpakowanie jej za pomocą operatora ``...`` (tzw. *pack expansion*).

.. code:: cpp

    template <typaname... Ts>  // template  parameter pack
    struct X
    {
        tuple<Ts...> data;  // pack expansion
    };

Rozpakowanie paczki parametrów (*pack expansion*) może zostać zrealizowane przy pomocy wzorca zakończonego elipsą ``...``:

.. code:: cpp

    template <typaname... Ts>  // template  parameter pack
    struct XPtrs
    {
        tuple<Ts const*...> ptrs;  // pack expansion
    };

    XPtrs<int, string, double> ptrs; // contains tuple<int const*, string const*, double const*>

Najczęstszym przypadkiem użycia wzorca przy rozpakowaniu paczki parametrów jest implementacja **perfect forwarding'u**:

.. code:: cpp

    template <typename... Args>
    void make_call(Args&&... params)
    {
        callable(forward<Args>(params)...);
    }


Idiom Head/Tail
---------------

Praktyczne użycie **variadic templates** wykorzystuje często idiom **Head/Tail** (znany również **First/Rest**). 

Idiom ten polega na zdefiniowaniu wersji szablonu akceptującego dwa parametry:
- pierwszego ``Head``
- drugiego ``Tail`` w postaci paczki parametrów

W implementacji wykorzystany jest parametr (lub argument) typu ``Head``, po czym rekursywnie wywołana jest implementacja dla rozpakowanej
paczki parametrów typu ``Tail``.

Dla szablonów klas idiom wykorzystuje specjalizację częściową i szczegółową (do przerwania rekurencji):

.. code:: cpp

    template <typename... Types>
    struct Count;

    template <typename Head, typename... Tail>
    struct Count<Head, Tail...>
    {
        constexpr static int value = 1 + Count<Tail...>::value; // expansion pack
    };

    template <>
    struct Count<>
    {
        constexpr static int value = 0;
    };

    //...
    static_assert(Count<int, double, string&>::value == 3, "must be 3");


W przypadku szablonów funkcji rekurencja może być przerwana przez dostarczenie odpowiednio przeciążonej funkcji.
Zostanie ona w odpowiednim momencie rozwijania rekurencji wywołana.

.. code:: cpp

    void print()
    {}

    template <typename T, typename... Tail>
    void print(const T& arg1, const Tail&... params)
    {
        cout << arg1 << endl;
        print(params...); // function parameter pack expansion
    }

Operator sizeof...
------------------

Operator ``sizeof...`` umożliwia odczytanie na etapie kompilacji ilości parametrów w grupie.

.. code:: cpp

    template <typename... Types>
    struct VerifyCount
    {
        static_assert(Count<Types...>::value == sizeof...(Types),
                        "Error in counting number of parameters");
    };

Forwardowanie wywołań funkcji
-----------------------------

**Variadic templates** są niezwykle przydatne do forwardowania wywołań funkcji.

.. code:: cpp
    
    template <typename T, typename... Args>
    std::unique_ptr<T> make_unique(Args&&... params)
    {
        return std::unique_ptr<T>(new T(forward<Args>(params)...);
    }

Ograniczenia paczek parametrów
------------------------------

Klasa szablonowa może mieć tylko jedną paczkę parametrów i musi ona zostać umieszczona na końcu listy parametrów szablonu:

.. code:: cpp

    template <size_t... Indexes, typename... Ts>  // error
    class Error;

Można obejść to ograniczenie w następujący sposób:

.. code:: cpp

    template <size_t... Indexes> struct IndexSequence {};

    template <typename Indexes, typename Ts...>
    class Ok;

    Ok<IndexSequence<1, 2, 3>, int, char, double> ok;


Funkcje szablonowe mogą mieć więcej paczek parametrów:

.. code:: cpp

    template <int... Factors, typename... Ts>
    void scale_and_print(Ts const&... args)
    {
        print(ints * args...);
    }

    scale_and_print<1, 2, 3>(3.14, 2, 3.0f);  // calls print(1 * 3.14, 2 * 2, 3 * 3.0)

**Uwaga!**
Wszystkie paczki w tym samym wyrażeniu rozpakowującym muszą mieć taki sam rozmiar.

.. code:: cpp

    scale_and_print<1, 2>(3.14, 2, 3.0f);  // error


"Nietypowe" paczki parametrów
-----------------------------

-  Podobnie jak w przypadku innych parametrów szablonów, paczka
   parametrów nie musi być paczką typów, lecz może być paczką stałych
   znanych na etapie kompilacji

.. code:: c++

    template <size_t... Values>
    struct MaxValue;  // primary template declaration
    
    template <size_t First, size_t... Rest>
    struct MaxValue<First, Rest...>
    {
        static constexpr size_t rvalue = MaxValue<Rest...>::value;
        static constexpr size_t value = (First < rvalue) ? rvalue : First;    
    };
    
    template <size_t Last>
    struct MaxValue<Last>
    {
        static constexpr size_t value = Last;  // termination of recursive expansion
    };


.. code:: c++

    static_assert(MaxValue<1, 5345, 3, 453, 645, 13>::value == 5345, "Error");


Variadic Mixins
---------------

**Variadic templates** mogą być skutecznie wykorzystane do implementacji
klas **mixin**

.. code:: c++

    #include <vector>
    #include <string>
    
    template <typename... Mixins>
    class X : public Mixins...
    {
    public:
        X(Mixins&&... mixins) : Mixins(mixins)... 
        {}
    };



Klasa taka może zostać wykorzystana później w następujący sposób:

.. code:: c++

    X<std::vector<int>, std::string> x({ 1, 2, 3 }, "text");
    
    x.std::string::size();


.. parsed-literal::

    (unsigned long) 4


.. code:: c++

    x.std::vector<int>::size();


.. parsed-literal::

    (unsigned long) 3


Curiously-Recurring Template Parameter (CRTP)
---------------------------------------------

Ciekawym sposobem użycia **variadic templates** jest implementacja
znanego idiomu **CRTP**.

Zaimplementujmy najpierw klasę licznika obiektów:

.. code:: c++

    template <typename T>
    class Counter 
    {
    public:
        Counter() { ++counter_; }
        ~Counter() { --counter_; }
        static size_t count() { return counter_; }
    private:
        static size_t counter_;
    };
    
    template <typename T> size_t Counter<T>::counter_;


Klasa ``Counter`` może zostać wielokrotnie użyta do implementacji
zliczania obiektów za pomocą idiomu **CRTP**. Dziedzicząc typ ``T`` po
``Counter<T>`` wstrzykujemy do ``T`` implementację zliczania obiektów.

.. code:: c++

    class Thing : public Counter<Thing> // OK
    {
    };


.. code:: c++

    Thing thing1, thing2;
    {
        Thing thing3;    
    }
    Thing::count();


.. parsed-literal::

    (unsigned long) 2


Innym praktycznym zastosowaniem **CRTP** jest implementacja operatorów
porównań dla klas.

-  Operatory ``==`` oraz ``!=`` mogą zostać zaimplementowane za pomocą
   funkcji pomocniczej ``equal_to()``.
-  Operatory ``<``, ``<=``, ``>``, ``>=`` mogą zostać zaimplementowane
   za pomocą funkcji ``equal_to()``, ``less_than()`` oraz
   ``greater_than()``
-  Klasy szablonowe ``Eq<T>`` oraz ``Rel<T>`` implementują generyczne
   operatory porównań wykorzystujące wyżej wymienione funkcje pomocnicze
   (operatory te są zadeklarowane jako zaprzyjaźnione - ``friend``)

.. code:: c++

    template <typename T>
    class Eq
    {
        friend bool operator==(T const& a, T const& b) { return a.equal_to(b); }
        friend bool operator!=(T const& a, T const& b) { return !a.equal_to(b); }
    };
    
    template <typename T>
    class Rel
    {
        friend bool operator<(T const& a, T const& b) { return a.less_than(b); }
        friend bool operator<=(T const& a, T const& b) { return !b.less_than(a); }
        friend bool operator>(T const& a, T const& b) { return b.less_than(a); }
        friend bool operator>=(T const& a, T const& b) { return !a.less_than(b); }    
    };


.. code:: c++

    struct AnotherThing : public Eq<AnotherThing>
    {
        int value;
        
        AnotherThing(int value) : value{value}
        {}
        
        bool equal_to(AnotherThing const& other) const
        {
            return value == other.value;
        }
    };


.. code:: c++

    AnotherThing at1{1};
    AnotherThing at2{2};
    
    (at1 != at2);


.. parsed-literal::

    (bool) true


.. code:: c++

    (at1 == at2);


.. parsed-literal::

    (bool) false


.. TODO - how it works


Użycie szablonu jako parametru szablonu upraszcza stosowanie **CRTP**

.. code:: c++

    template <template <typename> class CRTP>
    struct OtherThing : public CRTP<OtherThing<CRTP>>
    {
        int value;
        
        OtherThing(int value = 0) : value{value}
        {}
        
        bool equal_to(OtherThing const& other) const
        {
            return value == other.value;
        }
        
        bool less_than(OtherThing const& other) const
        {
            return value < other.value;
        }
    };


.. code:: c++

    OtherThing<Counter> counted_thing;
    OtherThing<Eq> equality_comparable_thing;
    OtherThing<Rel> relational_thing;


Jeżeli chcemy użyć wielu implementacji **CRTP** (np.
``Thing<Counter, Eq>``) dla jednej klasy musimy wykorzystać **variadic
templates**:

.. code:: c++

    template <template <typename> class... CRTPs>
    struct SuperThing : public CRTPs<SuperThing<CRTPs...>>...
    {
        int value;
        
        SuperThing(int value = 0) : value{value}
        {}
        
        bool equal_to(SuperThing const& other) const
        {
            return value == other.value;
        }
        
        bool less_than(SuperThing const& other) const
        {
            return value < other.value;
        }
    };


.. code:: c++

    SuperThing<Counter, Eq, Rel> a_thing{10};
    SuperThing<Counter, Eq, Rel> b_thing{20};
    
    a_thing.count();

    std::cout << (a_thing < b_thing) << std::endl;

    

W implementacji klasy szablonowej ``SuperThing``: 

* ``SuperThing`` jest nazwą szablonu 
* ``CRTPs`` jest szablonową paczką parametrów szablonu (*template template parameter pack*) 
* ``SuperThing<CRTPs...>`` jest nazwą typu 
* ``CRTPs<SuperThing<CRTPS...>>`` jest wzorcem paczki parametrów 
* ``CRTPs<SuperThing<CRTPS...>>...`` jest rozwinięciem wzorca 
* ``public CRTPs<SuperThing<CRTPs...>>...`` jest listą klas bazowych