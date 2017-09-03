---
title: Concurrency Programming in PHP
date: 2017-07-17 17:55
category: wiki
tags:
  - asynchronous
  - PHP
---

## 1. When is concurrency needed?

### Bad examples

1. Attempting to fetch the content from a remote.

```php
$handle = curl_init('http://www.an-extremely-slow-website.com/');
//This may take several seconds.
curl_exec($handle);
```

2. Performing a slow MySQL query.

```php
$pdo = new PDO($dsn, $username, $password);
//There are million of rows in table `history`, query cannot use index.
$stmt = $pdo->prepare('SELECT * FROM `history` WHERE `file_name` NOT LIKE "%.jpg"');
$stmt->execute();
```

3. Network transmission is slow.

```php
//File length is about 2MiBs in total.
while ($buffer = fread($handle, 8192)) {
    //Unfortunately, Client can only recieve around 100KiBs per second.
    fwrite($socket, $buffer, 8192);
}
fclose($handle);
```

4. Deferring is needed.

```php
while ($buffer = fread($socket, 8192)) {
    fwrite($handle, 8192);
    //We want to restrict transmission speed by sleeping 100 milliseconds every 8KiBs.
    usleep(100000);
}
```

### What do they have in common?

1. Something slow needs to be done.
2. I/O process waits and does nothing.

### Probable solutions

1. Spawn a thread/process for each request.
2. Handle requests asynchronously within one thread.

## 2. How to implement concurrency in PHP?

### Event-based model

![](event_model.png)

### Event-driven libraries in PHP

