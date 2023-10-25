
Sekwencje indeksów
==================

Sekwencje indeksów (**index sequences**) 

* są wykorzystywane do meta-programowania z wykorzystaniem szablonów 
* są używane w połączeniu z *variadic templates* (w szczególności z możliwością rozwijania paczek parametrów) 
* umożliwiają prostszą implementację meta-algorytmów

Wybieranie elementów z krotki
-----------------------------

Chcemy napisać szablon funkcji, który umożliwi selekcję elementów krotki
w następujący sposób:

.. code:: cpp

    auto existing = make_tuple(0, 1.0, '2', "three");
    auto selected = select<1, 0, 3, 1, 2, 0>(existing);

Aby tego dokonać będziemy chcieli wywołać operację ``get<I>`` dla
każdego indeksu z listy parametrów szablonu. Argumentem wywołania
funkcji będzie krotka.

.. code:: c++

    #include <tuple>
    
    template <size_t... Indexes, typename... Ts>
    auto select(std::tuple<Ts...> const& tpl)
    {
        return std::make_tuple(std::get<Indexes>(tpl)...);
    }


W implementacji wykorzystaliśmy wzorzec ``get<Indexes>(tpl)``. Wzorzec
ten rozwinięty jako ``get<Indexes>(tpl)...`` skutkuje przekazaniem listy
elementów do funkcji ``std::make_tuple()``.


.. code:: c++

    auto existing = std::make_tuple(0, 1.0, '2', "three");
    auto selected = select<1, 3>(existing);
    
    std::get<1>(selected);


.. parsed-literal::

    "three"


Sekwencje indeksów w C++14
--------------------------

Sekwencje indeksów są często wykorzystywane w procesie rozwijania paczek
parametrów szablonu.

C++14 dostarcza zestaw gotowych narzędzi do generowania własnych
sekwencji indeksów:

.. code:: cpp

    template <typename T, T...> struct integer_sequence;

    template<size_t... I>
    using index_sequence = integer_sequence<int, I...>;

    template <size_t N>
    using make_index_sequence = make_integer_sequence<size_t, N>;


Uproszczona implementacja sekwencji indeksów w C++11
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. code:: c++

    template <size_t... Seq>
    struct IndexSequence
    {
        typedef std::index_sequence<Seq...> type;
        typedef size_t value_type;
        
        static constexpr size_t size() noexcept
        {
            return sizeof...(Seq);
        }    
    };
    
    template <size_t N, size_t I, typename Seq>
    struct MakeSequence;
    
    template <size_t N, size_t... Seq>
    struct MakeSequence<N, N, IndexSequence<Seq...>>  // ends recursion at N
        : IndexSequence<Seq...> 
    {}; 
    
    template <size_t N, size_t I, size_t... Seq>
    struct MakeSequence<N, I, IndexSequence<Seq...>>
        : MakeSequence<N, I+1, IndexSequence<Seq..., I>>
    {};
    
    template <size_t N>
    using MakeIndexSequence = typename MakeSequence<N, 0, IndexSequence<>>::type;


.. code:: c++

    #include <type_traits>
    
    std::is_same<std::index_sequence<0, 1, 2, 3, 4>, MakeIndexSequence<5>>::value;


.. parsed-literal::

    (const bool) true


Zastosowanie sekwencji indeksów
-------------------------------

Typowe zastosowanie sekwencji indeksów polega na rozpakowaniu
odpowiedniego wzorca dla **variadic templates**. Algorytm postępowania
zwykle wygląda następująco:

1. Utworzenie odpowiedniej sekwencji.
   
   .. code:: c++
       
       using I = std::make_index_sequence<N>; // [0; N)

2. Utworzenie obiektu dla którego chcemy wykorzystać sekwencję indeksów
   
   .. code:: c++
   
       std::array<int , N * 2> a;

       
