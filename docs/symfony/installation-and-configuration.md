# Symfony2 - Installation and configuration

Add Isolate bundle dependencies to composer.json file in your project

```json
{
    require: {
        "isolate/unit-of-work": "1.0.*@dev",
        "isolate/lazy-objects": "1.0.*@dev",
        "isolate/isolate": "1.0.*@dev",
        "isolate/symfony-bundle": "1.0.*@dev"
    }
}
```

> *Before first stable version you need to add all isolate/symfony-bundle dependencies manually.*

Register bundle in application kernel:

```php
<?php

use Symfony\Component\HttpKernel\Kernel;
use Symfony\Component\Config\Loader\LoaderInterface;

class AppKernel extends Kernel
{
    public function registerBundles()
    {
        $bundles = array(
            // ...
            new Isolate\Symfony\IsolateBundle\IsolateBundle(),
        );

        return $bundles;
    }
}

```

# Bundle Configuration Reference

```yml
# app/config/config.yml

isolate:
    lazy_objects: "%kernel.cache_dir%/isolate/lazy_objects" # default value
    proxy_namespace: "Proxy" # default value
    default_transaction_factory: isolate.transaction.factory # default value
    persistence_contexts:
        context_name:
            transaction_factory: custom.transaction.factory.service
```

# Commands

In order to improve performance Lazy Objects proxy classes are generated and stored in cache.
Each time when you need to change wrapped object class (for example by adding new method) you should clear cache
using symfony ``cache:clear`` command.

Before production release you can use

```bash
$ php app/console isolate:lazy-objects:generate:proxies
```

It will generate proxies even when they don't exists. In development mode missing proxy classes are generated automatically.

# Entities

In order to create entity definitions that are automatically registered in Unit of Work you should use tagged services
feature of Symfony service container. Each definition should be created by instance of ``Isolate\Symfony\IsolateBundle\Entity\Definition\Factory``
that is registered as a service and tagged with ``isolate.lazy_object.definition.factory``.

Example implementation of Factory:

```php

<?php
namespace GithubManager\Application\AppBundle\Entity\Definition;

use GithubManager\Application\AppBundle\Entity\Definition\User\EditCommandHandler;
use GithubManager\Infrastructure\Github\Client;
use Isolate\Symfony\IsolateBundle\Entity\Definition\Factory;
use Isolate\UnitOfWork\Entity\ClassName;
use Isolate\UnitOfWork\Entity\Definition;

class UserFactory implements Factory
{
    /**
     * @var Client
     */
    private $client;

    /**
     * @param Client $client
     */
    public function __construct(Client $client)
    {
        $this->client = $client;
    }

    /**
     * @return Definition
     */
    public function createDefinition()
    {
        $definition = new Definition(
            new ClassName("GithubManager\Domain\User"),
            new Definition\Identity("id")
        );

        $definition->setObserved([
            new Definition\Property(
                "followedUsers",
                new Definition\Association(new ClassName("GithubManager\Domain\User"), Definition\Association::TO_MANY_ENTITIES)
            )
        ]);

        $definition->setEditCommandHandler(new EditCommandHandler($this->client));

        return $definition;
    }
}
```

Example of how to register factory as a service:

```yml
#src/GithubManager/Application/AppBundle/Resources/config/isolate.yml

services:
    isolate.entity.definition.factory.user:
        class: GithubManager\Application\AppBundle\Entity\Definition\UserFactory
        arguments:
            - @github.client
        tags:
            - { name: isolate.entity.definition.factory }
```

# Lazy Objects

In order to create proxy definitions that are automatically registered in Lazy Objects Wrapper you should use tagged services
feature of Symfony service container. Each definition should be created by instance of ``Isolate\Symfony\IsolateBundle\LazyObject\Definition\Factory``
that is registered as a service and tagged with ``isolate.lazy_object.definition.factory``.

Example implementation of Factory:

```php
<?php

namespace GithubManager\Application\AppBundle\Entity\LazyObject\Definition;

use GithubManager\Application\AppBundle\Entity\LazyObject\Definition\User\LazyProperty\FollowedUsersInitializer;
use GithubManager\Application\AppBundle\Entity\LazyObject\Definition\User\MethodReplacement\UnfollowReplacement;
use GithubManager\Domain\User;
use Isolate\LazyObjects\Proxy\ClassName;
use Isolate\LazyObjects\Proxy\Definition;
use Isolate\LazyObjects\Proxy\LazyProperty;
use Isolate\LazyObjects\Proxy\Method;
use Isolate\LazyObjects\Proxy\MethodReplacement;
use Isolate\LazyObjects\Proxy\Property\Name;
use Isolate\Symfony\IsolateBundle\LazyObject\Definition\Factory;
use GithubManager\Infrastructure\Github\Adapter\KnpLabs\Client;

class UserFactory implements Factory
{
    /**
     * @var Client
     */
    private $client;

    /**
     * @param Client $client
     */
    public function __construct(Client $client)
    {
        $this->client = $client;
    }

    /**
     * @return Definition
     */
    public function createDefinition()
    {
        return new Definition(
            new ClassName("GithubManager\Domain\User"),
            [
                new LazyProperty(
                    new Name("followedUsers"),
                    new FollowedUsersInitializer($this->client),
                    [new Method('getFollowedUsers')]
                )
            ],
            [
                new MethodReplacement(new Method("unfollow"), new UnfollowReplacement($this->client))
            ]
        );
    }
}
```

Example of how to register factory as a service:

```yml
#src/GithubManager/Application/AppBundle/Resources/config/isolate.yml

services:
    isolate.lazy_object.factory.user:
        class: GithubManager\Application\AppBundle\Entity\LazyObject\Definition\UserFactory
        arguments:
            - @github.client
        tags:
            - { name: isolate.lazy_object.definition.factory }
```

Read about [transaction factories](transaction-factories.md)
