---
title: 'PHP Retval: Used or Not?'
date: 2018-04-15 10:22:07
tags:
  - PHP
category: wiki
---
## 1. Deducing whether return value will be used

When we call a PHP function, it may return a value. However, we may not always use it. In this case, whatever the function does for returning that value is a waste of CPU cycles. In some rare cases, to improve performance, we may want to deduce whether the return value will be used.

Unfortunately, there's no way to do that in userland PHP, but in a PHP extension, that's quite easy. There's a macro defined in [zend.h](https://github.com/php/php-src/blob/PHP-7.2.4/Zend/zend.h).

```C
#define USED_RET() \
    (!EX(prev_execute_data) || \
     !ZEND_USER_CODE(EX(prev_execute_data)->func->common.type) || \
     (EX(prev_execute_data)->opline->result_type != IS_UNUSED))
```

We can see that the return value is considered used if at least one of the following conditions is met.

* There's no previous execute data.
* The function which called this function is not defined in userland PHP.
* The result type stored in the opline of previous execute data is not `IS_UNUSED`.

### 1.1 No previous execute data?

Well, the global scope has no previous execute data. You can `return` from global scope so that another script who `include`s this script will get the return value. For example:

```PHP
// foo.php
<?php
return 'foo';
```

```PHP
// bar.php
<?php
$bar = include 'foo.php';
```

Invoking `USED_RET()` in global scope will always get true. However, that doesn't concern us, because the native functions we implement in a PHP extension is never in global scope.

### 1.2 Not userland PHP?

The `func` property of previous execute data stores the pointer to the `zend_function` from which the current function is being invoked. For example, we define two native functions in an extension like this:

```C
PHP_FUNCTION(foo)
{
    RETVAL_LONG(EX(prev_execute_data)->func->common.type);
}
PHP_FUNCTION(bar)
{
    zval retval, foo;
    ZVAL_STRING(&foo, "foo");
    call_user_function(EX(function_table), NULL, &foo, &retval, 0, NULL);
    zval_ptr_dtor(&foo);
    RETVAL_LONG(Z_LVAL(retval));
}
```

Then, run the following script, and you will get "2 4 1" as output.

```PHP
$type_a = foo();
$type_b = eval('return foo();');
$type_c = bar();
echo "$type_a $type_b $type_c", PHP_EOL;
```

Let's take a look at [zend_compile.c](https://github.com/php/php-src/blob/PHP-7.2.4/Zend/zend_compile.c):

```C
#define ZEND_INTERNAL_FUNCTION              1
#define ZEND_USER_FUNCTION                  2
#define ZEND_OVERLOADED_FUNCTION            3
#define	ZEND_EVAL_CODE                      4
#define ZEND_OVERLOADED_FUNCTION_TEMPORARY  5

/* A quick check (type == ZEND_USER_FUNCTION || type == ZEND_EVAL_CODE) */
#define ZEND_USER_CODE(type) ((type & 1) == 0)
```

So it's clear that if the previous scope is in a function defined in PHP, the `type` property yields `ZEND_USER_FUNCTION`. If in `eval`ed code, `ZEND_EVAL_CODE`. If in a native function, `ZEND_INTERNAL_FUNCTION`. The `ZEND_USER_CODE()` macro is used to detect whether a function type is either `ZEND_USER_FUNCTION` or `ZEND_EVAL_CODE`. If so, that function is considered userland PHP.

The `USED_RET()` macro will always yield true if the function which invoked the current function is not defined in userland PHP. Why? Because there's no way to check whether a native function(like `bar()` in our example) will use the return value or not.

### 1.3 Result type of opline?

Finally, we check whether return value is used by checking the `result_type` property of the opline from the calling scope. You can understand opline as "line of opcode", which is the current line of opcode executed from the calling scope.

The `result_type` can be one of the following values:

```C
#define IS_CONST    (1<<0)
#define IS_TMP_VAR  (1<<1)
#define IS_VAR      (1<<2)
#define IS_UNUSED   (1<<3)  /* Unused variable */
#define IS_CV       (1<<4)  /* Compiled variable */
```

If the value of `result_type` is `IS_UNUSED`, we are certain that the return value of the called function will not be used.

## 2. Usage

As said in the previous section, we can use `USED_RET()` when the return value of our native function is not mandatory, and may cause performance overhead. Some built-in functions of PHP already take advantage of that macro, for example, functions which manipulate the internal pointer of a `zend_array`, such as `next()`, `prev()`, `end()`, `reset()`. If the user just want to do something to the pointer without fetching the corresponding value of the array, the function won't do the fetch & return job.

For example, in [array.c](https://github.com/php/php-src/blob/PHP-7.2.4/ext/standard/array.c):

```C
/* {{{ proto mixed next(array array_arg)
   Move array argument's internal pointer to the next element and return it */
PHP_FUNCTION(next)
{
    HashTable *array;
    zval *entry;
    
    ZEND_PARSE_PARAMETERS_START(1, 1)
        Z_PARAM_ARRAY_OR_OBJECT_HT_EX(array, 0, 1)
    ZEND_PARSE_PARAMETERS_END();
    
    zend_hash_move_forward(array);
    
    if (USED_RET()) {
        if ((entry = zend_hash_get_current_data(array)) == NULL) {
            RETURN_FALSE;
        }
        
        if (Z_TYPE_P(entry) == IS_INDIRECT) {
            entry = Z_INDIRECT_P(entry);
        }

        ZVAL_DEREF(entry);
        ZVAL_COPY(return_value, entry);
    }
}
/* }}} */
```

This macro is available in all current versions of PHP 7. Feel free to use it in your extension, if necessary.

## 3. How to do that in userland PHP?

You can't do that in userland PHP, not without the help of a native function. Write a function in your extension like this, and it's ready to use:

```C
PHP_FUNTION(used_ret)
{
    zend_execute_data* ex = EX(prev_execute_data);
    if (!ex)
        RETURN_TRUE;
    // Fetching the previous execute data of previous execute data.
    ex = ex->prev_execute_data;
    if (!ex || !ZEND_USER_CODE(ex->func->common.type))
        RETURN_TRUE;
    RETURN_BOOL(ex->opline->result_type != IS_UNUSED);
}
```

Just that simple :) Now we can test it with a PHP script.

```PHP
function foo()
{
    if (used_ret()) {
        echo 'Used.', PHP_EOL;
    } else {
        echo 'Not used.', PHP_EOL;
    }
    return 0;
}
foo();
$bar = foo();
```

Expected output:

> Not used.
> Used.