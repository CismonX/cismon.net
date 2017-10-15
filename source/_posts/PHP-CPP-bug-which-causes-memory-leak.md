---
title: PHP-CPP bug which causes memory leak
date: 2017-09-19 10:32:20
category: essay
tags:
  - PHP
  - Zend
---

## TL, DR

Recently I was working with PHP-CPP(2.0.0 release) for my projects. When running a test, I accidentally discovered (with `debug_zval_dump()`) that when an object gets out of scope, its refcount does not decrement, which causes memory leak.

This is not a normal behavior for a regular PHP object, however, this object is declared and instantiated in my C++ code instead of user space, thus its garbage collection cannot be automatically done by Zend Engine. There must be something wrong with PHP-CPP.

## Wrapping C++ objects in PHP

As we know, we can wrap an object which extends `Php::Base` into a `Php::Value` using constructor `Value::Value(const Base *object)`. And if the object is instantiated in C++, you have to make it accessible  to Zend Engine by calling constructor `Object::Object(const char *name, Base *base)` at least once. Make sure the object is instantiated with `new`, and once wrapped, you shall never attempt to `delete` it. (Similarly, never use `std::shared_ptr` for the object.) Otherwise you'll get segmentation fault.

PHP-CPP makes sure garbage collection of C++ objects wrapped within `Php::Value` is handled automatically, for the `Php::Base*` pointer is stored in a `std::unique_ptr` in a PHP-CPP's `Php::ObjectImpl` object. The latter gets destroyed once refcount becomes zero.

## Problem cause discovered

And here's the problem. Why didn't refcount decrease when `Php::Value` gets destroyed? Let's take a peek at the destructor of `Php::Value`:

```C++
/**
 *  Destructor
 */
Value::~Value()
{
    // reduce the refcount - if necessary
    Z_TRY_DELREF_P(_val);
}
```

And constructor for wrapping `Php::Base*`:

```C++
/**
 *  Wrap around an object
 *  @param  object
 */
Value::Value(const Base *object)
{
    // there are two options: the object was constructed from user space,
    // and is already linked to a handle, or it was constructed from C++
    // space, and no handle does yet exist. But if it was constructed from
    // C++ space and not yet wrapped, this Value constructor should not be
    // called directly, but first via the derived Php::Object class.
    auto *impl = object->implementation();

    // do we have a handle?
    if (!impl) throw FatalError("Assigning an unassigned object to a variable");

    // set it to an object
    Z_TYPE_INFO_P(_val) = IS_OBJECT;
    Z_OBJ_P(_val) = impl->php();

    // increase refcount
    GC_REFCOUNT(impl->php())++;
}
```

It seems that we discovered the cause of the problem. `GC_REFCOUNT` is the reference counter for a `zend_object`, while function `Z_TRY_DELREF_P()` decrements the refcount for a `zval`. Thus, refcount increments every time you wrap the object with `Value::Value(const Base *object)`, but never goes down when wrapped object is destroyed. Hence the problem.

## Problem solved

Changing `GC_REFCOUNT(impl->php())++` into `Z_ADDREF_P(_val)` will do the trick. Not long after I started writing this blog I discovered that the master branch of PHP-CPP's GitHub repository has already [fixed](https://github.com/CopernicaMarketingSoftware/PHP-CPP/pull/290) this bug. So it's highly recommended that you use the master branch of PHP-CPP(currently works fine with my projects), instead of the 2.0.0 release. See [here](https://github.com/CopernicaMarketingSoftware/PHP-CPP/compare/v2.0.0...master) for commits since the 2.0.0 release.