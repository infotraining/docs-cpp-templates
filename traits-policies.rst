Klasy cech i wytycznych
=======================

Klasy cech
----------

*Klasy cech (traits)* reprezentują dodatkowe właściwości parametru szablonu, które mogą być pomocne na etapie implementacji kodu szablonu.

Case Study - Kumulowanie elementów sekwencji
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. code-block:: c++

    template <typename T>
    T accumulate(const T* begin, const T* end)
    {
        T total = T{};

        while (begin != end)
        {
            total += *begin;
            ++begin;
        }

        return total;
    }

Problemy:

* określenie typu zmiennej kumulującej
* utworzenie wartości zerowej

Parametryzacja typu zmiennej kumulującej
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

.. code-block:: c++

    template <typename T>
    struct AccumulationTraits;

    template <>
    struct AccumulationTraits<uint8_t>
    {
        typedef unsigned int AccumulatorType;
    };

    template<>
    struct AccumulationTraits<float>
    {
        typedef double AccumulatorType;
    };

Szablon ``AccumulationTraits`` zwany jest szablonem **cechy**, gdyż przechowuje cechę typu, który jest parametrem szablonu.

Algorytm korzystający z klasy cech wygląda następująco:

.. code-block:: c++

    template <typename T>
    typename AccumulationTraits<T>::AccumulatorType 
    accumulate(const T* begin, const T* end)
    {
        using AccT = typename AccumulationTraits<T>::AccumulatorType;

        AccT total = T{};

        while (begin != end)
        {
            total += *begin;
            ++begin;
        }

        return total;
    }

Cechy wartości
^^^^^^^^^^^^^^

Klasy cech mogą również zawierać informację o stałych charakterystycznych dla opisywanego typu. Możemy w klasie cechy zdefiniować wartość zerową dla typu.

.. code-block:: c++

    template<typename T> struct AccumulationTraits;

    template<>
    struct AccumulationTraits<uint8_t>
    {
        using AccumulatorType = int;
        static constexpr AccumulatorType zero = 0;
    };

    template<>
    struct AccumulationTraits<float> 
    {
        using AccumulatorType = float;
        static constexpr float zero = 0.0f;
    };

    template<>
    struct AccumulationTraits<BigInt> 
    {
        using AccumulationTraits = BigInt;
        inline static BigInt const zero = BigInt{0};  // OK since C++17
    };     

Algorytm korzystający z klasy cech uwzględniającej wartość zerową:

.. code-block:: c++

    template <typename T>
    typename AccumulationTraits<T>::AccumulatorType accumulate(const T* begin, const T* end)
    {
        using AccT = typename AccumulationTraits<T>::AccumulatorType;

        AccT total = AccumulationTraits<T>::zero; // wykorzystanie cechy

        while (begin != end)
        {
            total += *begin;
            ++begin;
        }

        return total;
    }

Parametryzowanie cech typów
^^^^^^^^^^^^^^^^^^^^^^^^^^^

Parametryzacja cech wymaga dodatkowych parametrów szablonu. Aby stosowanie sparametryzowanych cech typów było wygodne, należy wykorzystać domyślne wartości parametrów szablonu. 

.. code-block:: c++

    template <typename T, typename Traits = AccumulationTraits<T>>
    typename Traits::AccumulatorType accumulate(const T* begin, const T* end)
    {
        using AccT = typename Traits::AccumulatorType;

        AccT total = Traits::zero;

        while (begin != end)
        {
            total += *begin;
            ++begin;
        }

        return total;
    }


Klasy wytycznych
----------------

*Wytyczne (policies)* reprezentują konfigurowalne zachowania ogólnych funkcji i typów

Klasa wytycznych – klasa udostępniająca zbiór metod implementujących określony sposób zachowania (algorytm)


.. code-block:: c++

    template 
    <
       typename T, 
       typename AccumulationPolicy = Sum, 
       typename Traits = AccumulationTraits<T>
    >
    typename Traits::AccumulatorType accumulate (const T* begin, const T* end)
    {
        using AccT = typename Traits::AccumulatorType;

        AccT total = Traits::zero;

        while (begin != end)
        {
            AccumulationPolicy::accumulate(total, *begin);
            ++begin;
        }

        return total;
    }


Domyślna klasa wytycznej:

.. code-block:: c++

    struct Sum
    {
        template <typename T1, typename T2>
        static void accumulate(T1& total, const T2& value)
        {
            total += value;
        }
    };


Zmodyfikowana klasa wytycznej:

.. code-block:: c++

    struct Multiply
    {
        template <typename T1, typename T2>
        static void accumulate(T1& total, const T2& value)
        {
            total *= value;
        }
    };

    int main() 
    { 
        int data[] = {1,2,3,4,5}; 
        // wyświetl iloczyn wszystkich wartości 
        std::cout << "the product of the integer values is " <<   
            accumulate<int, MultiplyPolicy>(begin(data), end(data)) << '\n'; 
    } 


Klasy parametryzowane wytycznymi
--------------------------------

Konstruowanie klas parametryzowanych wytycznymi polega na składaniu skomplikowanego zachowania klasy z wielu małych klas (wytycznych).

Każda wytyczna:

* Określa jeden sposób zachowania lub implementacji
* Ustala interfejs dotyczący jednej konkretnej czynności
* Może być implementowana na wiele sposobów

Klasa podstawowa:

.. code-block:: c++

    template <typename T>
    class Vector 
    {
    public:
        /* constructors */

        const T& at(size_t index) const;
        void push_back(const T& value);
        
        /* ... etc. ... */
    }; 
 

Klasa, której zachowanie jest sparametryzowane wytycznymi:

.. code-block:: c++

    template 
    <
        typename T, 
        typename RangeCheckPolicy, 
        typename LockingPolicy = NullMutex
    >
    class Vector : public RangeCheckPolicy
    {
        std::vector<T> items_;
        using mutex_type = LockingPolicy;
        mutable mutex_type mtx_;

    public:
        using iterator = typename std::vector<T>::iterator;
        using const_iterator = typename std::vector<T>::const_iterator;

        /* ... etc. ... */
    };


Przykładowa klasa wytycznej sprawdzającej poprawność zakresu:

.. code-block:: c++

    class ThrowingRangeChecker
    {
    protected:
        ~ThrowingRangeChecker() = default;

        void check_range(size_t index, size_t size) const
        {
            if (index >= size)
                throw std::out_of_range("Index out of range...");
        }
    };


Inna implementacja wytycznej kontrolującej zakres indeksów:

.. code-block:: c++

    class LoggingErrorRangeChecker
    {
    public:
        void set_log_file(std::ostream& log_file)
        {
            log_ = &log_file;
        }

    protected:
        ~LoggingErrorRangeChecker() = default;

        void check_range(size_t index, size_t size) const
        {
            if ((index >= size) && (log_ != nullptr))
                *log_ << "Error: Index out of range. Index="
                    << index << "; Size=" << size << std::endl;
        }

    private:
        std::ostream* log_{};
    };


Implementacja metody ``at()`` wektora z uwzględnieniem wytycznych:

.. code-block:: c++
    
    template 
    <
        typename T,
        typename RangeCheckPolicy,
        typename LockingPolicy
    >
    const T& Vector<T, RangeCheckPolicy, LockingPolicy>::at(size_t index) const
    {
        std::lock_guard<mutex_type> lk{mtx_};        
        
        RangeCheckPolicy::check_range(index, size());

        return (index < items_.size()) ? items_[index] : items_.back();
    } 

Kod klienta:

.. code-block:: c++

    Vector<int, ThrowingRangeChecker, StdLock> vec = {1, 2, 3};
    vec.push_back(4);

    try
    {
        auto value = vec.at(8);
    }
    catch(const std::out_of_range& e)
    {
        std::cout << e.what() << std::endl;
    }

 
Klasa cech iteratorów
---------------------

Biblioteka standardowa C++ często wykorzystuje technikę cech i wytycznych.
Jedną z bardziej przydatnych klas cech w bibliotece standardowej, jest klasa cech iteratorów.

.. code-block:: c++

    template <class Iterator> 
    struct iterator_traits 
    { 
        typedef typename Iterator::iterator_category iterator_category; 
        typedef typename Iterator::value_type value_type; 
        typedef typename Iterator::difference_type difference_type; 
        typedef typename Iterator::pointer pointer; 
        typedef typename Iterator::reference reference; 
    }; 

    template <class T> 
    struct iterator_traits<T*> 
    { 
        typedef random_access_iterator_tag iterator_category; 
        typedef T value_type; 
        typedef ptrdiff_t difference_type; 
        typedef T* pointer; 
        typedef T& reference; 
    };


Przykład wykorzystania klasy cech iteratorów:

.. code-block:: c++

    #include <iterator>
    template <typename Iter> 
    inline 
    typename std::iterator_traits<Iter>::value_type accum (Iter start, Iter end) 
    {
        typedef typename std::iterator_traits<Iter>::value_type VT; 
        VT total = VT(); // assume T() actually creates a zero value 
        while (start != end) 
        { 
            total += *start; 
            ++start; 
        } 
        return total; 
    } 

.. include:: tag-dispatch.rst