3. Ekstrakcja sekwencji indeksów z ``index_sequence`` i aplikacja ich we
   wzorcu **variadic template**. Proces ekstrakcji może wykorzystywać:

   - mechanizm dedukcji typów dla argumentów szablonu:

     .. code:: c++

        #include <array>
        #include <algorithm>
        #include <iostream>
            
        namespace Impl1
        {
            template <typename T, size_t N, size_t... I>
            auto select_evens_impl(std::array<T, N> const& a, std::index_sequence<I...>)
            {
                return std::array<T, sizeof...(I)>{ a[I * 2]...};    
            }
                
            template <typename T, size_t N>
            auto select_evens(std::array<T, N> const& a)
            {
                using Indexes = std::make_index_sequence<N/2>;
                return select_evens_impl(a, Indexes{});
            }
        }


     .. code:: c++

           std::array<int, 11> a = { 0, 1, 2, 3, 4, 5, 6, 7, 8, 9, 10 };
           auto even_1 = Impl1::select_evens(a);
       
           for(const auto& item : even_1)
               std::cout << item << " ";


     .. parsed-literal::

            0 2 4 6 8 

   -  lub mechanizm częściowej specjalizacji szablonów

      .. code:: c++

            #include <array>
            #include <algorithm>
            #include <iostream>

            namespace Impl2
            {
                template <typename T, size_t N, typename Seq>
                struct SelectEvens;

                template <typename T, size_t N, size_t... I>
                struct SelectEvens<T, N, std::index_sequence<I...>>
                {
                    static auto apply(std::array<T, N> const& a) 
                    {
                        return std::array<T, sizeof...(I)>{ a[I * 2]...};
                    }
                };


                template <typename T, size_t N>
                auto select_evens(std::array<T, N> const& a)
                {
                    using Indexes = std::make_index_sequence<N/2>;

                    return SelectEvens<T, N, Indexes>::apply(a);
                }
            }


      .. code:: c++

            auto even_2 = Impl2::select_evens(a);

            for(const auto& item : even_2)
                std::cout << item << " ";


      .. parsed-literal::

          0 2 4 6 8 


Meta-programowanie z użyciem krotek
-----------------------------------

Implementacja foreach dla krotki
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Chcemy napisać funkcję umożliwiającą wywołanie funkcji, obiektu
funkcyjnego lub lambdy dla każdego elementu krotki.

Przykład użycia mógłby wyglądać następująco:

.. code:: cpp

    auto printer = [](auto const& item) { std::cout << "value: " << item << std::endl; };
    auto tpl = std::make_tuple(1, 3.14, 'a', "text"s);

    tuple_apply(printer, tpl);

Implementacja wymaga zastosowania sekwencji indeksów.

.. code:: c++

    #include <utility>
    #include <tuple>
    #include <string>
    #include <iostream>
    
    template <typename F, typename T, size_t... I>
    void apply_tuple_impl(F f, T const& t, std::index_sequence<I...>)
    {
        std::initializer_list<bool> { (f(std::get<I>(t)), false)... };
    }
    
    template <typename F, typename... Ts>
    inline void tuple_apply(F f, std::tuple<Ts...> const& t)
    {
        using Indexes = std::make_index_sequence<sizeof...(Ts)>;
        
        apply_tuple_impl(f, t, Indexes{});
    }


Implementacja wykorzystuje idiom z ``std::initializer_list``, który
gwarantuje prawidłową kolejność wywołania funkcji.

.. code:: c++

    using namespace std::literals;
    
    auto printer = [](auto const& item) { 
        std::cout << "value: " << item << std::endl; 
    };
    
    auto tpl = std::make_tuple(1, 3.14, 'a', "text"s);
    
    tuple_apply(printer, tpl);


.. parsed-literal::

    value: 1
    value: 3.14
    value: a
    value: text


Powyższy kod jest uproszczoną implementacją funkcji ``std::apply()``
wprowadzonej w C++14:

.. code:: cpp

    <template <typename F, typename Tuple, size_t... I>
    auto apply_impl(F&& f, Tuple&& t, std::index_sequence<I...>)
    {
        return std::forward<F>(f)(std::get<I>(std::forward<Tuple>(t))...);
    }

    template <typename F, class Tuple>
    auto apply(F&& f, Tuple&& t)
    {
        using Indexes = std::make_index_sequence<std::tuple_size<Tuple>::value>;
        
        return std::apply_impl(std::forward<F>(f), std::forward<Tuple>(t), Indexes{});
    }

