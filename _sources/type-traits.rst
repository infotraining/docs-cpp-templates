Biblioteka klas cech typów - <type_traits>
==========================================

Biblioteka standardowa zawiera zestaw szablonów klas umożliwiających
cechowanie typów na etapie kompilacji. Dzięki cechom typów (**type
traits**) na etapie kompilacji:

* wybierana jest optymalna (wydajna) implementacja danej funkcjonalności (z wykorzystaniem SFINAE). 
* przeprowadzana jest statyczna asercja umożliwiająca wczesne wychwycenie błędów

Meta-funkcje
------------

Technika cechowania wykorzystuje meta-funkcje, najczęściej implementowane jako szablony klas przyjmujące jako parametr
cechowany typ i zwracające: 

* ``type_trait<T>::type`` - nowy typ będący efektem transformacji (np. usunięcia modyfikatora ``const``)
* ``type_trait<T>::value`` - stałą statyczną wartość ``constexpr`` (najczęściej typu ``bool``)

Meta-funkcje zwracające wartość
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Przykład implementacji meta-funkcji zwracającej stałą wartość ewaluowaną na etapie kompilacji:

.. code-block:: c++

    template<typename T>
    struct TypeSize 
    {
        static constexpr std::size_t value = sizeof(T);
    };

Wykorzystanie meta-funkcji:

.. code-block:: c++

    std::cout << TypeSize<int>::value << std::endl;

Meta-funkcje zwracające typy
~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Przy implementacji meta-funkcji często stosujemy specjalizację szablonów:

