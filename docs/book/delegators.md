# Delegators

`Laminas\ServiceManager` can instantiate [delegators](http://en.wikipedia.org/wiki/Delegation_pattern)
of requested services, decorating them as specified in a delegate factory
implementing the [delegator factory interface](https://github.com/laminas/laminas-servicemanager/tree/master/src/Factory/DelegatorFactoryInterface.php).

The delegate pattern is useful in cases when you want to wrap a real service in
a [decorator](http://en.wikipedia.org/wiki/Decorator_pattern), or generally
intercept actions being performed on the delegate in an
[AOP](http://en.wikipedia.org/wiki/Aspect-oriented_programming) fashioned way.

## Delegator factory signature

A delegator factory has the following signature:

```php
use Interop\Container\ContainerInterface;

public function __invoke(
    ContainerInterface $container,
    $name,
    callable $callback,
    array $options = null
);
```

The parameters passed to the delegator factory are the following:

- `$container` is the service locator that is used while creating the delegator
  for the requested service.
- `$name` is the name of the service being requested.
- `$callback` is a [callable](http://www.php.net/manual/en/language.types.callable.php) that is
  responsible for instantiating the delegated service (the real service instance).
- `$options` is an array of options to use when creating the instance; these are
  typically used only during `build()` operations.

## A Delegator factory use case

A typical use case for delegators is to handle logic before or after a method is
called.

In the following example, an event is being triggered before `Buzzer::buzz()` is
called and some output text is prepended.

The delegated object `Buzzer` (original object) is defined as following:

```php
class Buzzer
{
    public function buzz()
    {
        return 'Buzz!';
    }
}
```

The delegator class `BuzzerDelegator` has the following structure:

```php
use Laminas\EventManager\EventManagerInterface;

class BuzzerDelegator extends Buzzer
{
    protected $realBuzzer;
    protected $eventManager;

    public function __construct(Buzzer $realBuzzer, EventManagerInterface $eventManager)
    {
        $this->realBuzzer   = $realBuzzer;
        $this->eventManager = $eventManager;
    }

    public function buzz()
    {
        $this->eventManager->trigger('buzz', $this);

        return $this->realBuzzer->buzz();
    }
}
```

To use the `BuzzerDelegator`, you can run the following code:

```php
$wrappedBuzzer = new Buzzer();
$eventManager  = new Laminas\EventManager\EventManager();

$eventManager->attach('buzz', function () { echo "Stare at the art!\n"; });

$buzzer = new BuzzerDelegator($wrappedBuzzer, $eventManager);

echo $buzzer->buzz(); // "Stare at the art!\nBuzz!"
```

This logic is fairly simple as long as you have access to the instantiation
logic of the `$wrappedBuzzer` object.

You may not always be able to define how `$wrappedBuzzer` is created, since a
factory for it may be defined by some code to which you don't have access, or
which you cannot modify without introducing further complexity.

Delegator factories solve this specific problem by allowing you to wrap,
decorate or modify any existing service.

A simple delegator factory for the `buzzer` service can be implemented as
following:

```php
use Interop\Container\ContainerInterface;
use Laminas\ServiceManager\Factory\DelegatorFactoryInterface;

class BuzzerDelegatorFactory implements DelegatorFactoryInterface
{
    public function __invoke(ContainerInterface $container, $name, callable $callback, array $options = null)
    {
        $realBuzzer   = call_user_func($callback);
        $eventManager = $container->get('EventManager');

        $eventManager->attach('buzz', function () { echo "Stare at the art!\n"; });

        return new BuzzerDelegator($realBuzzer, $eventManager);
    }
}
```

You can then instruct the service manager to handle the service `buzzer` as a
delegate:

```php
use Laminas\ServiceManager\Factory\InvokableFactory;
use Laminas\ServiceManager\ServiceManager;

$serviceManager = new Laminas\ServiceManager\ServiceManager([
    'factories' => [
        Buzzer::class => InvokableFactory::class,
    ],
    'delegators' => [
        Buzzer::class => [
            BuzzerDelegatorFactory::class,
        ],
    ],
]);

// now, when fetching Buzzer, we get a BuzzerDelegator instead
$buzzer = $serviceManager->get(Buzzer::class);

$buzzer->buzz(); // "Stare at the art!\nBuzz!"
```

You can specify multiple delegators for a service. Each will add one decorator
around the instantiation logic of that particular service.

This latter point is the primary use case for delegators: *decorating the
instantiation logic for a service*.

## Delegator Factories and Service Aliases

In typical [service manager configurations](./configuring-the-service-manager.md) you have the opportunity to alias services. The following configuration would enable you to retrieve a `Buzzer` instance by its concrete implementation name and by the name of an interface that it implements, in this case, `BuzzerInterface`.

```php
$serviceManager = new Laminas\ServiceManager\ServiceManager([
    'factories' => [
        Buzzer::class => Laminas\ServiceManager\Factory\InvokableFactory::class,
    ],
    'aliases' => [
        BuzzerInterface::class => Buzzer::class,
    ],
]);
```

Currently, a delegator factory that targets an alias will not execute. Delegators must be configured using the resolved name of the service.

For example, given the following configuration, **no delegation would occur**:

```php
$serviceManager = new Laminas\ServiceManager\ServiceManager([
    'factories' => [
        Buzzer::class => Laminas\ServiceManager\Factory\InvokableFactory::class,
    ],
    'aliases' => [
        BuzzerInterface::class => Buzzer::class,
    ],
    'delegators' => [
        BuzzerInterface::class => [
            BuzzerDelegatorFactory::class, // will not be executed
        ],
    ],
]);
```

In order for delegation to occur, the above configuration would need to be modified to target the resolved service name:

```php
$serviceManager = new Laminas\ServiceManager\ServiceManager([
    'factories' => [
        Buzzer::class => Laminas\ServiceManager\Factory\InvokableFactory::class,
    ],
    'aliases' => [
        BuzzerInterface::class => Buzzer::class,
    ],
    'delegators' => [
        Buzzer::class => [
            BuzzerDelegatorFactory::class, // will now execute as expected
        ],
    ],
]);
```

Retrieving the `Buzzer` using its resolved name "`Buzzer::class`" or its alias "`BuzzerInterface::class`" will now both yield delegated instances.
