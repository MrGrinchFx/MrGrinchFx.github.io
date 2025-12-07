---
title: lvalues, rvalues, and xvalues. What do they do?
published: 2025-12-04
description: 'lvalues, rvalues, and more recently xvalues have been used in modern C++ to allow for move semantics in C++. This is a log of what I learned about this C++ concept.'
image: ''
tags: ['C++', 'programming']
category: 'Programming'
draft: true 
lang: 'en'
---

# What is an lvalue and an rvalue?

lvalue stands for "left" value, and rvalue stands for "right" value. This naming convention
arises from the typical order of assignment operations such as in the following code example:

:::note
We will see why I used quotations!
:::

```cpp
int x = 10;
```

Here the variable `x` is an lvalue and the value `10` is an rvalue. Ok so the naming convention
makes sense for now.

However, as you know, you can now make `x` assignable to another variable via:

```cpp
int y = x;
```

So what are what's going on here? I thought `x` was an lvalue? Why is it able to be on the right
side? Two answers for that:
1. `x` is an lvalue.
2. lvalues are not necessarily on the left side of operator.

This is interesting, so can I have rvalues on the left side?

No. [^1]

[^1]: There are ways of having rvalues on the left side, but they semantically don't make sense and there are no benefits. [Here's a video from Cppcon](https://www.youtube.com/watch?v=hkyZ8L343cU).

Let's look at an example:

```cpp
int x = 0;
10 = x;
```

Just from looking at the example, you can already see that something doesn't look right. We are
literally assigning the value 10 the value stored in the `x` variable. Where is it being stored to
if 10 is just a temporary value? In essence, you can't have an rvalue on the left side of the
operator.


Looking at this example, we now have to modify our definition of what it means to be an lvalue and
an rvalue. An lvalue more accurately is a value that is identifiable, meaning it has a location in
the process memory that can be referred to. rvalues are instead temporary values that are not
identifiable, meaning they don't have a stored location in memory.

## Why is this important?

You may ask yourself, why is this important?

Short answer: IT'S REALLY IMPORTANT.

Long answer: We need it for move semantics, a topic I will cover a little later to explain xvalues.

But here's an example:

```cpp 
#include <iostream>

void printName(std::string& name) { std::cout << name << std::endl; }

int main() {
  std::string firstName = "Harry ";
  std::string lastName = "Potter";

  std::string fullName = firstName + lastName;
  printName(fullName); // Will work fine
  printName(firstName + lastName); // Will cause a compiler Error
}
```

In this example, we have a `printName()` function, and in this function we take in string reference
to print. In the first call to the function, we pass in the `fullName` string which is created from
`firstName + lastName`. This will work as expected. But when we do the second call, where we do the 
`firstName + lastName` string concatenation within the function call, the compiler would raise an
error. Why does this happen? 

Let's have a look at the stack trace:
```
wrong-rvalue.cpp: In function ‘int main()’:
wrong-rvalue.cpp:11:23: error: cannot bind non-const lvalue reference of type ‘std::string&’ {aka ‘std::__cxx11::basic_string<char>&’} to an rvalue of type ‘std::__cxx11::basic_string<char>’
   11 |   printName(firstName + lastName); // Will cause a compiler Error
      |             ~~~~~~~~~~^~~~~~~~~~
wrong-rvalue.cpp:3:29: note: initializing argument 1 of ‘void printName(std::string&)’
    3 | void printName(std::string& name) { std::cout << name << std::endl; }
      |                ~~~~~~~~~~~~~^~~~
```

The reason is because when the string concatenation occurs, the resulting object is a temporary
object and is considered an rvalue. And from what we know about the function, it only accepts
string references. The first call to `printName()` will work fine because we have intialized
`fullName` in the previous line and it is considered an lvalue. This will then be accepted into the
function with no problem.

But what if I wanted to have my `printName()` accept both argument types?

With C++, you can do that with one simple trick:
```cpp
#include <iostream>

void printName(const std::string& name) { std::cout << name << std::endl; } //added a const keyword

int main() {
  std::string firstName = "Harry ";
  std::string lastName = "Potter";

  std::string fullName = firstName + lastName;
  printName(fullName); // Will work fine
  printName(firstName + lastName); // Will work fine now
}
```

With this change, we are now able to compile fine, because with a const reference's are able to
accept both lvalue as well as rvalues.

What if I wanted to only target rvalues?

You can do that too:
```cpp
#include <iostream>

void printName(std::string&& name) { std::cout << name << std::endl; } //added two &

int main() {
  std::string firstName = "Harry ";
  std::string lastName = "Potter";

  std::string fullName = firstName + lastName;
  printName(fullName); // Compiler Error 
  printName(firstName + lastName); // Will work fine 
}
```

Now the roles are reversed, where the lvalue input will throw a compiler error.

:::note
If there we had two function overloads with one accepting ```const std::string&``` and the other
```std::string&&```, if we were to call using an rvalue as an input, even though rvalues are compatible
for both function definitions, the function accepting ```std::string&&``` would be preferred.
:::


