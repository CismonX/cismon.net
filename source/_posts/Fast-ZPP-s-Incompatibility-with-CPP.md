---
title: Fast ZPP's Incompatibility with C++
date: 2017-12-18 23:35:07
category: essay
tags:
  - PHP
  - Zend
---

Since PHP 7.0, a new Zend API was implemented for faster parameter parsing.

For example, if your function accepts an integer parameter `foo`, then the code may look like this.

```C
ZEND_PARSE_PARAMETERS_START(1, 1)
    Z_PARAM_LONG(foo)
ZEND_PARSE_PARAMETERS_END();
```

However, if your extension is written in C++, the compiler will complain and refuse to compile.

You'll get error message like:

```
error: invalid conversion from 'int' to 'zend_expected_type {aka _zend_expected_type}' [-fpermissive]
```

Confused? Let's take a look at the macro definition, which is located in zend_API.h

```C
#define ZEND_PARSE_PARAMETERS_START_EX(flags, min_num_args, max_num_args) do { \
        const int _flags = (flags); \
        int _min_num_args = (min_num_args); \
        int _max_num_args = (max_num_args); \
        int _num_args = EX_NUM_ARGS(); \
        int _i; \
        zval *_real_arg, *_arg = NULL; \
        zend_expected_type _expected_type = IS_UNDEF; \
        char *_error = NULL; \
        zend_bool _dummy; \
// Some more code...
```

We can see on line 8 Zend's trying to initialize an enum `zend_expected_type` with value 0, which is forbidden in C++. In C++, you should either explicitly cast it with `static_cast` or initialize using a corresponding enum value.

Fortunately the value 0 is defined in macro `IS_UNDEF` (why this??), you can just redefine it instead of `sed` zend_API.h in your config.m4 script.

Now your code may look like this.

```C
#undef IS_UNDEF
#define IS_UNDEF Z_EXPECTED_LONG // Which is zero
ZEND_PARSE_PARAMETERS_START(1, 1)
    Z_PARAM_LONG(foo)
ZEND_PARSE_PARAMETERS_END();
#undef IS_UNDEF
#define IS_UNDEF 0
```

Ugly, but your code will compile. Cheers :)

P.S. The latest PHP 7.2 still have this problem. Perhaps I should report this issue to the PHP internals guys.
