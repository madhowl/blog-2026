# Dependency Injection на примере league/container

## 1. Что такое Dependency Injection и зачем нужен контейнер

Dependency Injection (внедрение зависимостей) — это паттерн, при котором класс не создаёт свои зависимости самостоятельно, а получает их извне. Это делает код слабосвязанным (decoupled) и тестируемым.

**Проблема жёсткой связи:**

```php
class Foo
{
    public function __construct()
    {
        // Foo сам создаёт Bar — жёсткая связь
        $this->bar = new Bar();
    }
}
```

**Решение через DI:**

```php
class Foo
{
    public function __construct(
        public readonly Bar $bar  // зависимость передаётся извне
    ) {}
}
```

Контейнер — это инструмент, который автоматизирует создание объектов и внедрение их зависимостей. `league/container` — лёгкая реализация PSR-11, поддерживающая PHP 8.3+ и совместимая с современными практиками .

**Ключевые возможности:**
- Простой API для регистрации и получения сервисов.
- Авто-связывание (auto-wiring) через рефлексию.
- Service Providers для организации конфигурации.
- Контекстная привязка (contextual binding) для внедрения разных реализаций одного интерфейса.
- Компиляция контейнера для production.

---

## 2. Установка и базовое подключение

**Требования:** PHP >= 8.3.0 .

```bash
composer require league/container
```

Подключение автозагрузчика:

```php
require 'vendor/autoload.php';

use League\Container\Container;
```

---

## 3. Базовое использование: регистрация и получение сервисов

### 3.1. Регистрация через `add()`

Метод `add()` регистрирует определение сервиса. По умолчанию каждый вызов `get()` создаёт **новый экземпляр** (prototype).

```php
$container = new Container();

// Регистрируем класс без зависимостей
$container->add(Acme\Bar::class);

// Регистрируем класс с зависимостью
$container->add(Acme\Foo::class)
          ->addArgument(Acme\Bar::class);

$foo = $container->get(Acme\Foo::class);

var_dump($foo instanceof Acme\Foo);       // true
var_dump($foo->bar instanceof Acme\Bar);  // true
```

Когда вы запрашиваете `Acme\Foo`, контейнер видит, что для его конструктора нужен `Acme\Bar`, сначала создаёт (или получает) `Bar`, затем передаёт его в `Foo` .

### 3.2. Псевдонимы (Aliases)

Псевдонимы позволяют привязать строковый идентификатор к классу. Это особенно полезно при работе с интерфейсами.

```php
$container->add('foo', Acme\Foo::class)
          ->addArgument(Acme\Bar::class);

$container->add('bar', Acme\Bar::class);

$foo = $container->get('foo');
```

**Практический пример с интерфейсом:**

```php
interface CacheInterface { /* ... */ }
class FileCache implements CacheInterface { /* ... */ }

// Привязываем интерфейс к реализации
$container->add(CacheInterface::class, FileCache::class);

// Теперь любой класс, зависящий от CacheInterface, получит FileCache
$container->add(LogService::class);
$logService = $container->get(LogService::class);
```

### 3.3. Общие экземпляры (Shared / Singleton)

Если нужен один и тот же экземпляр при каждом запросе, используйте `addShared()`:

```php
$container->addShared(Acme\Database::class);

$db1 = $container->get(Acme\Database::class);
$db2 = $container->get(Acme\Database::class);

var_dump($db1 === $db2); // true
```

---

## 4. Авто-связывание (Auto-Wiring)

Авто-связывание позволяет контейнеру автоматически разрешать зависимости по тим-хинтам конструктора без явной регистрации каждого класса. **По умолчанию оно отключено** — нужно зарегистрировать `ReflectionContainer` как делегат .

```php
use League\Container\Container;
use League\Container\ReflectionContainer;

$container = new Container();

// Включаем авто-связывание
$container->delegate(new ReflectionContainer());

// Теперь можно запрашивать классы, даже если они не зарегистрированы явно
$foo = $container->get(Acme\Foo::class);

var_dump($foo instanceof Acme\Foo);      // true
var_dump($foo->bar instanceof Acme\Bar); // true
```

**Ограничения авто-связывания:**
- Работает только с конструктором (constructor injection).
- Все внедряемые значения должны быть объектами (нельзя автоматически подставить строку или число).

**Кэширование резолюций:** если передать `true` в конструктор `ReflectionContainer`, контейнер будет кэшировать созданные экземпляры:

```php
$container->delegate(new ReflectionContainer(true));

$foo1 = $container->get(Acme\Foo::class);
$foo2 = $container->get(Acme\Foo::class);

var_dump($foo1 === $foo2); // true
```

---

## 5. Контекстная привязка (Contextual Binding)

