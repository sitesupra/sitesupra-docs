.. index::
    single: Internals; Dependency injection
    single: Dependency injection

Dependency Injection
====================

Core Concepts
-------------

SiteSupra Dependency Injection layer or DI in short is based on `Pimple <http://pimple.sensiolabs.org/>`_.
If you're curious about what DI is you can read the following Wiki articles:

* `Inversion of control <http://en.wikipedia.org/wiki/Inversion_of_control>`__;
* `Dependency injection <http://en.wikipedia.org/wiki/Dependency_injection>`__.

We also recommend read `Symfony's documentation <http://symfony.com/doc/current/book/service_container.html>`_ about basic DI principles.

SiteSupra main container class is ``Supra\Core\DependencyInjection\Container``.
It extends Pimple's `Container <https://github.com/silexphp/Pimple/blob/master/src/Pimple/Container.php>`_, implements SiteSupra's
``Supra\Core\DependencyInjection\ContainerInterface``, implements some hard-coded methods (remember, we're a CMS, and
not a full-stack framework, some items like Doctrine or Cache are always present), and provides parameters
handling.

.. note::

    It's almost certain that we will drop hard-coded getters later and build container definition on-the-fly in the
    same way Symfony does.

Everything is simple, right? Last but not least to note would be that any object implementing
``Supra\Core\DependencyInjection\Container\ContainerAware`` will be provided with ``Container`` on instantiation
(calling setContainer).

Container Building Process
--------------------------

.. note::

    The code will be refactored soon.

SiteSupra core class (``Supra\Core\Supra``, extending ``Supra\Core\DependencyInjection\ContainerBuilder``) builds and
returns ``Container`` object during call to ``buildContainer``. This is done in the following steps:

* Pre-setting some basic variables and objects (like directories, `HttpFoundation <https://github.com/symfony/HttpFoundation>`_ objects, :doc:`supra_cli`, and so on);
* Injecting packages (allowing to expose their basic configuration);
* Building configuration (there the configuration is being validated, default values set, container parameters are substituted, and so on);
* Finishing configuration when packages can override or extend config values of other packages;
* Firing ``Supra::EVENT_CONTAINER_BUILD_COMPLETE`` event.

Package Integration and Two-pass Container Building
---------------------------------------------------

First of all, a package needs to be registered. This is done by overriding ``registerPackages`` in ``SupraApplication``
class (located in ``supra/SupraApplication.php``). This method simply returns array of package instances, like in the example below:

.. code-block:: php
    :linenos:

    <?php

    use Supra\Core\Supra;

    class SupraApplication extends Supra
    {
        protected function registerPackages()
        {
            return array(
                new \Supra\Package\Framework\SupraPackageFramework(),
                new \Supra\Package\Cms\SupraPackageCms(),
                new \Supra\Package\CmsAuthentication\SupraPackageCmsAuthentication(),
                new \Supra\Package\DebugBar\SupraPackageDebugBar(),

                new \Sample\SamplePackage()
            );
        }
    }

.. TODO it would be nice to demonstrate output


Each package must extend ``Supra\Core\Package\AbstractSupraPackage``.
You can override any of the following methods to alter SiteSupra behavior:

* ``boot()`` method will be called during SiteSupra boot, see :doc:`http_kernel`;
* ``inject(ContainerInterface $container)`` method will be called during Container building in package injection phase (see above);
* ``finish(ContainerInterface $container)`` method will be called finishing Container build after the configuration is processed;
* ``shutdown()`` method will be called during SiteSupra shutdown, see :doc:`http_kernel`.

Package Configuration
---------------------

As mentioned above package configuration may occur in two phases - injection phase and finishing phase. Let's look at both of them starting from ``inject()``:

.. code-block:: php
    :linenos:

    <?php

    public function inject(ContainerInterface $container)
    {
        $this->loadConfiguration($container);

        $container->getConsole()->add(new DoFooBarCommand());

        $container[$this->name.'.some_service_name'] = function (ContainerInterface $container) {
            return new SomeService();
        };

        if ($container->getParameter('debug')) {
            //prepare some extended logging, for example
        }
    }

The most important call would be ``$this->loadConfiguration()`` (line 5). This method loads configuration file (by
default it is ``Resources/config/config.yml``). To load your own configuration pass the file name to the method as a second parameter .

This call parses config file, processes the configuration using package configuration definition (more on that on
`Symfony configuration component article <http://symfony.com/doc/current/components/config/definition.html>`_), and stores
the values for further processing.

Later you can access already defined services (see line ``line 7``, which though is not a very good approach since it
instantiates the service), add your own service definitions (``lines 9-11``), and access container parameters (``line 13``).