1. [libevent](https://pecl.php.net/package/event)
2. [libev](https://pecl.php.net/package/ev)
3. [libuv](https://pecl.php.net/package/uv) 

### Frameworks based on event-driven libraries

1. [ReactPHP](http://reactphp.org/)
2. [Amp](http://amphp.org/)
3. [Workerman](http://www.workerman.net/)
4. [Swoole](http://www.swoole.com/)

### Features and advantages

1. Asynchronous, non-blocking I/O for web service, filesystem, and database connection, etc.
2. Implementing high concurrency in a single thread. No cost for forking or spawning threads.

## 3.  The Amp Framework

### Basic usage

1. Event watchers

| Watcher Type | Callback Signature                       |
| ------------ | ---------------------------------------- |
| defer()      | function (string $watcherId, $callbackData) |
| delay()      | function (string $watcherId, $callbackData) |
| repeat()     | function (string $watcherId, $callbackData) |
| onReadable() | function (string $watcherId, $stream, $callbackData) |
| onWritable() | function (string $watcherId, $stream, $callbackData) |
| onSignal()   | function (string $watcherId, $signal, $callbackData) |

2. Controlling event watchers

| Method        | Behaviour                                |
| ------------- | ---------------------------------------- |
| run()         | Start the event loop with all watcher active. |
| stop()        | Terminate the event loop and continue execution to the next line after `run()`. |
| enable()      | Resume a disabled watcher back to the event loop. |
| disable()     | Temporarily remove a watcher from the event loop. |
| reference()   | Mark a watcher as referenced.            |
| unreference() | Mark a watcher as unreferenced.          |
| cancel()      | Destroy a watcher.                       |

3. Demo

```php
use Amp\Loop;
//Same as call Loop::defer() with callback right before calling Loop::run() without callback.
Loop::run(function (string $id) {
    echo "Event loop started. Watcher id is $id.\n";
    //Get called right after this tick.
    $id = Loop::defer(function (string $id, $param) {
        echo "Watcher id is $id. Watcher id of last tick is $param.\n";
    }, $id);
    echo "Watcher id of next tick is $id.\n";
    $count = 0;
    //Loop::repeat() callbacks get called after every specified interval.
    Loop::repeat(1000, function (string $id) use (&$count) {
        ++$count;
        echo "Timer callback is called for $count time(s).\n";
        if ($count == 7)
            //Loop::delay() is similar with Loop::repeat(),
            //except for that the former is destroyed right after its tick.
            Loop::delay(2500, function () use ($id) {
                //Loop::cancel() removes a specified watcher from the event loop.
                Loop::cancel($id);
            });
    });
    pcntl_signal(SIGINT, SIG_IGN);
    //Get called whenever a specific signal is sent to the process.
    $id = Loop::onSignal(SIGINT, function () {
        echo "SIGINT received. Exiting event loop.\n";
        //When Loop::stop() is called,
        //the event loop will stop right after current tick.
        Loop::stop();
    });
    //All watchers are referenced by default.
    //Unreferenced watchers won't keep the event loop alive.
    Loop::unreference($id);
});
//When there are no available watchers, the event loop exits automatically.
echo "Terminated.\n";
```

### Promises

1. What are Promises?

![](promises.jpg)

2. Asynchronous functions should return an instance of a class which implements `Amp\Promise`.
3. Promises are created by an instance of `Amp\Deferred`, which resolves the promised value, and throws an exception when an error occurs.
4. Unlike the Promises implemented in JavaScript and ReactPHP, etc, **thenables** in Amp are implemented with Coroutines.
5. Demo

```php
use Amp\Loop;
function asyncDivide($divisor, $dividend, $delay) {
    //Promises are created by Amp\Deferred.
    $deferred = new Amp\Deferred;
    Loop::delay($delay, function () use ($divisor, $dividend, $deferred) {
        $divisor = intval($divisor);
        $dividend = intval($dividend);
        if (!$dividend)
            //Reject and emit an error.
            $deferred->fail(new DivisionByZeroError('Divided by zero'));
        else
            //Resolve a result.
            $deferred->resolve($divisor / $dividend);
    });
    //The async function shall return a Promise.
    return $deferred->promise();
}
Loop::run(function () {
    //Call a function asynchronously.
    $promise = asyncDivide(4, 5, 1000);
    //The following event occurs when the Promise is resolved or rejected.
    $promise->onResolve(function (?Throwable $error, $result) {
        if ($error)
            echo $error->getMessage(), PHP_EOL;
        else
            echo 'Result is ',$result, PHP_EOL;
    });
});
```

6. Promise Combinators (in namespace `Amp\Promise`) combine multiple promises to a single Promise.

| Function | Behaviour                                |
| -------- | ---------------------------------------- |
| all()    | Resolve when all Promises in the group resolve. |
| some()   | Resolve when no less than one Promise resolves. |
| any()    | Resolve even when all Promises fail.     |
| first()  | Resolve when the first Promise in the group resolves. |

7. Promise Helpers (in namespace `Amp\Promise`)

| Function  | Behaviour                                |
| --------- | ---------------------------------------- |
| rethrow() | Forward errors generated by the given Promise to the event loop. |
| timeout() | Throw an exception if the given Promise fail to resolve or reject. |
| wait()    | Synchronously wait for a Promise to resolve. |

### Coroutines

1. What are coroutines?

![](coroutine.png)

2. In Amp, all yields of coroutines must be one of the following type.

| Type                           | Description                              |
| ------------------------------ | ---------------------------------------- |
| Amp\Promise                    | Control will be returned to the Coroutine once resolved. |
| React\Promise\PromiseInterface | Will be adapted to `Amp\Promise`.        |
| array                          | Array of Promises will be combined implicitly to `Amp\Promise\All`. |

3. Coroutine helpers (in Amp namespace)

| Function                                 | Behaviour                                |
| ---------------------------------------- | ---------------------------------------- |
| coroutine(callable $callback) : callable | Wrap a function into a coroutine.        |
| asyncCoroutine(callable $callback) : callable | Callback function do not return a Promise when called. |
| call(callable \$callback, ...\$args) : Promise | Call the given function, and return a Promise. |
| asyncCall(callable \$callback, ...\$args) : void | Do not return a Promise.                 |

4. Examples:

```php
function asyncDivide($divisor, $dividend, $delay) {
    return \Amp\coroutine(function () use ($divisor, $dividend, $delay) {
        yield new Amp\Delayed($delay);
        return $divisor / $dividend;
    });
}
Amp\Loop::run(function () {
    $value = yield asyncDivide(3, 4, 500)();
    var_dump($value);
});
```

```php
function asyncDivide($divisor, $dividend, $delay) {
    return Amp\call(function () use ($divisor, $dividend, $delay) {
        yield new Amp\Delayed($delay);
        return $divisor / $dividend;
    });
}
Amp\Loop::run(function () {
    $value = yield asyncDivide(3, 4, 500);
    var_dump($value);
});
```

### Iterators

1. In Amp, an iterator iterates through a set of Promises, and resolves alongside with the Promises. It can be recognized as a "special" Promise which can be resolved multiple times.
2. Iterators are created by `Amp\Emitter`.
3. Iterator functions are listed below.

| Method                  | Behaviour                                |
| ----------------------- | ---------------------------------------- |
| Iterator::getCurrent()  | If Promise resolves to `true`, consume value of current position. |
| Iterator::advance()     | Return a Promise which indicates whether there's a value to consume. |
| Emitter::emit()         | Emits a new value to the Iterator.       |
| Emitter::complete()     | Mark an iterator as complete and no further emits will be done. |
| Emitter::iterate()      | Return instance of Iterator.             |
| Iterator\fromIterable() | Converts arrays or `Traversable` objects into an Iterator. |

4. Examples:

```php
function subtractToZero($init, $interval) {
    $value = $init;
    //Iterators are created by Amp\Emitter.
    $emitter = new Amp\Emitter;
    Loop::repeat($interval, function ($id) use ($emitter, &$value) {
        if ($value > 0)
            $emitter->emit(--$value);
        else {
            $emitter->complete();
            //Cancel timer event when complete.
            Loop::cancel($id);
        }
    });
    //Return the iterator.
    return $emitter->iterate();
}
Loop::run(function () {
    $iterator = subtractToZero(10, 100);
    while (yield $iterator->advance())
        var_dump($iterator->getCurrent());
});
```

5. Producer is a simplified form of emitter which can be used when all values can be emitted in a single coroutine.

```php
Amp\Loop::run(function () {
    $iterator = new \Amp\Producer(function (callable $emit) {
        static $i = 0;
        while (++$i < 10) {
            yield new \Amp\Delayed(200);
            yield $emit($i);
        }
    });
    while (yield $iterator->advance())
        var_dump($iterator->getCurrent());
});
```

6. Iterator combination functions combine an array of Iterators into a single Iterator.

| Function          | Behaviour                              |
| ----------------- | -------------------------------------- |
| Iterator\concat() | Iterators are resolved one by one.     |
| Iterator\merge()  | Iterators are resolved simultaneously. |

7. Iterator transformation functions intervene the resolution of Iterators using Producer.

| Function          | Behaviour                                |
| ----------------- | ---------------------------------------- |
| Iterator\map()    | Transform the resolved value into another value. |
| Iterator\filter() | Resolved value is omitted if filter callback returns false. |

```php
Loop::run(function () {
    $iterator = \Amp\Iterator\map(subtractToZero(10, 200), function ($value) {
        return "Current value is $value.\n";
    });
    while (yield $iterator->advance())
        echo $iterator->getCurrent();
});
```

```php
Loop::run(function () {
    $iterator = \Amp\Iterator\filter(subtractToZero(10, 200), function ($value) {
        return $value != 3;
    });
    while (yield $iterator->advance())
        var_dump($iterator->getCurrent());
});
```

### Cancellation

1. Amp provides cancellation of a specific asynchronous operation. but it does not and **cannot** automatically handle cancellation. Instead, you should handle cancellation manually after its request.
2. Cancellation is implemented using `Amp\CancellationTokenSource` and `Amp\CancellationToken`.

| Method                                | Behaviour                                |
| ------------------------------------- | ---------------------------------------- |
| CancellationTokenSource::getToken()   | Returns a unique CancellationToken instance. |
| CancellationTokenSource::cancel()     | Emits a Cancellation request to its CancellationToken. |
| CancellationToken::isRequested()      | Resurns whether there is a Cancellation request. |
| CancellationToken::throwIfRequested() | Throws CancelledException if Cancellation request exists. |
| CancellationToken::subscribe()        | Callback will be invoked when the request occurs. |
| CancellationToken::unsubscribe()      | Disable a specified callback by id.      |

3. Examples:

```php
use Amp\Loop;
function subtractToZero($init, $interval, $token = null) {
    $value = $init;
    $emitter = new Amp\Emitter;
    Loop::repeat($interval, function ($id) use ($emitter, &$value, $token) {
        //Cancellation requests are received by isRequested() method.
        if ($value > 0 && (!isset($token) || !$token->isRequested()))
            $emitter->emit(--$value);
        else {
            $emitter->complete();
            Loop::cancel($id);
        }
    });
    return $emitter->iterate();
}
Loop::run(function () {
    $token_source = new \Amp\CancellationTokenSource;
    $iterator = subtractToZero(10, 200, $token_source->getToken());
    Loop::delay(1500, function () use ($token_source) {
        //Cancel this operation 1500 milliseconds after current tick.
        $token_source->cancel();
    });
    while (yield $iterator->advance())
        var_dump($iterator->getCurrent());
});
```

```php
Loop::repeat($interval, function ($id) use ($emitter, &$value, $token) {
    //Callback which is subscribed to a Cancellation Token
    //will be invoked before the callback marked as cancelled.
    $token->subscribe(function () use ($id, $emitter) {
        Loop::cancel($id);
    });
    if ($value > 0)
        $emitter->emit(--$value);
    else {
        $emitter->complete();
        Loop::cancel($id);
    }
});
```