Konwersja tablicy do krotki
~~~~~~~~~~~~~~~~~~~~~~~~~~~

Załóżmy, że mamy tablicę zdefiniowaną jako:

.. code:: cpp

    std::array<std::string, 10> a = { "one", "two", "three" };

Chcemy skonwertować ją do krotki przy pomocy następującej funkcji:

.. code:: cpp

    auto tpl = array2tuple(a); // tpl is a tuple<std::string, std::string, etc.>

Implementacja funkcji ``array2tuple()`` polega na: \* utworzeniu
sekwencji indeksów dla każdego elementu tablicy \* użyciu sekwencji do
rozwinięcia wzorca **variadic template** w wywołaniu ``make_tuple()``

.. code:: c++

    template <typename Array, size_t... I>
    inline constexpr auto array2tuple_impl(Array const& a, std::index_sequence<I...>)
    {
        return std::make_tuple(a[I]...);
    }
    
    template<typename T, size_t N, typename Indexes = std::make_index_sequence<N>>
    inline constexpr auto array2tuple(std::array<T, N> const& a)
    {
        return array2tuple_impl(a, Indexes{});
    } 


.. code:: c++

    std::array<int, 3> arr = { 1, 2, 3 };
    array2tuple(arr);


.. parsed-literal::

    (std::tuple<int, int, int>) { 1, 2, 3 }


Konwersja krotki do tablicy
~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. code:: c++

    #include <utility>
    #include <tuple>
    #include <iostream>
    
    template <typename... Ts, size_t... I>
    inline auto tuple2array_impl(std::tuple<Ts...> const& t, std::index_sequence<I...>)
    {
        using TT = std::common_type_t<Ts...>;
        return std::array<TT, sizeof...(Ts)> { static_cast<TT>(std::get<I>(t))...};
    }
    
    template <typename... Ts, typename Indexes = std::make_index_sequence<sizeof...(Ts)>>
    inline auto tuple2array(std::tuple<Ts...> const& t)
    {
        return tuple2array_impl(t, Indexes{});
    }


.. code:: c++

    auto t4 = std::make_tuple(1, -1L, 3.14);
    auto a3 = tuple2array(t4);
    
    for(const auto& item : a3)
        std::cout << item << " ";


.. parsed-literal::

    1 -1 3.14 


Operacje na sekwencjach indeksów
--------------------------------

Przesunięcie indeksów
~~~~~~~~~~~~~~~~~~~~~

Zaimplementujmy operację przesunięcia wartości indeksów o zadaną wartość
(poprzez dodanie tej wartości do każdego elementu sekwencji).

.. code:: c++

    #include <utility>
    
    template <size_t Shift, typename Seq>
    struct ShiftSequence;
    
    template <size_t Shift, size_t... Seq>
    struct ShiftSequence<Shift, std::index_sequence<Seq...>>
    {
        using type = std::index_sequence<(Shift + Seq)...>;
    };


.. code:: c++

    #include <type_traits>
    
    using OriginalSeq = std::make_index_sequence<5>;
    using ShiftedSeq = ShiftSequence<5, OriginalSeq>::type;
    
    static_assert(std::is_same<ShiftedSeq, std::index_sequence<5, 6, 7, 8, 9>>::value, "Error");
  

Zgodnie z konwencją wprowadzoną w C++14 tworzymy dodatkowo alias
zakończony ``_t``, który upraszacza wywoływanie metafunkcji
transformujących typy.

.. code:: c++

    template <size_t Shift, typename Seq>
    using ShiftSequence_t = typename ShiftSequence<Shift, Seq>::type;
    
    using OriginalSeq = std::make_index_sequence<5>;
    using ShiftedSeq = ShiftSequence_t<5, OriginalSeq>;
    
    static_assert(std::is_same<ShiftedSeq, std::index_sequence<5, 6, 7, 8, 9>>::value, "Error"); 


Transformacja indeksów
~~~~~~~~~~~~~~~~~~~~~~

