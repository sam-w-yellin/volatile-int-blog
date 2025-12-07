---
title: "Can you survive the C++ Auto Type Gauntlet?"
date: 2025-12-05T12:00:00-08:00
draft: false
tags: ["C++","type","deduction", "auto", "quiz"]
---

One of the most iconic C++ features is the language's ability to deduce types with the `auto` keyword. In this post, I'll give a bunch of example code snippits. Your job is to assess what will be deduced for `v` in each snippit. Determine:
1. The type
2. If it is a reference or not
3. CV qualifiers

Some of these may not even compile, so "this won't work" is a totally valid answer.

Each section increases in difficulty. Good luck!

# The Gauntlet

## Basics
Basic assignments and deduction from constants and straightforward types.

{{< quiz_question
      answer="**Type:** `int`"
      explanation="**Explanation:** Straightforward type deduction from an integer constant."
>}}
auto v = 5;
{{< /quiz_question >}}

{{< quiz_question
      answer="**Type:** `double`"
      explanation="**Explanation:** Notably different than integers. Floating points default to the larger `double` instead of `float`."
>}}
auto v = 0.1;
{{< /quiz_question >}}

{{< quiz_question
      answer="**Type:** `int`"
      explanation="**Explanation:** Integer type derived from the assigned-from variable."
>}}
int x;
auto v = x;
{{< /quiz_question >}}

{{< quiz_question
      answer="**Type:** Fails to compile."
      explanation="**Explanation:** All types in an expression defined with `auto` have to be the same."
>}}
auto v = 5, w = 0.1;
{{< /quiz_question >}}

{{< quiz_question
      answer="**Type:** int"
      explanation="**Explanation:** Unless, of course, you are using structured binding."
>}}
std::pair<int, float> x {1, 2.0};
auto [v, w] = x;
{{< /quiz_question >}}

{{< quiz_question
      answer="**Type:** `int*`"
      explanation="**Explanation:** Auto will deduce pointers."
>}}
int x;
auto v = &x;
{{< /quiz_question >}}

{{< quiz_question
      answer="**Type:** `std::nullptr_t`"
      explanation="**Explanation:** `nullptr` has its own type."
>}}
auto v = nullptr;
{{< /quiz_question >}}

{{< quiz_question
      answer="**Type:** `std::initializer_list<int>`"
      explanation="**Explanation:** It might seem like this should create a container, but it won't!"
>}}
auto v = { 1, 2, 3 };
{{< /quiz_question >}}


{{< quiz_question
      answer="**Type:** `int*`"
      explanation="**Explanation:** C-style arrays decay to a pointer. The decay happens before `auto` is evaluated."
>}}
int x[5];
auto v = x;
{{< /quiz_question >}}

{{< quiz_question
      answer="**Type:** `int (*) (int)`"
      explanation="**Explanation:**  `auto` can deduce function pointers."
>}}
int foo(int x) {
    return x;
}
auto v = foo;
{{< /quiz_question >}}

## Intermediate
Exploring how references and CV-qualifiers are handled.
{{< quiz_question
      answer="**Type:** `int`"
      explanation="**Explanation:** `auto` drops CV qualifiers."
>}}
volatile const int x = 1;
auto v = x;
{{< /quiz_question >}}

{{< quiz_question
      answer="**Type:** `volatile const int*`"
      explanation="**Explanation:** Unless, of course, they are applied to a pointed-to or reference type"
>}}
volatile const int x = 1;
auto v = &x;
{{< /quiz_question >}}

{{< quiz_question
      answer="**Type:** `int`"
      explanation="**Explanation:** `auto` will never deduce a reference on its own."
>}}
int x;
int& y = x;
auto v = y;
{{< /quiz_question >}}

{{< quiz_question
      answer="**Type:** `int&`"
      explanation="**Explanation:** References are deduced if explicitly marked as a reference."
>}}
int x;
auto& v = x;
{{< /quiz_question >}}

{{< quiz_question
      answer="**Type:** `int (&) [5]`"
      explanation="**Explanation:** When binding arrays to references, they don't decay. So, `auto` deduces the actual array type."
>}}
int x[5];
auto& v = x;
{{< /quiz_question >}}

{{< quiz_question
      answer="**Type:** `int (*) (int)`"
      explanation="**Explanation:**  Remember - CV qualifiers are thrown away during function resolution!"
>}}
int foo(const int x) {
    return x;
}
auto v = foo;
{{< /quiz_question >}}

## Advanced
Forwarding references, `decltype(auto)`, and lambdas.

{{< quiz_question
      answer="**Type:** `int&`"
      explanation="**Explanation:** A forwarding reference like `auto&&` can bind to lvalue or rvalue expressions. We get an lvalue reference because `x` is an lvalue."
>}}
int x;
auto&& v = x;
{{< /quiz_question >}}