Контекстная привязка позволяет внедрять **разные реализации одного интерфейса** в зависимости от того, какой класс его запрашивает .

**Сценарий:** два сервиса зависят от `CacheInterface`, но одному нужен файловый кэш, другому — Redis.

```php
interface CacheInterface { /* ... */ }
class FileCache implements CacheInterface { /* ... */ }
class RedisCache implements CacheInterface { /* ... */ }

class LogService
{
    public function __construct(public readonly CacheInterface $cache) {}
}

class UserService
{
    public function __construct(public readonly CacheInterface $cache) {}
}
```

**Регистрация с контекстными аргументами:**

```php
$container->add(LogService::class)
          ->addContextualArgument(CacheInterface::class, FileCache::class);

$container->add(UserService::class)
          ->addContextualArgument(CacheInterface::class, RedisCache::class);

$logService = $container->get(LogService::class);
$userService = $container->get(UserService::class);

var_dump($logService->cache instanceof FileCache);   // true
var_dump($userService->cache instanceof RedisCache); // true
```

**Важное ограничение:** контекстная привязка работает только для сервисов, зарегистрированных через `add()` или `addShared()`. Если класс разрешается исключительно через авто-связывание (ReflectionContainer) без явной регистрации, контекстные аргументы не применяются .

---

## 6. Service Providers (для среднего уровня)

Service Providers позволяют организовать регистрацию сервисов в отдельные классы. Определения внутри провайдера регистрируются **лениво** — только когда запрашивается один из сервисов, которые провайдер предоставляет .

```php
use League\Container\ServiceProvider\AbstractServiceProvider;

class DatabaseServiceProvider extends AbstractServiceProvider
{
    public function provides(string $id): bool
    {
        $services = [
            PDO::class,
            UserRepository::class,
        ];
        
        return in_array($id, $services, true);
    }
    
    public function register(): void
    {
        $this->getContainer()
            ->add(PDO::class)
            ->addArgument('mysql:host=localhost;dbname=app')
            ->addArgument('user')
            ->addArgument('pass');
        
        $this->getContainer()
            ->add(UserRepository::class)
            ->addArgument(PDO::class);
    }
}

// Регистрация провайдера
$container->addServiceProvider(new DatabaseServiceProvider());
```

**Bootable Service Providers:** если нужно выполнить код немедленно при добавлении провайдера (например, загрузить конфигурацию), реализуйте `BootableServiceProviderInterface` с методом `boot()` .

---

## 7. Краткий справочник по API

| Метод | Назначение |
|-------|-----------|
| `add($id, $concrete = null)` | Регистрирует сервис (новый экземпляр при каждом `get`) |
| `addShared($id, $concrete = null)` | Регистрирует сервис как singleton |
| `addArgument($arg)` | Добавляет аргумент для конструктора |
| `addContextualArgument($type, $concrete)` | Контекстная привязка для конкретного потребителя |
| `get($id)` | Получает экземпляр сервиса |
| `delegate($container)` | Подключает делегат (например, `ReflectionContainer` для авто-связывания) |
| `addServiceProvider($provider)` | Регистрирует провайдер сервисов |

---

## 8. Упражнения для закрепления (10–15 минут)

1. **Базовое внедрение:** создайте классы `Logger` и `UserService`, где `UserService` зависит от `Logger` через конструктор. Зарегистрируйте оба в контейнере и получите `UserService`.

2. **Интерфейс и реализация:** определите интерфейс `MailerInterface` и две реализации: `SmtpMailer` и `LogMailer`. Привяжите интерфейс к `SmtpMailer` и получите сервис, зависящий от `MailerInterface`.

3. **Авто-связывание:** включите `ReflectionContainer` и попробуйте получить класс с вложенной зависимостью (например, `Controller -> Service -> Repository`) без явной регистрации. Убедитесь, что все зависимости разрешились.

4. **Контекстная привязка (продвинутое):** создайте два сервиса, оба зависящих от `LoggerInterface`, но один должен получить `FileLogger`, а другой — `NullLogger`. Используйте `addContextualArgument()`.

---

## 9. Что дальше

- **PSR-11:** `league/container` реализует `Psr\Container\ContainerInterface`, что позволяет заменять его на другой контейнер без изменения кода потребителей.
- **Компиляция контейнера:** для production можно скомпилировать контейнер в standalone PHP-класс, устранив накладные расходы на рефлексию в runtime .
- **Интроспекция:** методы `getDefinitionIds()` и `getServiceProviderIds()` помогают отлаживать конфигурацию контейнера .

Dependency Injection — это не про контейнер, а про архитектуру. Контейнер лишь автоматизирует рутинную работу по созданию объектов. Начните с явной регистрации зависимостей (как в разделе 3), затем постепенно подключайте авто-связывание и контекстные привязки по мере роста приложения.