Możemy zaimplementować meta-algorytm będący odpowiednikiem
``std::transform()`` z biblioteki STL.

Jako meta-funkcji transformującej na etapie kompilacji możemy użyć
szablonowego parametru szablonu (*template template parametr*).

.. code:: c++

    template <typename Seq, template <size_t> class Op>
    struct Transform;
    
    template <template <size_t> class Op, size_t... I>
    struct Transform<std::index_sequence<I...>, Op>
    {
        using type = std::index_sequence<Op<I>::value...>;
    };
    
    template <typename Seq, template <size_t> class Op>
    using Transform_t = typename Transform<Seq, Op>::type;
   

Użycie meta-algorytmu ``Transform_t`` wygląda następująco:

.. code:: c++

    template <size_t N>
    struct Square
    {
        static constexpr size_t value = N * N;
    };
    
    
    using OriginalSeq = std::make_index_sequence<5>;
    using ShiftedByOneSeq = ShiftSequence_t<1, OriginalSeq>;
    using NewSeq = Transform_t<ShiftedByOneSeq, Square>;
    
    static_assert(std::is_same<NewSeq, std::index_sequence<1, 4, 9, 16, 25>>::value, "Error");


Generowanie sekwencji
~~~~~~~~~~~~~~~~~~~~~

Meta-algorytm będą odpowiednikiem STL-owego ``generate_n()`` może być
zaimplementowany w następujący sposób:

.. code:: c++

    template <size_t N, template <size_t> class Gen>
    struct Generate_n
    {
        using type = Transform_t<std::make_index_sequence<N>, Gen>;
    };
    
    template <size_t N, template <size_t> class Gen>
    using Generate_n_t = typename Generate_n<N, Gen>::type;
    
    using Squares = Generate_n_t<5, Square>;
    static_assert(std::is_same<Squares, std::index_sequence<0, 1, 4, 9, 16>>::value, "Error");
   

Modyfikowanie indeksów za pomocą rekursji
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Odwracanie sekwencji
^^^^^^^^^^^^^^^^^^^^

Niektóre z meta-algorytmów wymagają rekurencyjnej implementacji z
wykorzystaniem idiomu **Head/Tail**.

Zaimplementujmy algorytm odwracający sekwencję indeksów:

.. code:: c++

    template <typename Seq>
    struct Reverse;
    
    template <typename Seq>
    using Reverse_t = typename Reverse<Seq>::type;
    
    template <typename Seq, size_t Item>
    struct PushBack;
    
    template <typename Seq, size_t Item>
    using PushBack_t = typename PushBack<Seq, Item>::type;
    
    template <size_t... I, size_t Item>
    struct PushBack<std::index_sequence<I...>, Item>
    {
        using type = std::index_sequence<I..., Item>;
    };
    
    template <size_t First, size_t... Rest>
    struct Reverse<std::index_sequence<First, Rest...>>
    {
        using Tail = std::index_sequence<Rest...>;
        using Rtail = Reverse_t<Tail>;
        using type = PushBack_t<Rtail, First>;
    };
    
    template <>
    struct Reverse<std::index_sequence<>>
    {
        using type = std::index_sequence<>;
    };


.. code:: c++

    using ReversedSquares = Reverse_t<Squares>;
    static_assert(std::is_same<ReversedSquares, std::index_sequence<16, 9, 4, 1, 0>>::value, "Error");
  

Usuwanie elementów sekwencji
^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Zaimplementujmy meta-algorytm usuwający z sekwencji indeksów indeksy
spełniające predykat przekazany za pomocą meta-funkcji:

