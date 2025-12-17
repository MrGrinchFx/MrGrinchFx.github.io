---
title: lvalues, rvalues, and xvalues. What do they do?
published: 2025-12-04
description: 'lvalues, rvalues, and more recently xvalues have been used in modern C++ to allow for move semantics in C++. This is a log of what I learned about this C++ concept.'
image: ''
tags: ['C++', 'programming']
category: 'Programming'
draft: false 
lang: 'en'
---
# What is an lvalue and an rvalue?

lvalue stands for "left" value, and rvalue stands for "right" value. This naming convention arises from the typical order of assignment operations such as in the following code example:

:::note
We will see why I used quotations! 
:::

```cpp
int x = 10;
```

Here the variable ```x``` is an lvalue and the value ```10``` is an rvalue. Ok so the naming convention makes sense for now.

However, as you know, you can now make ```x``` assignable to another variable via:

```cpp
int y = x;
```
So what's going on here? I thought ```x``` was an lvalue? Why is it able to be on the right side? Two answers for that:

```x``` is an lvalue.

lvalues are not necessarily on the left side of the operator.

This is interesting, so can I have rvalues on the left side?

No. [^1]
 [^1]:There are ways of having rvalues on the left side, but they semantically don't make sense and there are no benefits. Here's a video from Cppcon.
Let's look at an example:

```cpp
int x = 0;
10 = x;
```

Just from looking at the example, you can already see that something doesn't look right. We are literally assigning the value stored in x to the literal 10. Where is it being stored to if 10 is just a temporary value? In essence, you can't have an rvalue on the left side of the operator.

Looking at this example, we now have to modify our definition of what it means to be an lvalue and an rvalue. An lvalue more accurately is a value that is identifiable, meaning it has a location in the process memory that can be referred to. rvalues are instead temporary values that are not identifiable (or at least, do not have a stable address we should care about).

Why is this important?
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

In this example, we have a ```printName()``` function, and in this function we take in a string reference to print. In the first call to the function, we pass in the ```fullName``` string which is created from ```firstName + lastName.``` This will work as expected. But when we do the second call, where we do the ```firstName + lastName ```string concatenation within the function call, the compiler would raise an error. Why does this happen?

Let's have a look at the stack trace:
```
wrong-rvalue.cpp: In function ‘int main()’:
wrong-rvalue.cpp:11:23: error: cannot bind non-const lvalue reference of type ‘std::string&’ {aka ‘std::__cxx11::basic_string<char>&’} to an rvalue of type ‘std::__cxx11::basic_string<char>’
   11 |    printName(firstName + lastName); // Will cause a compiler Error
      |              ~~~~~~~~~~^~~~~~~~~~
wrong-rvalue.cpp:3:29: note: initializing argument 1 of ‘void printName(std::string&)’
    3 | void printName(std::string& name) { std::cout << name << std::endl; }
      |                 ~~~~~~~~~~~~^~~~
```

The reason is because when the string concatenation occurs, the resulting object is a temporary object and is considered an rvalue. And from what we know about the function, it only accepts string references (specifically lvalue references). The first call to ```printName()``` will work fine because we have initialized ```fullName``` in the previous line and it is considered an lvalue. This will then be accepted into the function with no problem.

But what if I wanted to have my ```printName()``` accept both argument types?

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

With this change, we are now able to compile fine, because const references are able to accept both lvalues as well as rvalues.

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
Suppose we had two function overloads with one accepting ```const std::string&``` and the other ```std::string&&```. If we were to call using an rvalue as an input, even though rvalues are compatible for both function definitions, the function accepting std::string&& would be preferred.
:::

Knowing this, this leads into what xvalues are, and how they play into the scheme of move semantics.

When we say data is "moved", it generally means that we don't copy the contents of memory to another piece of memory. Rather, we transfer the ownership of the memory block to a different variable, and the old variable's ties to the memory are cut. This avoids having to copy over variably large values in memory unnecessarily.

It is important to note that for primitive types like int or double, "moving" is effectively the same as "copying" because the data is so small. Move semantics really shine when dealing with large objects that manage their own resources, like ```std::vector``` or ```std::string```.

For example, if we have an lvalue to lvalue situation (i.e., ```std::vector<int> b = a;```), this will introduce a copy operation. If ```a``` contains millions of entries, we perform millions of copies. We wouldn't want to copy that data unnecessarily if we don't plan to use ```a``` anymore.

What needs to be done is to allow the lvalue a to become moveable so that we can transfer its internal data pointer to b. This is achieved via the ```std::move()``` method defined in the STL library introduced in C++11.

What ```std::move()``` does under the hood is that it converts an lvalue into an xvalue. xvalues are values that can be moved AND have a location in memory. By doing this, when we call for the assignment of one variable with an xvalue variable, we will move the contents of the xvalue into the memory address of the lvalue.

This means the original xvalue is left in a valid but unspecified state (usually empty), while the new variable takes ownership of the data. Inherently, one could say that an xvalue is both an rvalue (because it can be moved) and an lvalue (because it has an identity) at the same time.

:::note 
Officially, the C++ standard defines rvalues as "prvalues" or pure rvalues, while rvalues and xvalues are categorized as "rvalues". lvalues and xvalues are both designated as "glvalues", which stands for "generalized lvalue".
:::

So what we have covered so far is what these 5 terms mean in the grand scheme of modern C++ design. I may in the future go over how classes implement their own move semantics, and what considerations need to be taken in order to ensure correct behaviour.

