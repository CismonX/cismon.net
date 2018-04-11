---
title: '''Pedantic'' Copy-on-Write Inspection in PHP 7.2'
date: 2018-04-11 09:53:17
tags:
  - PHP
  - Zend
category: essay
---
## 1. Copy-on-Write mechanism in PHP

PHP beginners may read something like this in books when learning the basics of PHP.

```PHP
$foo = ['foo' => 'baz']; // ['foo' => 'baz']: refcount = 1
$bar = $foo;             // ['foo' => 'baz']: refcount = 2
$bar['foo'] = 'bar';     // ['foo' => 'baz']: refcount = 1, ['foo' => 'bar']: refcount = 1
```

PHP arrays are refcounted, so when multiple `zval`s are referring to a same `zend_array`, it won't be copied, however, its refcount is incremented. When any of these arrays is going to be modified, Zend Engine checks the `zend_array`'s refcount. If it's not 1, create a duplicate of the array, modify it, assign the `zval` to it, and decrement the original array. This is called **separation**.

The same is true of passing an array to a function parameter by value.

```PHP
function add_foo($arr) { // ['foo' => 'baz'] : refcount = 2
    $arr['foo'] = 'bar'; // ['foo' => 'baz'] : refcount = 1, ['foo' => 'bar'] : refcount = 1
    // ...
}
$arr = ['foo' => 'baz']; // ['foo' => 'baz'] : refcount = 1
add_foo($arr);           // ['foo' => 'baz'] : refcount = 1
```

However, PHP objects are treated differently. You can modify a `zend_object`'s `property_table` without separation.

```PHP
$arr = ['foo' => 'baz'];
$foo = new ArrayObject($arr); // obj(['foo' => 'baz']) : refcount = 1
$bar = $foo;                  // obj(['foo' => 'baz']) : refcount = 2
$foo[] = 'abc';               // obj(['foo' => 'baz', 'abc']) : refcount = 2
add_foo($bar);                // obj(['foo' => 'bar', 'abc']) : refcount = 2            
```

## 2. Messing around with arrays in PHP extensions

In PHP extensions, separation should be done manually. Careless or unexperienced PHP extension developers may unintentionally mess thing up. Consider the following code.

```C
PHP_FUNCTION(add_foo_internal)
{
    zend_array* arr;
    ZEND_PARSE_PARAMETERS_START(1, 1)
        Z_PARAM_ARRAY(arr)
    ZEND_PARSE_PARAMETERS_END();
    zval bar;
    ZVAL_STRING(&bar, "bar");
    zend_hash_str_add(arr, "foo", sizeof "foo" - 1, &bar);
    // ...
}
```

Looks like pretty legitimate code. However, calling this function may bring you catastrophes.

```PHP
$foo = ['bar' => 'baz']; // ['foo' => 'baz']: refcount = 1
$bar = $foo;             // ['foo' => 'baz']: refcount = 2
add_foo_internal($bar);  // ['foo' => 'bar']: refcount = 2
```

*"Hey, wait a second, `add_foo_internal()` behaves just like `add_foo()` which accepts the parameter by reference. That's not really a bad thing, after all?"*

You're **WRONG**.

As a result of separation, if `add_foo()` accepts the parameter by reference, only `$bar` will become `['foo' => 'bar']`, while `$foo` will remain the same. However, the `add_foo_internal()` function actually modifies the `zend_array` by force, thus changing the values of all the `zval`s referencing this `zend_array`, which violates the copy-on-write mechanism of PHP.

Most of the times, that is not what you want. Even if it doesn't break your extension, it will certainly break userland PHP.

## 3. HT_ASSERT_RC1 macro in PHP 7.2+

Since PHP 7.2, there's a `HT_ASSERT_RC1()` macro defined in [zend_hash.c](https://github.com/php/php-src/blob/PHP-7.2.4/Zend/zend_hash.c), which is invoked before performing any modification on a `zend_array` in all hashtable APIs defined in this source file.

```C
#if ZEND_DEBUG
#define HT_ASSERT(ht, expr) \
    ZEND_ASSERT((expr) || ((ht)->u.flags & HASH_FLAG_ALLOW_COW_VIOLATION))
#else
#define HT_ASSERT(ht, expr)
#endif

#define HT_ASSERT_RC1(ht) HT_ASSERT(ht, GC_REFCOUNT(ht) == 1)
```

This assertion ensures that whenever a `zend_array` is modified, its refcount must be 1. If you are using a debug build of PHP, calling `add_foo_internal()` will trigger assertion failure like this:

> php: /opt/php-7.2.4/Zend/zend_hash.c:549: _zend_hash_add_or_update_i: Assertion `((ht)->gc.refcount == 1) || ((ht)->u.flags & (1<<6))' failed.

[UPGRADING.INTERNALS](https://github.com/php/php-src/blob/PHP-7.2.4/UPGRADING.INTERNALS) didn't mention this new feature, but it's one of the things I like most in PHP 7.2. The rather "pedantic" inspection of copy-on-write violation on PHP arrays is indeed helpful for PHP extension developers. It helped me found out some potentially hazardous bugs in some of my old extensions.

*"All right, I know PHP arrays are refcounted, and should be separated before modifying. But in some cases, I have to modify a `zend_array` whose refcount isn't 1. What should I do?"*

Just add a flag in the `zend_array` you'd like to mess with, by invoking `HT_ALLOW_COW_VIOLATION()` macro.

```C
#if ZEND_DEBUG
#define HT_ALLOW_COW_VIOLATION(ht) (ht)->u.flags |= HASH_FLAG_ALLOW_COW_VIOLATION
#else
#define HT_ALLOW_COW_VIOLATION(ht)
#endif
```

However, even if you know what you're doing, "having to do" such a thing is usually a code smell and caused by bad design. It is recommended that you refactor the code and avoid doing so.