.. code:: c++

    template <typename Seq, template <size_t> class Pred>
    struct RemoveIf;
    
    template <typename Seq, template <size_t> class Pred>
    using RemoveIf_t = typename RemoveIf<Seq, Pred>::type;
    
    template <typename Seq, size_t Item>
    struct PushFront;
    
    template <typename Seq, size_t Item>
    using PushFront_t = typename PushFront<Seq, Item>::type;
    
    template <size_t... I, size_t Item>
    struct PushFront<std::index_sequence<I...>, Item>
    {
        using type = std::index_sequence<Item, I...>;
    };
    
    template <template <size_t> class Pred, size_t First, size_t... Rest>
    struct RemoveIf<std::index_sequence<First, Rest...>, Pred>
    {
        using Tail = RemoveIf_t<std::index_sequence<Rest...>, Pred>;
        using type = std::conditional_t<Pred<First>::value, Tail, PushFront_t<Tail, First>>;
    };
    
    template <template <size_t> class Pred>
    struct RemoveIf<std::index_sequence<>, Pred>
    {
        using type = std::index_sequence<>;
    };
 

.. code:: c++

    template <size_t I>
    struct IsOdd
    {
        static constexpr bool value = I % 2;
    };

   
.. code:: c++

    using Indexes = std::make_index_sequence<10>;
    using Evens = RemoveIf_t<Indexes, IsOdd>;
    
    static_assert(std::is_same<Evens, std::index_sequence<0, 2, 4, 6, 8>>::value, "Error");
 

Rozwijanie wielu paczek parametrów
----------------------------------

Szablony klas mają dość istotne ograniczenie w postaci możliwości
zdefiniowania tylko jednej paczki argumentów na liście argumentów
szablonu. Rozwiązanie tego typu ograniczenia polega na zastosowaniu
zagnieżdżonych, częściowo specjalizowanych szablonów w celu rozpakowania
kolejnych paczek argumentów. Każdy poziom zagnieżdżenia rozpakowuje
pojedynczą paczkę argumentów. Na najniższym poziomie zagnieżdżenia
dostępna jest zawartość wszystkich paczek.

Implementacja podana poniżej ilustruje tę technikę dla sekwencji
indeksów, ale łatwo może być uogólniona dla dowolnych paczek argumentów.

.. code:: c++

    #include <utility>
    
    template <typename Seq1, typename Seq2, size_t... Seq3>
    struct Merge3
    {
        template <typename> struct Do1;
        
        template <size_t... DoSeq1>
        struct Do1<std::index_sequence<DoSeq1...>>
        {
            // we have Seq1 & Seq3 here
            
            template <typename> struct Do2;
            
            template <size_t... DoSeq2>
            struct Do2<std::index_sequence<DoSeq2...>>
            {
                // we have Seq1, Seq2 & Seq3 here
                using type = std::index_sequence<DoSeq1..., DoSeq2..., Seq3...>;
            };
            
            using type = typename Do2<Seq2>::type;
        };
        
        using type = typename Do1<Seq1>::type;
    };


Łączenie ze sobą sekwencji indeksów może zostać zaimplementowane za
pomocą techniki zagnieżdżonego rozpakowywania paczek:

.. code:: c++

    template <typename... IndexSequences>
    struct Concatenate;
    
    template <typename... IndexSequences>
    using Concatenate_t = typename Concatenate<IndexSequences...>::type;
    
    template <typename FirstSeq, typename... RestSeq>
    struct Concatenate<FirstSeq, RestSeq...>
    {
        template <typename HeadIndexes>
        struct First;
        
        template <size_t... HeadIndexes>
        struct First<std::index_sequence<HeadIndexes...>>
        {
            // recurse to concatenate tail sequences...
            using RestType = Concatenate_t<RestSeq...>;
            
            template <typename TailIndexes> struct Rest;
            template <size_t... TailIndexes>
            struct Rest<std::index_sequence<TailIndexes...>>
            {
                // extract merged tail, prepend first
                using type = std::index_sequence<HeadIndexes..., TailIndexes...>;
            };
            
            // ... using pass it up
            using type = typename Rest<RestType>::type;
            
        };
        
        using type = typename First<FirstSeq>::type;
    };
    
    template <>
    struct Concatenate<>
    {
        using type = std::index_sequence<>;
    };


Wykorzystanie meta-algorytmu ``Concatenate_t``:

.. code:: c++

    using IncreasingSeq = std::make_index_sequence<10>;
    using DecreasingSeq = Reverse_t<IncreasingSeq>;
    
    using Doppler = Concatenate_t<IncreasingSeq, DecreasingSeq>;
