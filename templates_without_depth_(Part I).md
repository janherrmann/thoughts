# Templates without depth (Part I)

Copyright Jan Herrmann, 2017

## Introduction

### Sum of the first n natural numbers

Look at the following simple example on doing template metaprogramming:

    template <size_t N>
    struct sum
    {
        static const size_t value = N + sum<N-1>::value;
    };
    template <>
    struct sum<0>
    {
        static const size_t value 0;
    };

This toy example implements a compile time function which calculates the sum
of all positive numbers from 0 to n. But when we try to do this calculation
with n=1000

    int main() {
        cout << sum<1000>::value << endl;
    }

the compiler will probably complain about reaching the
[instantiation depth](http://coliru.stacked-crooked.com/a/56cebb10f60d1661).
The solution is not to increase the maximum but a simple 
[static assert](http://coliru.stacked-crooked.com/a/96af026bddaf9315):

    int main() {
        static_assert(sum<500>::value!=0, "");
        cout << sum<1000>::value << endl;
    }

Now the question is why this assert solves our problem. The simple answer is
memoization. The static_assert forces the compiler to instantiate sum<500>,
compute its value and memorize it. Later during computing `sum<1000>::value`
the recursion is stoped when `sum<500>::value` is needed and the memorized
result is used. Of course static_assert is not the only solution and it is
very unflexible, too. But there are other mechanisms to create a specific
order of template instantiations with the aim of reducing instantiation depth:

* template arguments are instantiated before the template: 
    in `using result = second_t<first_t<argument>>` `first_t` is evaluated before
    `second_t`
* template arguments are instantiated from left to right:
    `using result = third_t<first_t<argument>, second_t<argument>>` the order is
    `first_t`, `second_t` and `third_t`
* with `std::make_index_sequence` there exists a facility for generating numbers
    for templates, as long as we know how many templates to instantiate we can
    order them and create a map from size_t to our desired result

Now our example gets a little bit more lengthy. First we rename `sum` to
`sum_impl`:

    template <size_t N>
    struct sum_impl
    {
        static const size_t value = N + sum_impl<N-1>::value;
    };
    template <>
    struct sum_impl<0>
    {
        static const size_t value=0;
    };

As a second step we create a maping from an index sequence to our `sum_impl`:

    template<class Sequence>
    struct sum_map;

    template<size_t... Ints>
    struct sum_map<index_sequence<Ints...>>
    {
        using type = index_sequence<sum_impl<Ints>::value...>;
    };

With this map we can create all sums from 0 to n with
`using result = sum_map<make_index_sequence<n+1>>>`.

As a third step we need to get the last element out of this
`index_sequence`. This is a non trivial task, at least if we need
a diffrent kind of sequence e.g. a type sequence. But an easy solution
can be implemented for an 2 element argument pack:

    template<class, class Second>
    struct second
    {
        using type=Second;
    };

The idea behind this is to pass the `sum_map` as the first argument and 
the `sum_impl` we are interested in as the second argument. Arguments are
instantiated from left to right, first we create all `sum_impl` elements
we need inside of `sum_map` and as a second step we lookup the `sum_impl` 
we are interested in:

    template<size_t N>
    using sum = typename second<typename sum_map<make_index_sequence<N>>::type,sum_impl<N>>::type;

As we can [see](http://coliru.stacked-crooked.com/a/a6bddf19506dbdaa) it is 
possible to compute `sum<10000>::value` with an instantiation depth of 15 . 
With a perfect `make_index_sequence` it [works with an instantiation depth of 4](http://coliru.stacked-crooked.com/a/0656d99b10153515). 

Now the question is wether there are other (more important) metafunctions
which can take advantage from memoization, have a known recursion depth
and can be used iteratively with an index sequence. And indeed we can
implement indexed access and fold this way.

### Indexed access to type sequences

Indexed access for type lists has been implemented by Andrei Alexandrescu in
Modern C++ Design. He has implemented TypeAt the following way:

    template <class TList, unsigned int index> struct TypeAt;

    template <class Head, class Tail>
    struct TypeAt<Typelist<Head, Tail>, 0>
    {
        typedef Head Result;
    };

    template <class Head, class Tail, unsigned int i>
    struct TypeAt<Typelist<Head, Tail>, i>
    {
        typedef typename TypeAt<Tail, i - 1>::Result Result;
    };

With C++11 and variadic templates it [becomes](http://coliru.stacked-crooked.com/a/72f403953dd30231):

    template<class... T>
    struct type_sequence
    {};

    template<class Sequence, size_t index>
    struct type_at;

    template<class Head, class... Tail, size_t Index>
    struct type_at<type_sequence<Head, Tail...>, Index>
    {
        using type = typename type_at<type_sequence<Tail...>, Index-1>::type;
    };

    template<class Head, class... Tail>
    struct type_at<type_sequence<Head, Tail...>,0>
    {
        using type = Head;
    };

    template<class TypeSequence, size_t Index>
    using type_at_t = typename type_at<TypeSequence, Index>::type;

You can find lots of implementations of this algorithm
(for example [from Peter Dimov](http://pdimov.com/cpp2/simple_cxx11_metaprogramming_2.html))
and all these implementations are inefficient. To see and remove
this inefficiency we nedd to transform the code: we add the
following metafunctions and implement type_at_t in terms of them ([full implementation](http://coliru.stacked-crooked.com/a/b692a9cf6bb23a3e)):

* `head` (trivial therefor not shown here)
* `tail` (trivial)
* `remove_first_n`

```cpp

    template<class Sequence, size_t N>
    struct remove_first_n
    {
        using type = typename remove_first_n<typename tail<Sequence>::type, N-1>::type;
    };

    template<class Sequence>
    struct remove_first_n<Sequence, 0>
    {
        using type = Sequence;
    };

    template<class Sequence, size_t Index>
    using type_at_t = typename head<typename remove_first_n<Sequence, Index>::type>::type;
    
```

As we see the structure of the algorithm has been moved to `remove_first_n`.
Now we access all elements of a sequence (highly simplified):

    remove_first_n<type_sequence<int,float,double>,0>::type         :=
        type_sequence<int,float,double>

    remove_first_n<type_sequence<int,float,double>,1>::type         :=
        remove_first_n<
            tail<type_sequence<int,float,double>>::type,0>::type    :=
        remove_first_n<type_sequence<float,double>,0>::type         :=
        type_sequence<float,double>

    remove_first_n<type_sequence<int,float,double>,2>::type         :=
        remove_first_n<
            tail<type_sequence<int,float,double>>::type,1>::type    :=
        remove_first_n<type_sequence<float,double>,1>::type         :=
        remove_first_n<
            tail<type_sequence<float,double>>::type,0>::type        :=
        remove_first_n<type_sequence<double>,0>::type               :=
        type_sequence<double>

We have 6 instantiations of remove_first_n. With a [slightly
modified source](http://coliru.stacked-crooked.com/a/dd9bfd7623350d79)
and clang we can omit erros on every instantiation and count them.
To access all types of a n element type sequence get (n*n+n)/2 
instantiations of `remove_first_n`. The heard of this
algorithm is `typename remove_first_n<typename tail<Sequence>::type, N-1>::type`
and it is saying: *remove one element and then n-1 elements* but
we can change it to `typename tail<typename remove_first_n<Sequence, N-1>::type>::type`
which means *remove n-1 elements and then one element*. This looks
like its not important but [with the modified source](http://coliru.stacked-crooked.com/a/e0d58463b9e38229)
we have have only 3 errors wich means 3 instantiation (of `remove_first_n`):

    remove_first_n<type_sequence<int,float,double>,0>::type             :=
        type_sequence<int,float,double>

    remove_first_n<type_sequence<int,float,double>,1>::type             :=
        tail<remove_first_n<type_sequence<int,float,double>,0>::type>   :=
        tail<type_sequence<int,float,double>> :=
        type_sequence<float,double>

    remove_first_n<type_sequence<int,float,double>,2>::type             :=
        tail<remove_first_n<type_sequence<int,float,double>,1>::type>   :=
        tail<type_sequence<float,double>> :=
        type_sequence<float,double>

For every instantiation we lookup a previously computed one
and reduce the amount of computation. Now implementing
a safe `type_at` is trivial.

### Implementation of fold

Besides map, fold is the workhorse in functional programming and
consequently in template metaprogramming. As long as we know
which size our type sequence has, we know how many fold operations
we need. Furthermore we know in which order they have to be computed.
The simple formula is:

    fold(Sequence, State, Operation, Index) =
        Operation(
            fold(Sequence, State, Operation, Index-1),
            type_at(Sequence, Index)
        )
    fold(Sequence, State, Operation, 0) =
        Operation(State,type_at(Sequence, 0))

So we can implement fold the following way:

    template<
        class Sequence,
        class State,
        template<class,class> class Operation,
        size_t Index>
    struct fold_impl
    {
        using type = typename Operation<
            typename fold_impl<Sequence,State,Operation,Index-1>::type,
            typename type_at<Sequence, Index>::type
        >::type;
    };

and for the recursion base case:

    template<
        class Sequence,
        class State,
        template<class,class> class Operation
    >
    struct fold_impl<Sequence, State, Operation, 0>
    {
        using type = typename Operation<
            State,
            typename type_at<Sequence, 0>::type
        >::type;
    };

Like with `sum` we need a maping for fold:

    template<
        class Sequence, 
        class State, 
        template<class,class> class Operation, 
        class Index_Sequence
    >
    struct fold_map;

    template<
        class Sequence,
        class State,
        template<class,class> class Operation,
        size_t... Ints
    >
    struct fold_map<Sequence, State, Operation, index_sequence<Ints...>>
    {
        using type = type_sequence<typename fold_impl<Sequence, State, Operation, Ints>::type ...>;
    };

and with an additional `sequence_size` metafunction we can create our fold:

    template <
        class Sequence,
        class State,
        template<class,class> class Operation
    >
    using fold = typename second<
        typename fold_map<
            Sequence,
            State,
            Operation, 
            make_index_sequence<sequence_size<Sequence>::type::value>
        >::type,
        typename fold_impl<Sequence, State, Operation, sequence_size<Sequence>::type::value-1>::type
    >::type;

The [example](http://coliru.stacked-crooked.com/a/85800da881a6ba06) works with
an instantiation depth of 6. With larger type_sequences `std::make_index_sequence`
can use more space. Furthermore we don't need a safe version of `type_at` to
implement fold as we are iterating from the start over the sequence and access
all elements in the order they are computed. But of course there is room for
improvement.
