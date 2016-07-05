---
layout: post
title: Empty C++ COW strings
---

GCC implementation of standard C++ library used to have Copy-On-Write std::string implementation. It had the benefit of making string copies fast but it also made strings hard to use in multi-threaded programs. Then C++11 happened and prohibited(?) COW implementations and GCC5 has another implementation of std::string with optimizations for small strings. Yet GCC 4.x still uses COW strings even in C++11 mode for binary compatibility reasons.

Over the years I trained myself to look for bugs caused by the abuse strings as in the example below:

```cpp
std::string a("A");
std::string a2 = a;

printf("'%s' '%s'\n", a.c_str(), a2.c_str());
*(char *)a2.c_str() = 'B';
printf("'%s' '%s'\n", a.c_str(), a2.c_str());
```

As the result changing the value of one string it silently "changed" the value of some other string because they all point to the same memory structure:

```
'A' 'A'
'B' 'B'
```

GCC5 fixes that implicitly:

```
'A' 'A'
'A' 'B'
```

This issue was quite easy to find during code reviews because the strings were close to each other in the code and raw pointers and casts are screaming "here be bugs".

Recently I debugged another issue with COW strings caused by legal but slightly incorrect C++ code. Turns out that GCC C++ library implementation optimized empty strings as well: the default constructor of std::string creates strings that all point to the same empty string singleton. It's easy to mess up the empty string using the first technique. But it's also easy to do with a perfectly legal operator[] but forgetting to perform bound checks. In all other cases operator[] would create a copy of the shared memory structure but for some reason it silently skips doing a copy for empty strings instead of doing it, raising exception or aborting the program. Good luck finding the single erroneous assignment million lines of code away from where it shows on the surface!

```cpp
std::string empty;
// in a code far, far away...
std::string empty2;

printf("'%s' '%s'\n", empty.c_str(), empty2.c_str());
empty2[0] = '?';
printf("'%s' '%s'\n", empty.c_str(), empty2.c_str());
std::cout << "'" << empty << "' '" << empty2 << "'\n";
```

C++ code that checks the size of the string will continue to work (like std::cout) but C-style strings and I/O will look really strange:

```
'' ''
'?' '?'
'' ''
```

Oh by the way, GCC5 implicitly limits the scope of damage in this case as well:

```
'' ''
'' '?'
'' ''
```


All this confirms that it's easy to shoot yourself in the foot with C++ but it also shows how much better a C++11-compliant implementation is compared to the "old" C++.
