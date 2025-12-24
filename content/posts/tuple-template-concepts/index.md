---
title: "Applying Template Concepts to Tuples in C++ "
date: 2025-12-23T00:00:00-08:00
draft: false
tags: ["crunch", "C++", "template", "tuple", "concept"]
---

This article is going to walk through a few different ways to constrain the types within a tuple used as a template parameter. In C++, tuples are a collection of values of different types. You can access different elements at compile time via the `get` method. The very commonly used std::pair is a two-element tuple-like type.

While developing Crunch <insert link to the intro article>, I had quite a few use cases for applying template concepts to tuples and found some good - and not so good - ways to do it. Hopefully by the end of this article, you feel comfortable writing template concepts targeting `std::tuple` parameters!

For folks who want an at-a-glance solution and are less interested in the motivation and details, I'll quickly show what we're going to build up to. 

*When you need to inspect types or constexpr values of each element*:
```c++
template <typename Tuple>
concept ElementsAreIntegral =
    []<std::size_t... Is>(std::index_sequence<Is...>) {
        return (std::integral<
            std::remove_cvref_t<std::tuple_element_t<Is, Tuple>>
        > && ...);
    }(std::make_index_sequence<std::tuple_size_v<Tuple>>{});
```

*When you don't need to inspect each element*
```c++
template<typename Tuple>
concept ElementsSatisfyConcept = requires {
    std::apply(
        []<typename... Ts>(Ts&&...)
            requires (std::remove_cvref_t<YourConcept<Ts>> && ...) {},
        std::declval<Tuple>()
    );
};
```
Note: In practice, `std::apply` often produces noticeably worse diagnostics than index-sequence approaches, so you might opt for the first solution even if you don't need to access elements.

The rest of this article will provide the context on why that works, and motivate how you could arrive there yourself from the ground up.


# Setting the Stage
Take a template class which takes some tuple:

```c++
template <typename Tuple>
class Foo {
    Tuple my_tuple;
};
```

Going forward, I will omit the constraint that actually forces `Tuple` to be a `std::tuple` because it is a bit orthogonal to the topic of applying concepts to tuple *elements*. A simple one might look something like this:
```c++
template <typename T>
concept TupleLike =
    requires {
        typename std::tuple_size<T>::type;
    } &&
    requires(T t) {
        std::get<0>(t);
    };
```

Let's say that we want each element in `Tuple` to satisfy `std::integral` or compilation should fail. Something ergonomically similar to this:

```c++
template <typename Tuple>
requires ElementsAreIntegral<Tuple>
class Foo {
    Tuple my_tuple;
};
```

In this case, we don't really care what the value or type of each individual element in the tuple is - we just care that the type satisfies some constraint. The intuitive high-level approach to accomplishing this is:

1. Somehow "unpack" the tuple into each of its elements.
2. Apply the `std::integral` concept to each
3. Fail the concept if any do not satisfy `std::integral`.

First, let's look at a naive approach, and then we'll clean it up.

# The Manual Approach
In our first approach, we're going to tackle each of the above steps independently. I call this "manual" because it uses fewer language features and more boilerplate as compared to the next solutions we will look at. You should almost never write this in real code, but I want to demonstrate the mechanics with a minimal set of language features.

First off, we know that we need a `concept` called `ElementsAreIntegral`. 

```c++
template<typename Tuple>
concept ElementsAreIntegral = // ...something...
```
Our goal is to have `ElementsAreIntegral` evaluate to `true` when the elements meet `std::integral` and `false` when they do not.

First, we know that we ultimately are going to need to unpack this tuple and check the elements.  So we know we need something that expands out all the tuple elements, applies the target concept, and returns false if any are not satisfied:

```c++
// Note: this cannot be our final concept — it requires an index pack
template <typename Tuple, std::size_t... Is>
concept ElementsAreIntegralWithSequenceIdx =
    (std::integral<std::tuple_element_t<Is, Tuple>> && ...);
``` 

