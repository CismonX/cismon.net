---
title: Changes of Zend API in PHP 7.3 you should be aware of
date: 2018-02-06 18:35:20
category: essay
tags:
  - PHP
  - Zend
---

After an unsuccessful attempt to compile my extension with the latest PHP, I discovered that you can no longer directly update `GC_REFCOUNT()`. As in zend_types.h, macros concerning GC refcount are defined as below:

```C
#define GC_REFCOUNT(p)          zend_gc_refcount(&(p)->gc)
#define GC_SET_REFCOUNT(p, rc)  zend_gc_set_refcount(&(p)->gc, rc)
#define GC_ADDREF(p)            zend_gc_addref(&(p)->gc)
#define GC_DELREF(p)            zend_gc_delref(&(p)->gc)
```

Meanwhile, in PHP 7.2 and older versions:

```C
#define GC_REFCOUNT(p)          (p)->gc.refcount
```

Here's a simple workaround for compatibility.

```c
#if PHP_VERSION_ID < 70300
#define GC_ADDREF(p)            ++GC_REFCOUNT(p)
#define GC_DELREF(p)            --GC_REFCOUNT(p)
#define GC_SET_REFCOUNT(p, rc)  GC_REFCOUNT(p) = rc
#endif
```

This change in internal API was intended to eliminate race-conditions in multi-thread applications, as mentioned in this [pull request](https://github.com/php/php-src/pull/2880).

Other notable API changes can be found [here](https://github.com/php/php-src/blob/master/UPGRADING.INTERNALS), with which you can make your extension compatible with PHP 7.3.