{{< quiz_question
      answer="**Type:** `int&&`"
      explanation="**Explanation:** `x()` returns a prvalue, and prvalues assigned to forwarding references result in an rvalue reference."
>}}
auto x = [] () -> int { 
    return 1;
};
auto&& v = x();
{{< /quiz_question >}}

{{< quiz_question
      answer="**Type:** `int&`"
      explanation="**Explanation:** This time `x()` returns an lvalue, and lvalues assigned to forwarding references result in an lvalue reference."
>}}
int x;
auto y = [&] () -> int& { 
    return x;
};
auto&& v = y();
{{< /quiz_question >}}

{{< quiz_question
      answer="**Type:** `int&`"
      explanation="**Explanation:** `(x)` is an expression. `decltype(expression)` yields a reference when the expression is an lvalue."
>}}
int x;
decltype(auto) v = (x);
{{< /quiz_question >}}

{{< quiz_question
      answer="**Type:** `Foo&&`"
      explanation="**Explanation:** prvalues like Foo{} will bind to a rvalue reference."
>}}
struct Foo {};
auto&& v = Foo{};
{{< /quiz_question >}}

{{< quiz_question
      answer="**Type:** `Foo`"
      explanation="**Explanation:** For any prvalue expression `e`, `decltype(e)` evaluates to the *type* of e."
>}}
struct Foo {};
decltype(auto) v = Foo{};
{{< /quiz_question >}}

{{< quiz_question
      answer="**Type:** `int&&`"
      explanation="**Explanation:** For any xvalue expression `e`, `decltype(e)` evalutes to a rvalue reference to the type of e."
>}}
int x;
decltype(auto) v = std::move(x);
{{< /quiz_question >}}

{{< quiz_question
      answer="**Type:** Fails to compile."
      explanation="**Explanation:**  Function id-expressions do not decay to pointers when evaluating with `decltype`!"
>}}
int foo(int x) {
    return x;
}
decltype(auto) v = foo;
{{< /quiz_question >}}

{{< quiz_question
      answer="**Type:** `int (&) (int)`"
      explanation="**Explanation:**  Parenthesized function symbol expressions are deduced as a *reference* to the function."
>}}
int foo(int x) {
    return x;
}
decltype(auto) v = (foo);
{{< /quiz_question >}}

## Oof
Abandon all hope, ye who attempt to deduce the types of lambda captures in expressions with `decltype(auto)`. 

{{< quiz_question
      answer="**Type:** `int`"
      explanation="**Explanation:** `decltype(auto)` always deduces id-expressions strictly with the type of the target symbol. Even though `x` is captured by reference, it is not deduced as a reference."
>}}
int x;
[&] {
    decltype(auto) v = x;
}
{{< /quiz_question >}}

{{< quiz_question
      answer="**Type:** `int&`"
      explanation="**Explanation:** However, lvalues in parenthesized expressions are deduced as references via `decltype(auto)`."
>}}
int x;
[&] {
    decltype(auto) v = (x);
}
{{< /quiz_question >}}

{{< quiz_question
      answer="**Type:** `int`"
      explanation="**Explanation:** lvalue id-expressions are deduced strictly as their type via `decltype(auto)`."
>}}
int x;
[=] {
    decltype(auto) v = x;
}
{{< /quiz_question >}}

{{< quiz_question
      answer="**Type:** `const int&`"
      explanation="**Explanation:** Captures by value are `const`, so the type of `x` is `const int`. lvalues in a parenthesized expression are deduced as references - so we end with `const int&`."
>}}
int x;
[=] {
    decltype(auto) v = (x);
}
{{< /quiz_question >}}

{{< quiz_question
      answer="**Type:** `int&`"
      explanation="**Explanation:** Unless, of course, we make the lambda `mutable`. Then the captures are non-`const`."
>}}
int x;
[=] mutable {
    decltype(auto) v = (x);
}
{{< /quiz_question >}}

{{< quiz_question
      answer="**Type:** Fails to compile."
      explanation="**Explanation:** Lambdas cannot capture references. When `y` is captured, it captures the referred-to value `x`. Because captures are const, the captured value ends up as `const int`. However, `decltype(auto)` sees the symbol `y`'s declaration as an `int&` and deduces the type of `v` as `int&`. Compilation fails on discarding the `const` qualifier when trying to assign a `const int` to an `int&`."
>}}
int x;
int& y = x;
[=] () {
    decltype(auto) v = y;
}();
{{< /quiz_question >}}

# Conclusion
You can check any of these by using this [GodBolt link](https://godbolt.org/z/f83vKh8j8). Shoutout to this [Stack Overflow thread](https://stackoverflow.com/questions/81870/is-it-possible-to-print-the-name-of-a-variables-type-in-standard-c/56766138#56766138) which provided the code snippit to print human readable names for a variable.

Do you have any interesting examples to share? Reach out at sam@volatileint.dev and let me know! If you found this article interesting, consider subscribing to the [newsletter](https://volatileint.dev/newsletter) to hear about new posts!
