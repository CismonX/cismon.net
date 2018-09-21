---
title: Unveiling the Mysteries of PHP Object Properties
date: 2018-09-21 15:40:18
category: essay
tags:
  - PHP
---

## 1. Properties and hashtables

In statically typed languages, such as C++, properties of objects can be accessed via offset, which is determined at compile time. However, PHP is dynamically typed. There's no way we can determine which type a `zval` is until we execute to the exact line of opcode, so it's impractical trying to access properties via offset.

In the Zend implementation of PHP, properties of an object is stored in a `zend_array`, aka a hashtable. See the definition of `zend_object`:

```C
struct _zend_object {
    zend_refcounted_h           gc;
    uint32_t                    handle;
    zend_class_entry           *ce;
    const zend_object_handlers *handlers;
    HashTable                  *properties;
    zval                        properties_table[1];
};
```

Buckets of `zobj->properties` are key-value pairs of property names and values. Whenever a property of a `zend_object` is being accessed, the property name is applied to the hash function.

*"So what's `zobj->properties_table` anyway?"*

Well, it's the place where values of default properties are stored, and the closest thing we have to fetching properties by offset. Let's suppose there's a userland PHP class defined as below.

```PHP
class Pair
{
    public $first;
    public $second;
    function __construct($first = null, $second = null)
    {
        $this->first = $first;
        $this->second = $second;
    }
}
```

The `$first` and `$second` property of class `Pair` are defined explicitly in the code, thus, they are considered as default properties. Value of `$first` is stored in `zobj->properties_table[0]`, and `$second` in `zobj->properties_table[1]`, which is determined at compile time. So you can see that the number of default properties affect the size of `zend_object`.

However, that doesn't mean that those properties can be accessed via offset. Consider the following code:

```PHP
declare(strict_types=1);        // Use strict typing.

function foo(Pair $bar, $baz = null) : Pair
{
    echo $bar->first;           // No, hashtable is used.
    if ($baz instanceof Pair) {
        echo $baz->second;      // Still no.
    }
    return $bar;
}

echo foo(new Pair())->first;    // Noooo..
```

After a short tour of the PHP source code or some debugging with GDB, it will dawn upon you, that even the offset of a default property can be determined at compile time in theory, the Zend engine doesn't do the optimization, not even in the incoming PHP 7.3 (You can RFC it if you like, but believe me, that could be quite a nasty job).

*"Wait a sec. I didn't see a hashtable holding the names of those default properties."*

Of course not. That hashtable is defined in the class entry, whose attributes are determined at compile time.

```C
struct _zend_class_entry {
    char type;
    zend_string *name;
    struct _zend_class_entry *parent;
    int refcount;
    uint32_t ce_flags;
    int default_properties_count;
    int default_static_members_count;
    zval *default_properties_table;
    zval *default_static_members_table;
    zval *static_members_table;
    HashTable function_table;
    HashTable properties_info;
    HashTable constants_table;
    // ...
```

Whenever you're trying to access a property, it first search the `zobj->ce->properties_info`, whose buckets are key-value pairs of default property names and their offsets. If the key does not exist, search `zobj->properties`. See [the source code](https://github.com/php/php-src/blob/php-7.2.9/Zend/zend_object_handlers.c) for details. Behaviors on `zend_object`s are controlled by `zobj->handlers`. Unless overriden by native code, those std handler functions will always get called.

Having to perform a second hash search makes accessing dynamic property slower. Meanwhile, as `zobj->properties` is `NULL` by default, allocating and initializing a `zend_array` as container for dynamic properties is also expensive.

For performance comparison, try the following benchmark.

```PHP
// 1. write property
$foo = new Pair(0, 0);
$start = microtime(true);
for ($i = 0; $i < 10000000; ++$i) {
    $foo->first = $i;
}
$duration = microtime(true) - $start;
// 2. call constructor
$start = microtime(true);
for ($i = 0; $i < 1000000; ++$i) {
    $foo = new Pair($i, $i);
}
$duration = microtime(true) - $start;
```

For the first benchmark, it's about 32% faster when using a `Pair` with default properties than one without. For the second one, it's 41%. (Using PHP 7.2.9 ZTS DEBUG on Darwin, and similar results for other builds of PHP)

## 2. Access default properties by offset

Well, in userland PHP, there's nothing more we can do. Just bear in mind that dynamic properties work poorly in performance. Always define properties explicitly if you can.

However, in native context (e.g. in a PHP extension), performance can be improved drastically, as there's no need for hashtables when accessing default properties.

With Zend API `zend_declare_property()` and its variants, we can declare default properties in `PHP_MINIT_FUNCTION`, which will be called only once when the extension gets loaded. Then we can access the values of those properties by `zobj->properties_tables[offset]`. It's recommended that you just use  macro `OBJ_PROP_NUM()`.

See the following example for a `Pair::__construct()`:

```C
PHP_METHOD(Pair, __construct)
{
    zval* first;
    zval* second;
    ZEND_PARSE_PARAMETERS_START(2, 2)
        Z_PARAM_ZVAL(first)
        Z_PARAM_ZVAL(second)
    ZEND_PARSE_PARAMETERS_END();
    // Get current `zend_object`.
    zend_object* zobj = Z_OBJ_P(getThis());
    // Get pointer to properties by offset.
    zval* pair_first = OBJ_PROP_NUM(zobj, 0);
    zval* pair_Second = OBJ_PROP_NUM(zobj, 1);
    // Update `$this->first`.
    Z_TRY_ADDREF_P(first);
    zval_ptr_dtor(pair_first);
    ZVAL_COPY_VALUE(pair_first, first);
    // Update `$this->second`.
    Z_TRY_ADDREF_P(second);
    zval_ptr_dtor(pair_second);
    ZVAL_COPY_VALUE(pair_second, second);
}
```

