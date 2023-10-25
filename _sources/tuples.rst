************
Krotki w C++
************

Krotki – motywacja
==================

Chcemy napisać funkcję zwracającą trzy wartości statystyk dla wektora liczb całkowitych:

* wartość minimalną
* wartość maksymalną
* wartość średnią

**Problem** – funkcja może zwrócić tylko jedną wartość

**Rozwiązanie 1** – przekazanie parametrów przez referencję

.. code:: cpp

    void calculate_stats(const vector<int>& data, int& min, int& max, double& avg) 
    { 
        min = *min_element(data.begin(), data.end());
        max = *max_element(data.begin(), data.end());
        avg = accumulate(data.begin(), data.end(), 0.0) / data.size();
    }

**Rozwiązanie 2** – użycie ``std::tuple``

.. code:: cpp

    std::tuple<int, int, double> calculate_stats(const vector<int>& data)
    {
        auto min = *min_element(data.begin(), data.end());
        auto max = *max_element(data.begin(), data.end());
        auto avg = accumulate(data.begin(), data.end(), 0.0) / data.size();

        return std::tuple<int, int, double>(min, max, avg);
    }

    
    // Przykładowe użycie 
    vector<int> data = { 5, 1, 35, 321, 23, 5, 9, 88, 44, 324 };

    auto stats = calculate_stats(data);

    cout << "Min: " << get<0>(stats) 
         << "; Max: " << get<1>(stats) 
         << "; Avg: " << get<2>(stats) << endl;

Konstruowanie krotek
====================

Konstrukcja obiektu typu ``std::tuple`` wymaga deklaracji typu i (opcjonalnego) udostępnienia listy wartości początkowych, zgodnych co do typu z typami poszczególnych elementów montowanej krotki. 

Konstruktor domyślny krotki inicjalizuje wartościowo każdy element (*value initialized*).

.. code-block:: cpp

    tuple<int, double, string> triple(42, 3.14, "Test krotki");
    tuple<short, string> another;  // default value for every element

Funkcja ``make_tuple(wart1, wart2, ...)`` tworzy krotkę, dedukując typy elementów na podstawie typów argumentów wywołania.

.. code:: cpp

    tuple<int, double> get_values() 
    {
        return std::make_tuple(3, 5.56);
    }

Krotki z referencjami
=====================

Funkcja ``make_tuple()`` określa domyślnie typy elementów jako modyfikowalne i niereferencyjne. 

.. code:: cpp 

    void foo(const A& a, B& b) 
    { 
        //... 
        make_tuple(a, b); // tworzy tuple<A, B>
        //...
    }

Aby elementy krotki były referencjami należy skorzystać z wrapperów ``std::ref() oraz std::cref()``

.. code:: cpp 

    A a; B b; const A ca = a; 

    make_tuple(cref(a), b); // tworzy tuple<const A&, B> 

    make_tuple(ref(a), b); // tworzy tuple<A&, B> 

    make_tuple(ref(a), cref(b)); // tworzy tuple<A&, const B&> 

    make_tuple(cref(ca)); // tworzy tuple<const A&> 

    make_tuple(ref(ca)); // tworzy tuple<const A&> 

Odwołania do elementów krotek
=============================

Elementy krotki są dostępne przy pomocy zewnętrznej funkcji ``get<Index>()`` zwracającej referencję do odpowiedniego elementu krotki.
    
.. code:: cpp 

    double d = 2.7; A a; 

    tuple<int, double&, const A&> t(1, d, a); 
    const tuple<int, double&, const A&> ct = t; 

    int i = get<0>(t); 
    int j = get<0>(ct); // ok 
    get<0>(t) = 5; // ok 
    get<0>(ct) = 5; // błąd! nie można przypisać do const 
    double e = get<1>(t); // ok 
    get<1>(t) = 3.14; // ok 
    get<2>(t) = A(); // błąd! nie można przypisać do const 
    A aa = get<3>(t); // błąd! indeks poza zakresem ... 
    ++get<0>(t); // ok, można używać jak zmiennej 

Przypisywanie i kopiowanie krotek
=================================

Krotki można przypisywać i kopiować (wywołując konstruktor kopiujący) pod warunkiem, że dla odpowiednich typów krotki źródłowej i docelowej dostępne są wymagane konwersje.

.. code:: cpp

    class A {}; 
    class B : public A {};
    struct C { C(); C(const B&); }; 
    struct D { operator C() const; };
    ... 
    tuple<char, B*, B, D> t; 
    ... 
    tuple<int, A*, C, C> a(t); // ok 
    a = t; // ok 

Porównywanie krotek
===================

Krotki redukują operatory ``==``, ``!=``, ``<``, ``>``, ``<=`` i ``>=`` do odpowiadających im operatorów typów przechowywanych w krotce.

Równość pomiędzy krotkami a i b jest zdefiniowana przez:

* ``a == b`` jeśli dla każdego ``i``: ``a[i] == b[i]``
* ``a != b`` jeśli istnieje ``i``: ``a[i] != b[i]``

Operatory ``<``, ``>``, ``<=`` oraz ``>=`` implementują porównywanie leksykograficzne. 

.. code:: cpp 

    tuple<string, int, A> t1("same?", 2, A()); 
    tuple<string, long, A> t2("same?", 2, A()); 

    t1 == t2; // true 


Wiązanie zmiennych w krotki
===========================

Funkcja szablonowa ``std::tie`` umożliwia wiązanie samodzielnych zmiennych w krotki. Wszystkie elementy krotki utworzonej przez ``std::tie`` są modyfikowalnymi referencjami.

.. code:: cpp 

    vector<int> data = { 5, 1, 35, 321, 23, 5, 9, 88, 44, 324 };

    int min, max;
    double avg;

    tie(min, max, avg) = calculate_stats(data);
    
    cout << "Min: " << min << "; Max: " << max << "; Avg: " << avg << endl;


Wyjście: 

::

    1 324 85.5


Mechanizm wiązania działa również z obiektami szablonu ``std::pair<T>``. 

Algorytm ``std::minmax_element()`` zwraca parę iteratorów wskazujących na wystąpienia najmniejszej i największej wartości
w podanym zakresie. Zwracaną parę można przypisać do zmiennych za pomocą funkcji ``std::tie()``

.. code:: cpp

    std::vector<int>::iterator min_it, max_it;
    tie(min_it, max_it) = std::minmax_element(data.begin(), data.end());

Obiekt ``std::ignore`` umożliwia ignorowanie elementu krotki przy operacji wiązania zmiennych

.. code:: cpp
 
    int min, max;

    tie(min, max, ignore) = calculate_stats(data);

Krotki – podsumowanie
=====================================

* Krotki umożliwiają łatwe zwracanie wielu wartości z funkcji i wiązanie ich z samodzielnymi zmiennymi
* Upraszczają implementację operatorów porównań dla klas użytkownika