Its possible the the tuple was declared with reference or CV-qualified parameters, and your concept probably takes an unqualified, non-reference value. So you'll probably want to actually make sure we strip the reference & CV qualification from the tuple elements:
```c++
// Note: this cannot be our final concept — it requires an index pack
template <typename Tuple, std::size_t... Is>
concept ElementsAreIntegralWithSequenceIdx =
    (std::integral<std::remove_cvref_t<
        std::tuple_element_t<Is, Tuple>>
    > && ...);
``` 

This looks really close to what we want. We are assuming we somehow create a variadic template pack `Is` which represents the index of each element. Then, when unpacking, we use the index to get the tuple element then logically `AND` the result of each individual tuple element's concept check.

How do we get `Is`? The `STL` contains a few utility functions that combine to do  exactly what we need. First, we can use `std::tuple_size_v` to extract out the size of a the tuple, then we can pass the result as a template parameter to `std::make_index_sequence` to get the `Is` parameter pack.  This snippet does what we need for a given `Tuple` typename:

```c++
std::make_index_sequence<std::tuple_size_v<Tuple>>;
```

Now we have most of the logic in place. But we need to bridge the gap between the desired concept - which takes only the `Tuple` parameter - and this concept which takes in the tuple and a sequence of indices. Intuitively, it probably seems like this is the right sort of approach:

```c++
template <typename Tuple, std::size_t... Is>
concept ElementsAreIntegralImpl =
   (std::integral<std::remove_cvref_t<
        std::tuple_element_t<Is, Tuple>>
    > && ...);

template<typename Tuple>
concept ElementsAreIntegral =
    ElementsAreIntegralImpl<Tuple, std::make_index_sequence<std::tuple_size_v<Tuple>>>;
```

But there's a problem here. `Is` is a *parameter pack*. `std::make_index_sequence<std::tuple_size_v<Tuple>>` returns a *type* which has the parameter pack we want *inside*. Something like `std::index_sequence<0, 1, 2, ..., N>`. In order to get at that set of indices inside the sequence, we need to figure out a way to bind to that sequence. We can do this with a *partial template specialization*.

What we need is a template that is specialized on the `std::index_sequence` parameter pack. Something like:
```c++
template <std::size_t...Is>
struct sequence_extractor<std::index_sequence<Is...>> {};
```

Now, we have a mechanism to bind the results of `std::make_index_sequence` to a template parameter pack. We need to utilize that mechanism, while still returning either a boolean value to the top-level `ElementsAreIntegral` concept. We can is utilize a struct that extracts out the `std::index_sequence` as a parameter pack, and have that struct extend `std::bool_constant`, where the `value` of that boolean constant is predicated on the result of `ElementsAreIntegralImpl`. That's quite a mouthful, let's write it out step by step. 

First, we define our top-level concept predicated on the `::value` of a struct. We'll call that struct `SequenceHelper`. It is templated on the Tuple and the result of `std::make_index_sequence` - both parameters are required because we need to forward them onto the `ElementsAreIntegralImpl` concept.
```c++
template <typename Tuple>
concept ElementsAreIntegral =
    SequenceHelper<
        Tuple, 
        std::make_index_sequence<std::tuple_size_v<Tuple>>
    >::value
```

Now, we need to actually write our `SequenceHelper`. Remember, this thing's job is to bind a template parameter to the `std::index_sequence`'s parameter pack, and forward it to the `ElementsAreIntegralImpl` which actually checks if each element fulfills `std::integral`. The result of that concept should be stored in the struct's `::value` result. We can have the struct inherit from `bool_constant` so that it has a `::value` static data member. Something like this:

```c++
// base
template <typename Tuple, typename IndexSeq> 
struct SequenceHelper : std::false_type {};

// partial specialization
template <typename Tuple, std::size_t...Is>
struct SequenceHelper<Tuple, std::index_sequence<Is...>>
    : std::bool_constant<ElementsAreIntegralImpl<Tuple, Is...>> {};
```

See how we were able to *extract* the parameter pack from the `std::index_sequence` and pass it to `ElementsAreIntegralImpl`? We now can pull it all together for our final, working implementation:

```c++
template <typename Tuple, std::size_t... Is>
concept ElementsAreIntegralImpl =
    (std::integral<
        std::remove_cvref_t<std::tuple_element_t<Is, Tuple>>
    > && ...);

// base
template <typename Tuple, typename IndexSeq> 
struct SequenceHelper : std::false_type {};

// partial specialization
template <typename Tuple, std::size_t...Is>
struct SequenceHelper<Tuple, std::index_sequence<Is...>>
    : std::bool_constant<ElementsAreIntegralImpl<Tuple, Is...>> {};

template <typename Tuple>
concept ElementsAreIntegral =
    SequenceHelper<
        Tuple, 
        std::make_index_sequence<std::tuple_size_v<Tuple>>
    >::value;
```

This example can be seen in action at this [Godbolt link](https://godbolt.org/z/6fW61fqbs).

# Variadic Lambda Approach
Along with template concepts, C++20 introduced *generic template lambda* functions. These play in super interesting ways with template metaprogramming, and are very useful in simplifying the (in my view, very messy and hard to read) code in the "manual" approach. Here's the basic idea: instead of the "helper" structs for retrieving the parameter pack from inside the `std::index_sequence` and containing the result of the concept applied to each parameter, a template lambda can accept the sequence as a parameter. Then we can collapse both the "implementation" concept and the pack extraction into the top-level concept.

So, we want a concept that calls a templated lambda function which:
1. Takes a `std::index_sequence` as an input parameter
2. Is templated on that sequence's parameter pack
3. Calls `std::integral` on each element in the pack

If the result evaluates to `false` - meaning any of the elements is not integral, the concept evaluates to false.

```c++
template <typename Tuple>
concept ElementsAreIntegral = [] <std::size_t... Is>
    (std::index_sequence<Is...>) {
        return (std::integral<std::remove_cvref_t<
            std::tuple_element_t<Is, Tuple>>
        > && ...);
    }(std::make_index_sequence<std::tuple_size_v<Tuple>>{});
```

And that's it. All the sequence-extraction is now handled by our templated lambda, so all the proxy types go away. Feel free to play around with the [Godbolt example](https://godbolt.org/z/YTzjsxM81).

# std::apply Approach

You may have realized something about these previous examples - we don't *really* care what the type or value of any of the tuple elements is. All we really care about is if the element fulfills some concept. So, do we really need to actually create a `std::index_sequence` that can access each element individually? If you don't need to actually work with the type of each element, we can get even simpler than our previous example. When we don't really care about the elements we can take advantage of `std::apply` and variadic template lambdas together.

The intuition here is to utilize the same strategy of binding the tuple elements as a parameter pack to a lambda. Our goal is to create a lambda which is only well-formed if every element meets our desired concept, and then call it on the tuple inside a `requires` statement. This will fail if any element does not pass - without ever extracting an element explicitly. We'll start with a variadic, templated lambda. Remember - the code in a `concept`'s `requires` statement is not actually evaluated. All that matters is it is *well formed*. 

```c++
[]<typename... Ts>(Ts&&...) {}
```

We are going to bind the tuple elements as the parameter pack. We want this lambda's template to require each element fulfills `std::integral`.

```c++
[]<typename... Ts>(Ts&&...) requires
    (std::integral<std::remove_cvref_t<Ts>> && ...) {}
```

We now want to unpack each tuple element into a parameter pack, and call this lambda on those elements. Luckily, this is *exactly* what `std::apply` does.

```c++
std::apply(
  []<typename... Ts>(Ts&&...) requires
    (std::integral<std::remove_cvref_t<Ts>> && ...) {},
  // ...something...
);
```

The "something" we want is an instance of the tuple. Bringing it together as a concept we get:
```c++
template<typename Tuple>
concept ElementsAreIntegral = requires {
    std::apply(
        []<typename... Ts>(Ts&&...) requires
            (std::integral<std::remove_cvref_t<Ts>> && ...) {},
        std::declval<Tuple>()
    );
};
```
Note: The `std::declval` call is used to instantiate a concrete Tuple parameter in a concept's `requires` clause. 

This concept's `requires` clause is only well formed if the constrained lambda is a viable overload, which is only the case if each element in the parameter pack satisfies `std::integral`.

The [Godbolt as proof](https://godbolt.org/z/E1j9e4PE4). It's worth calling out that the error message through this approach is slightly more cryptic than in the others. We do eventually see the message about the failed concept, but it is preceded by lots of failures around `std::apply` as opposed to the extremely clear errors from the prior examples.

# Real-World Example
`Crunch`'s `CRUNCH_MESSAGE_FIELDS` macro creates a `constexpr` `get_fields` function for easy iteration over all the fields in a message. It returns a tuple where each element is `crunch::field::Field`. I needed to apply two different template concepts to the set of elements in the tuple:

1. Each field needed to satisfy the `ValidField` concept, which essentially ensures that the field is one of the Crunch field types (Scalar, Array, Map, Submessage).
2. The field's `FieldId` needs to be unique among all the fields in the message.

To accomplish this, I utilized both the lambda-based approaches. The [first case](https://github.com/sam-w-yellin/crunch/blob/24f25f48600b88ea0039ae87b74a35fcda471b1f/include/crunch/messages/crunch_messages.hpp#L66) was a great candidate for the `std::apply` approach - the specifics of each type was not important, just that it satisfies a particular concept.

```c++
template <typename Tuple>
concept tuple_members_are_valid_fields = requires {
    std::apply([]<typename... Ts>(Ts&&...)
                   requires(ValidField<std::remove_cvref_t<Ts>> && ...)
               {},
               std::declval<Tuple>());
};
```

In the [second case](https://github.com/sam-w-yellin/crunch/blob/24f25f48600b88ea0039ae87b74a35fcda471b1f/include/crunch/messages/crunch_messages.hpp#L86) we actually needed to inspect a value inside of each tuple element - the `FieldId` - so I used the `std::make_index_sequence` approach.

```c++
template <typename Tuple>
concept has_unique_field_ids =
    []<std::size_t... Is>(std::index_sequence<Is...>) {
        return !has_duplicates<std::remove_cvref_t<
            std::tuple_element_t<Is, Tuple>>::field_id...>::value;
    }(std::make_index_sequence<std::tuple_size_v<Tuple>>{});
```

I'll briefly touch on the implementation of `has_duplicates` because it's rather interesting, although not strictly related to the rest of this deep dive. It's implemented like this (only the partial specialization is shown):
```c++
template <FieldId Head, FieldId... Tail>
struct has_duplicates<Head, Tail...> {
    static constexpr bool value =
        ((Head == Tail) || ...) || has_duplicates<Tail...>::value;
};
```

This has the really neat behavior of expanding out an equality check between all elements. First, we expand out comparing the `Head` element against each other element in the tuple. Then, when we pass in `Tail...` recursively to `has_duplicates`, the first element in `Tail` becomes the next head, and the comparisons continue. At the end, we have checked each element against each other. A quick walkthrough with a concrete set of `FieldId`s `{1, 2, 3}`:

1. Head == 1, Tail == {2, 3}: 1 == 2 || 1 == 3. Then, OR with the result of call `has_duplicates<{2, 3}>`
2. Head == 2, Tail == {3}: 2 == 3.
3. Combine: 1 == 2 || 1 == 3 || 2 == 3

# Conclusion
Template metaprogramming can be hard to wrap your head around. Moreso than other C++ programming tasks, it can feel like the language is "getting in the way". But with enough experience you'll start to be able to break down your desired outcome into achievable steps and compose concepts like the ones we saw today. Hopefully some of these patterns are useful in your own code. 

If you’d like to discuss this pattern or how it might apply to your system, feel free to reach out: sam@volatileint.dev

If you found this article valuable, consider subscribing to the [newsletter](https://volatileint.dev/newsletter) to hear about new posts!