.. code-block:: c++

    // generic implementation for containers
    template<typename C>
    struct ElementT 
    {
        using type = typename C::value_type
    ;

    //partial specialization for arrays of known bounds
    template<typename T, std::size_t N>
    struct ElementT<T[N]> 
    {          
        using type = T;     
    };

    //partial specialization for arrays of unknown bounds
    template<typename T> 
    struct ElementT<T[]> 
    {          
        using type = T;
    };

Meta-funkcje mogą być wykorzystane w innych szablonach:

.. code-block:: c++

    template <typename Container>
    void process(const Container& c)
    {
        typename ElementT<Container>::type item{};

        //...
    }

 
Stałe całkowite - *integral constants*
--------------------------------------

Szablon ``std::integral_constant`` pozwala opakować w w postaci
meta-funkcji stałą zdefiniowaną na etapie kompilacji.

.. code:: c++

    template<class T, T v>
    struct integral_constant 
    {
        static constexpr T value = v;
        typedef T value_type;
        typedef integral_constant type; // using injected-class-name
        constexpr operator value_type() const noexcept { return value; }
        constexpr value_type operator()() const noexcept { return value; } //since c++14
    };


Wykorzystanie meta-funkcji ``integral_constant`` wygląda następująco:

.. code:: c++

    integral_constant<int, 5>::value;


.. parsed-literal::

    (const int) 5


Biblioteka standardowa definiuje pomocniczy alias dla stałej typu ``bool``:

.. code:: c++

    template <bool v>
    using bool_constant = integral_constant<bool, v>;


Zdefiniowane są również dwa typowe przypadki takich stałych:

.. code:: c++

    using true_type = bool_constant<true>; // integral_constant<bool, true>
    using false_type = bool_constant<false>; // integral_constant<bool, false>


Klasy cech typów
----------------

Klasy cech transformujące typy
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Często w trakcie pisania kodu szablonowego zachodzi potrzeba transformacji typu określonego
parametru szablonu (np. wymagane jest usunięcie lub dodanie referencji, modyfikatora ``const`` lub ``volatile``, itp.).

W takim przypadku możemy posłużyć się klasą cechy transformującej (implementowaną jako meta-funkcja):

.. code-block:: c++

    template <typename T>
    struct remove_reference
    {
        using type = T;
    };

    template <typename T>
    struct remove_reference<T&>
    {
        using type = T;
    };

    template <typename T>
    struct remove_reference<T&&>
    {
        using type = T;
    };

Od C++14 do klas cech dodane są odpowiednie aliasy szablonów, które umożliwiają uniknięcie konieczności deklaracji ``typename``
przed zagnieżdżonym typem ``type``:

.. code-block:: c++

    template <typename T>
    using remove_reference_t = typename remove_reference<T>::type;

Użycie cech transformujących wygląda następująco:

.. code-block:: c++

    auto closure = [](auto&& item) {
        remove_reference_t<decltype(item)> item_cpy = item;
    };

Cechy transformujące typy w bibliotece standardowej
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Biblioteka standardowa w nagłówku ``<type_traits>`` definiuje zbiór klas cech transformujących:

+--------------------------+------------------------------------------------------------------+
|          Cecha           |                       Rezultat ``::value``                       |
+==========================+==================================================================+
| ``remove_reference``     | usuwa referencję z typu (``int& -> int``)                        |
+--------------------------+------------------------------------------------------------------+
| ``add_lvalue_reference`` | dodaje lvalue referencję  (``double -> double&``)                |
+--------------------------+------------------------------------------------------------------+
| ``add_rvalue_reference`` | dodaje rvalue referencję  (``double -> double&&``)               |
+--------------------------+------------------------------------------------------------------+
| ``remove_pointer``       | usuwa wskaźnik z typu (``int* -> int``)                          |
+--------------------------+------------------------------------------------------------------+
| ``add_pointer``          | dodaje wskaźnik (``int -> int*``)                                |
+--------------------------+------------------------------------------------------------------+
| ``remove_const``         | usuwa modyfikator ``const``  (``const int& -> int&``)            |
+--------------------------+------------------------------------------------------------------+
| ``remove_volatile``      | usuwa modyfikator ``volatile`` (``volatile int -> int``)         |
+--------------------------+------------------------------------------------------------------+
| ``remove_cv``            | usuwa modyfikatory ``const`` i ``volatile``                      |
+--------------------------+------------------------------------------------------------------+
| ``add_const``            | dodaje modyfikator ``const``  (``int -> const int``)             |
+--------------------------+------------------------------------------------------------------+
| ``add_volatile``         | dodaje modyfikator ``volatile``  (``double -> volatile double``) |
+--------------------------+------------------------------------------------------------------+

Cecha ``std::decay``
^^^^^^^^^^^^^^^^^^^^

Przydatną cechą jest zdefiniowana w bibliotece standardowej cecha ``std::decay``.

Dokonuje ona transformacji odpowiadającej następującym przekształceniom:

* usuwane są referencje
* usuwane są modyfikatory ``const`` lub ``volatile``
* tablice konwertowane są do wskaźników
* funkcje konwertowane są do wskaźników do funkcji

.. code-block:: c++

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


Klasy cech - predykaty
~~~~~~~~~~~~~~~~~~~~~~

Implementacja cech typów, które pełnią rolę predykatów, polega zwykle na zdefiniowaniu ogólnego
szablonu dziedziczącego po ``false_type`` (dla typów nie posiadających
określonej cechy). Kolejnym krokiem jest dostarczenie wersji
specjalizowanej szablonu dla typów z cechą, która dziedziczy po typie
``true_type``.

Specjalizacja szablonu może być całkowita:

.. code:: c++

    template <typename T>
    struct IsVoid : std::false_type
    {};
    
    template <>
    struct IsVoid<void> : std::true_type
    {};


.. code:: c++

    IsVoid<int>::value;


.. parsed-literal::

    (const bool) false


.. code:: c++

    IsVoid<void>::value;


.. parsed-literal::

    (const bool) true


lub częściowa:


.. code:: c++

    template <typename T>
    struct IsPointer : std::false_type{};
    
    template <typename T>
    struct IsPointer<T*> : std::true_type{};


.. code:: c++

    IsPointer<int>::value;


.. parsed-literal::

    (const bool) false


.. code:: c++

    IsPointer<const int*>::value;


.. parsed-literal::

    (const bool) true


Standardowe cechy typów - predykaty
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Biblioteka standardowa definiuje szeroki zbiór meta-funkcji, które
umożliwiają odpytanie na etapie kompilacji, czy dany typ posiada
odpowiednie cechy.

Cechy podstawowe
^^^^^^^^^^^^^^^^

.. tabularcolumns:: |l|L|

+-----------------------------------+-------------------------------------------------------------+
|         Cecha podstawowa          |                    Rezultat ``::value``                     |
+===================================+=============================================================+
| ``is_array<T>``                   | ``true`` jeśli ``T`` jest typem tablicowym                  |
+-----------------------------------+-------------------------------------------------------------+
| ``is_class<T>``                   | ``true`` jeśli ``T`` jest klasą                             |
+-----------------------------------+-------------------------------------------------------------+
| ``is_enum<T>``                    | ``true`` jeśli ``T`` jest typem wyliczeniowym               |
+-----------------------------------+-------------------------------------------------------------+
| ``is_floating_point<T>``          | ``true`` jeśli ``T`` jest typem zmiennoprzecinkowym         |
+-----------------------------------+-------------------------------------------------------------+
| ``is_function<T>``                | ``true`` jeśli ``T`` jest funkcją                           |
+-----------------------------------+-------------------------------------------------------------+
| ``is_integral<T>``                | ``true`` jeśli ``T`` jest typem całkowitym                  |
+-----------------------------------+-------------------------------------------------------------+
| ``is_member_object_pointer<T>``   | ``true`` jeśli ``T`` jest wskaźnikiem do składowej          |
+-----------------------------------+-------------------------------------------------------------+
| ``is_member_function_pointer<T>`` | ``true`` jeśli ``T`` jest wskaźnikiem do funkcji  składowej |
+-----------------------------------+-------------------------------------------------------------+
| ``is_pointer<T>``                 | ``true`` jeśli ``T`` jest typem wskaźnikowym                |
|                                   | (ale nie wskaźnikiem do składowej)                          |
+-----------------------------------+-------------------------------------------------------------+
| ``is_lvalue_reference<T>``        | ``true`` jeśli ``T`` jest referencją do l-value             |
+-----------------------------------+-------------------------------------------------------------+
| ``is_rvalue_reference<T>``        | ``true`` jeśli ``T`` jest referencją do r-value             |
+-----------------------------------+-------------------------------------------------------------+
| ``is_union<T>``                   | ``true`` jeśli ``T`` jest unią                              |
|                                   | (bez wsparcia kompilatora zawsze zwraca ``false``           |
+-----------------------------------+-------------------------------------------------------------+
| ``is_void<T>``                    | ``true`` jeśli ``T`` jest typu void                         |
+-----------------------------------+-------------------------------------------------------------+
| ``is_null_pointer<T>``            | ``true`` jeśli ``T`` jest typu ``std::nullptr_t``           |
+-----------------------------------+-------------------------------------------------------------+



Cechy kompozytowe
^^^^^^^^^^^^^^^^^

Cechy kompozytowe są kompozycją najczęściej kilku cech podstawowych.


.. tabularcolumns:: |l|L|

+--------------------------+----------------------------------------------------------------------------------+
| Cecha grupowana          | Rezultat ``::value``                                                             |
+==========================+==================================================================================+
| ``is_arithmetic<T>``     | ``is_integral<T>::value ||``                                                     |
|                          | ``is_floating_point<T>::value``                                                  |
+--------------------------+----------------------------------------------------------------------------------+
| ``is_fundamental<T>``    | ``is_arithmetic<T>::value || is_void<T>::value || is_null_pointer<T>::value``    |
+--------------------------+----------------------------------------------------------------------------------+
| ``is_compound<T>``       | ``!is_fundamental<T>::value``                                                    |
+--------------------------+----------------------------------------------------------------------------------+
| ``is_object<T>``         | ``is_scalar<T>::value || is_array<T>::value  || is_union<T>::value  ||``         | 
|                          | ``is_class<T>::value``                                                           |
+--------------------------+----------------------------------------------------------------------------------+
| ``is_reference<T>``      | ``is_lvalue_reference<T>`` || ``is_rvalue_reference<T>``                         |
+--------------------------+----------------------------------------------------------------------------------+
| ``is_member_pointer<T>`` | ``is_member_object_pointer<T> || is_member_function_pointer<T>``                 |
+--------------------------+----------------------------------------------------------------------------------+
| ``is_scalar<T>``         | ``is_arithmetic<T>::value || is_enum<T>::value || is_null_pointer<T>::value``    |
|                          | ``is_pointer<T>::value || is_member_pointer<T>::value``                          |
+--------------------------+----------------------------------------------------------------------------------+

Właściwości typów
^^^^^^^^^^^^^^^^^

Standard używa terminu **właściwość typu** w celu zdefiniowania cechy opisującej wybrane atrybuty typu.

Wybrane właściwości typu:


.. tabularcolumns:: |l|L|

+------------------------------+------------------------------------------------------+
|       Właściwość typu        |                 Rezultat ``::value``                 |
+==============================+======================================================+
| ``is_const<T>``              | ``true`` jeśli ``T`` jest typem ``const``            |
+------------------------------+------------------------------------------------------+
| ``is_volatile<T>``           | ``true`` jeśli ``T`` jest typem ulotnym              |
+------------------------------+------------------------------------------------------+
| ``is_polymorphic<T>``        | ``true`` jeśli ``T`` posiada przynajmniej jedną      |
|                              | funkcję wirtualną                                    |
+------------------------------+------------------------------------------------------+
| ``is_trivial<T>``            | ``true`` jeśli ``T`` jest typem trywialnym           |
+------------------------------+------------------------------------------------------+
| ``is_trivially_copyable<T>`` | ``true`` jeśli ``T`` jest trywialnie kopiowalne      |
|                              |                                                      |
+------------------------------+------------------------------------------------------+
| ``is_standard_layout<T>``    | ``true`` jeśli ``T`` jest typem o standardowym       |
|                              | layout'cie                                           |
+------------------------------+------------------------------------------------------+
| ``is_pod<T>``                | ``true`` jeśli ``T`` jest typem POD                  |
+------------------------------+------------------------------------------------------+
| ``is_abstract<T>``           | ``true`` jeśli ``T`` jest typem abstrakcyjnym        |
+------------------------------+------------------------------------------------------+
| ``is_unsigned<T>``           | ``true`` jeśli ``T`` jest typem całkowitym bez znaku |
|                              | lub typem wyliczeniowym zdefiniowanym przy pomocy    |
|                              | typu ``unsigned``                                    |
+------------------------------+------------------------------------------------------+
| ``is_signed<T>``             | ``true`` jeśli ``T`` jest typem całkowitym ze        |
|                              | znakiem lub typem wyliczeniowym zdefiniowanym przy   |
|                              | pomocy typu ``signed``                               |
+------------------------------+------------------------------------------------------+

Cechy typów i statyczne asercje
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Jednym z podstawowych zastosowań cech typów jest wykorzystanie ich do
statycznych asercji w kodzie. Takie asercje nie pozwalają wygenerować
błędnego kodu i jednocześnie podnoszą czytelność komunikatów o błędach w
szablonach.

Przykład:

.. code:: c++

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