Each package has it's own configuration definition. Concrete configuration object is created during call to ``getConfiguration()``
method. By default, if there is a package named ``SupraPackageFooBar`` in namespace ``Com\Package\FooBar``, then the method will search
for configuration definition ``SupraPackageFooBarConfiguration`` in namespace ``Com\Package\FooBar\Configuration``. Of
course, you can always override you package's method ``getConfiguration()`` and implement your own logic.

The configuration class should extend ``Supra\Core\Configuration\AbstractPackageConfiguration`` and implement
``ConfigurationInterface``. This forces you to implement function ``getConfigTreeBuilder()``, returning instance of
``Symfony\Component\Config\Definition\Builder\TreeBuilder``. If you're curious about what is a ``TreeBuilder`` and how
exactly the configuration is being defined, please read `Defining a Hierarchy of Configuration Values Using the TreeBuilder <http://symfony.com/doc/current/components/config/definition.html#defining-a-hierarchy-of-configuration-values-using-the-treebuilder>`_
on official Symfony documentation web site. Let's take configuration of ``SupraPackageFrameworkConfiguration`` as an example:

.. code-block:: php
    :linenos:

    <?php

    class SupraPackageFrameworkConfiguration extends AbstractPackageConfiguration implements ConfigurationInterface
    {
        /**
         * Generates the configuration tree builder.
         *
         * @return \Symfony\Component\Config\Definition\Builder\TreeBuilder The tree builder
         */
        public function getConfigTreeBuilder()
        {
            $treeBuilder = new TreeBuilder();

            $treeBuilder->root('framework')
                    ->children()
                        ->append($this->getAuditDefinition())
                        //some other definitions are skipped for illustrative purposes
                        ->append($this->getServicesDefinition())
                    ->end();

            return $treeBuilder;
        }

        public function getAuditDefinition()
        {
            $definition = new ArrayNodeDefinition('doctrine_audit');

            $definition->children()
                    ->arrayNode('entities')
                        ->prototype('scalar')->end()
                    ->end()
                    ->arrayNode('ignore_columns')
                        ->prototype('scalar')->end()
                    ->end()
                ->end();

            return $definition;
        }
    }

Root node (``line 14``) must match your package name. The rest of configuration definition is standard for
Symfony-based applications (``lines 24-38``), except for call of ``->append($this->getServicesDefinition())``, which is
inherited from ``AbstractPackageConfiguration`` and enables parsing of ``services`` section of your configuration file.

Package configuration files are simple yml files as shown below:

.. code-block:: yaml
    :linenos:

    services:
        supra.framework.session_storage_native:
            class: \Symfony\Component\HttpFoundation\Session\Storage\NativeSessionStorage
            parameters: [[], "@supra.framework.session_handler_doctrine"]
        supra.framework.session_handler_doctrine:
            class: \Supra\Package\Framework\Session\DoctrineSessionHandler
    #some config parts are skipped for illustrative purposes
    doctrine:
        #some config parts are skipped for illustrative purposes
        credentials:
            hostname: localhost
            username: root
            password: ~
            charset: utf8
            database: supra9
        connections:
            default:
                host: %framework.doctrine.credentials.hostname%
                user: %framework.doctrine.credentials.username%
                password: %framework.doctrine.credentials.password%
                dbname: %framework.doctrine.credentials.database%
                charset: %framework.doctrine.credentials.charset%
                driver: mysql
                event_manager: public
        entity_managers:
            public:
                connection: default
                event_manager: public
        default_entity_manager: public
        default_connection: default
    doctrine_audit:
        entities: []
        ignore_columns:
            - created_at
            - updated_at
            - lock


.. @TODO need to confirm the last statement on setter injection support is correct

``Lines 1-6`` define services. Key is service ID, 'class' defines class name and 'parameters' section enables setter injection
(note that you can inject other services referenced with '@' as shown in ``line 4``). Setter injection is not yet supported.

First level keys will become container parameters prefixed with package name. In the example above,
container parameters are 'framework.doctrine' and 'framework.doctrine_audit', and you can call something like
``$container->getParameter('framework.doctrine_audit')['entities']`` later in your code.

You may also reference any parameter using percent notation (``%parameter.name%``). In the example above, ``line 18``
references value from ``line 11``, possibly overridden by another package or main SiteSupra's ``config.yml``.

After calling ``inject()`` method of all packages, container builder merges configuration values (also replacing /
referencing parameters), and starts calling ``finish()`` method of all packages, in load order. You ``finish()`` method
can look like so:

.. code-block:: php
    :linenos:

    <?php

    public function finish(ContainerInterface $container)
    {
        //extend some other package service
        $container->extend('some.other.service', function ($originalService, $container) {
            $originalService->callSomeMethod();

            return new SomeWrapper($originalService);
        };

        $doctrineConfig = $container->getParameter('framework.doctrine');

        //processed configuration from example above. with merged parameters and optionally overridden by main config.yml
        $connectionDetails = $doctrineConfig['connections']['default'];
    }

So, summing up:

1. You define your configuration in ``inject()`` method;
2. Container processes your configuration and merges it;
3. You retrieve processed values from container in ``finish()`` method and define your services;
4. Resulting container is available throughout SiteSupra.

Main SiteSupra Configuration File (config.yml)
----------------------------------------------

Default SiteSupra configuration file ``supra/config.yml.example``:

.. code-block:: yaml
    :linenos:

    cms:
        active_theme: default
    framework:
        doctrine:
            credentials:
                hostname: localhost
                username: root
                password: ~
                charset: utf8
                database: supra9
    cms_authentication:
        users:
            shared_connection: null
            user_providers:
                doctrine:
                    supra.authentication.user_provider.public:
                        em: public
                        entity: CmsAuthentication:User
            provider_chain: [ doctrine.entity_managers.public ]

Top-level keys correspond to package names, corresponding values are deep-merged with default values resolved in injection
phase. Here you can see how default 'doctrine.configuration' values are merged with defaults from SupraPackageFramework;
any part of configuration can be overridden.

Container Parameter Handling, Parameter Substitution
----------------------------------------------------

*Parameters* are SiteSupra-specific extension to Pimple. Basically they represent simple key-value storage (with all
the getters and setters. Refer to ``Supra\Core\DependencyInjection\Container`` for more information. However, some
of the methods are worth to be noted separately:

* ``replaceParameters`` searches array of data and replaces all parameters enclosed in percent signs (like %foo.bar%) to their respective values;
* ``replaceParametersScalar`` replaces all parameters enclosed in percent signs (like %foo.bar%) to their respective values in a scalar variable (string);
* ``getParameter`` threads dots inside parameter name as internal array keys (thus allowing you to call ``$container->getParameter('foo.bar.buz.example')`` instead of ``$container->getParameter('foo.bar')['buz']['example']``).

Standard Container Parameters
-----------------------------

Standard container parameters that can help you in development process are listed below.

Directories
~~~~~~~~~~~

There is a number of container parameters reflecting SiteSupra directory structure:

* ``directories.project_root`` for project root folder (with ``composer.json`` and other core files);
* ``directories.supra_root`` for directory where ``Supra.php`` and ``config.yml`` reside;
* ``directories.storage`` for storage folder;
* ``directories.cache`` for cache folder (inside storage root);
* ``directories.web`` for webroot (this is where SiteSupra entry point, ``index.php``, is);
* ``directories.public`` for asset root, ``Resources\public`` folders of every package are symlinked there.

Environments and Debugging
~~~~~~~~~~~~~~~~~~~~~~~~~~

Some parameters are affected by current :doc:`development settings <development_and_production>`:

* ``environment`` shows current environment - currently on of ``cli``, ``prod``, or ``dev``;
* ``debug`` shows current debug state - either ``true`` or ``false``.

Service Definition
------------------

Adding ``->addServiceDefinition()`` to package configuration will allow that package to define services.
Service definition has to reside under section ``services`` in configuration file.

A simple service definition contains service id and class name:

.. code-block:: yaml
    :linenos:

    services:
        locale.manager:
            class: \Supra\Core\Locale\LocaleManager

you can provide constructor arguments as an array:

.. code-block:: yaml
    :linenos:

    services:
        supra.doctrine.event_subscriber.table_name_prefixer:
            class: \Supra\Core\Doctrine\Subscriber\TableNamePrefixer
            parameters: ['su_', '']

or even use container parameters as arguments:

.. code-block:: yaml
    :linenos:

    services:
        supra.framework.session_storage_native:
            class: \Symfony\Component\HttpFoundation\Session\Storage\NativeSessionStorage
            parameters: [[], "@supra.framework.session_handler_doctrine"]

Unfortunately, caller injections are not possible with SiteSupra yet. But still you can use common Pimple's approach
during ``inject()`` or ``finish()``:

.. code-block:: php
    :linenos:

    <?php

    $container['some.service'] = function ($container) use ($dependency1, $dependency2) {
        $service = new SomeService($dependency1);

        $service->setDependency2($dependency2);

        $service->intialize();

        return $service;
    };

