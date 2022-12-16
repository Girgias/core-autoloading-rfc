

# Function/closure binding

Okay, so to try to explain what I want to use it for and why....

* Auto-loading sometimes just isn't worth it
* Configurable per namespace functions are cool.
* Stuff I wouldn't personally condone using.

## Auto-loading sometimes just isn't worth it

So, this is something I've been thinking about since I gave some talks about how awesome dependency is. In particular, one bit of the talk is about the downsides of dependency injection.

[This slide](http://docs.basereality.com/InterfaceSegregationPortsmouth/#/42) and the next one, show how easy 'bad' code is to use compared to 'dependency injected' code.

Although I told people in the talk that you get used to, I actually got really annoyed at this overhead when I was actually writing tests for 'properly' written code. 

Having to create and inject a logger class in every single test is just not good value for money.

It kind of pains me to say it, but the Laravel way of using 'facades' aka global functions hidden behind a static class is probably a better way of doing things for certain types of object, particularly those where you are only going to have one (or maybe a couple) of types of object in a given environment.

For example, in a testing environment I would only use an in-memory logger, and so I'd have some code like this:

```php
// Boring logger
interface Logger
{
    public function log(string $level, string $message);
}


// Logger used for tests.
class InMemoryDevLogger implements Logger
{
    private $log_entries = [];

    public function log(string $level, string $message) {
        $this->log_entries[] = [$level, $message];
        frwite(STDERR, $level . " : " . $message);
    }

    public function findMessage(string $pattern): array|null {
        foreach ($this->log_entries as $log_entry) {
            [$level, $message] = $log_entry;
            if (preg_match($pattern, $message) === 1) {
                return [$level, $message];
            }
        }

        return null;
    }
}

```

And then if you had some code that needed testing:

```php
// Currently, doing everything with dependency injection.

function do_the_needful(
    UserParams $user,
    UserRepo $userRepo,
    Logger $logger
) {
    $user = $userRepo->findUser($user->id);
    if ($user === null) {
        $logger->log("INFO", "User not found");
    }
    $logger->log("INFO", "User found, about to foo User");

    $result = fooUser($user);

    if ($result !== true) {
        $logger->log("INFO", "Failed to foo user.");
        return new ErrorResponse();
    }
    $logger->log("INFO", "User has been foo'd.");
    return new SuccessResponse();
}

```

You would write the test as:

```php

class SomeTest {

    function test_do_the_needful_UserNotFound()
    {
        $userParams = createFakeUserParams();
        $emptyRepo = new EmptyUserRepo();
        $logger = new InMemoryLogger();
        
        $result = do_the_needful(
            $userParams,
            $emptyRepo,
            $logger
        );
        
        $this->assertInstanceOf($result, ErrorResponse::class);
        $messageFound = $logger->findMessage(/* Some appropriate pattern */);
        $this->assertTrue($messageFound);
    }
}
```

This isn't the worst thing in the world....but it's just tedious.

## Why not just use Laravel style 'facades' ?

One of the really nice thing about using DI is that all of the config for your application can be at the top-level of your app in a single place. That much nicer than using a service locator where the creation of objects can be all over the place.

It's also really nice to have a powerful enough config system to avoid needing separate 'dev' config, and 'prod' config.   

So, I'd really like to be able to setup a function autoloader like this: 

```php

function bindLoggerFunctions()
{
    if (ENVIRONMENT == "DEV") {
        $logger = new InMemoryLogger();
        function_add('log', $logger->log(...));
        function_add('log_search', $logger->findMessage(...));
        return;
    }
    if (ENVIRONMENT == "PROD") {
        $logger = createMonologLogger();
        function_add('log', $logger->log(...));
        function_add('log_search', log_search_not_allowed(...));
    }
    throw \Exception("Unknown env: " . ENVIRONMENT);
}

function log_search_not_allowed()
{
    throw new \Exception("Searching logs is not allowed.");
}

function loader($name) {
    if ($name === 'log' || $name === 'log_search') {
        bindLoggerFunctions();
    } 
}

```

Which would allow me to reduce my code to:


```php 
function do_the_needful(
    UserParams $user,
    UserRepo $userRepo
) {
    $user = $userRepo->findUser($user->id);
    if ($user === null) {
        log("INFO", "User not found");
    }
    log("INFO", "User found, about to foo User");

    $result = fooUser($user);

    if ($result !== true) {
        log("INFO", "Failed to foo user.");
        return new ErrorResponse();
    }
    log("INFO", "User has been foo'd.");
    return new SuccessResponse();
}
```

And would reduce the test code to by a bit:
```php

class SomeTest {

    function test_do_the_needful_UserNotFound()
    {
        $userParams = createFakeUserParams();
        $emptyRepo = new EmptyUserRepo();

        $result = do_the_needful(
            $userParams,
            $emptyRepo
        );
        
        $this->assertInstanceOf($result, ErrorResponse::class);
        $messageFound = log_search(/* Some appropriate pattern */);
        $this->assertTrue($messageFound);
    }
}
```

It's not _the_ biggest thing in the world, and it's really easy for code-purists to say "This isn't a big enough saving to be worth the hackery" but anything that reduces the costs of writing tests is worth at least some amount of tradeoff.


I think this pattern would be worth using for any type of object that never had any different implementations used in a test environment. So for example, a UserRepo is going to have different implementations of things like:

* EmptyUserRepo - for testing users not found
* SeededUserRepo - that contains prepared test accounts.
* ErrorUserRepo - all operations throw exceptions.

So that wouldn't be a good match for function binding....but for loggers, there just doesn't seem to be any value in 'doing it properly'.


# Configurable per namespace functions are cool

Okay, so the second thing that would be possible (and is slightly contradictory to the first one), is that


```php

namespace {

  funtion getLogClassNameForNamespace($namespace) {
        if ($namespace === 'Foo') {
            // For some reason, we care about Info log level stuff
            // in the Foo namespace.
            return InfoLogger::class;
        }

        // For everything else, we only care about warning.
        return WarningLogger::class;
  }

  function loader($name) {
    global $injector;

    $namespace_parts = explode('\\', $name);
    $function_name = array_pop($namespace_parts);
    $namespace = implode($namespace_parts);
    
    if ($function_name === 'log') {
        $loggerClassname = getLogClassNameForNamespace($namespace);
        $logger = $logger->make($loggerClassname);
        function_bind($name, $logger->log(...));
    }
  }

  autoload_register_function('loader');
}

namespace Foo {
    log('Info', "Hello there sailor!");
}

namespace Bar {
    log('Info', "Hello there sailor!");
}

```

* Stuff I wouldn't personally condone using

So I don't think I would ever code something like this:

```php

function add($x, $y) {
  return $x + $y;
}

function loader($name) {
  if (str_starts_with($name, "add_") === true) {
    $number = substr($name, strlen("add_"));
    $number = (int)$number;
    $fn = function ($x) use ($number) {
        return add($x, $y);    
    }
    function_create("add_", $fn);
  }
}

autoload_register_function('loader');

$result = add_2(2);

```

But I have a suspicion people would be able to create some interesting stuff.