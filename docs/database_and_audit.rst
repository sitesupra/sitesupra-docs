.. index::
    single: Doctrine
    single: EntityAudit
    single: Internals; Database

Database (Doctrine 2) and EntityAudit
=====================================

SiteSupra uses `Doctrine ORM <http://www.doctrine-project.org/>`_ for database operations (a nice introduction to Doctrine is
available at `Symfony docs <http://symfony.com/doc/current/book/doctrine.html>`_).

..
  However, SiteSupra supports content versioning, which is implemented using `EntityAudit library <https://github.com/simplethings/EntityAudit>`_.

  This implies some interesting features, such as abstract entities, custom id generation and some other things that will
  be covered in this article.

Doctrine Configuration
----------------------

In most cases you do not need to configure anything but set up database credentials in ``supra/config.yml``. Commonly,
you have to override ``framework.doctrine`` parameter only:

.. code-block:: yaml
    :linenos:

    framework:
        doctrine:
            credentials:
                hostname: localhost
                username: root
                password: ~
                charset: utf8
                database: supra9


.. TODO unclear

If you need auditing for your project issues, you will have to add them to framework.doctrine_audit.entities, along with
default values, like shown below:

.. code-block:: yaml
    :linenos:

    framework:
        doctrine_audit:
            entities:
                Supra\Package\Cms\Entity\Abstraction\Localization
                Supra\Package\Cms\Entity\Abstraction\AbstractPage
                Supra\Package\Cms\Entity\Page
                Supra\Package\Cms\Entity\GroupPage
                Supra\Package\Cms\Entity\Template
                Supra\Package\Cms\Entity\PageLocalization
                Supra\Package\Cms\Entity\PageLocalizationPath
                Supra\Package\Cms\Entity\TemplateLocalization
                Supra\Package\Cms\Entity\Abstraction\Block
                Supra\Package\Cms\Entity\Abstraction\PlaceHolder
                Supra\Package\Cms\Entity\PagePlaceHolder
                Supra\Package\Cms\Entity\TemplatePlaceHolder
                Supra\Package\Cms\Entity\PageBlock
                Supra\Package\Cms\Entity\TemplateBlock
                Supra\Package\Cms\Entity\BlockProperty
                Supra\Package\Cms\Entity\BlockPropertyMetadata
                Supra\Package\Cms\Entity\ReferencedElement\LinkReferencedElement
                Supra\Package\Cms\Entity\ReferencedElement\ImageReferencedElement
                Supra\Package\Cms\Entity\ReferencedElement\ReferencedElementAbstract
                Package\MyCustomPackage\Entity\CustomAuditedEntity

However, this will override previous values and possibly mess other packages. Thus, a better approach would we adding them
during package injection phase:

.. code-block:: php
    :linenos:

    <?php

    public function inject(ContainerInterface $container)
    {
        $frameworkConfiguration = $container->getApplication()->getConfigurationSection('framework');

        //add audited entities
        $frameworkConfiguration['doctrine_audit']['entities'] = array_merge(
            $frameworkConfiguration['doctrine_audit']['entities'],
            array(
                'Package\MyCustomPackage\Entity\CustomAuditedEntity',
            )
        );

        $container->getApplication()->setConfigurationSection('framework', $frameworkConfiguration);
    }

Basically, you can override any package configuration by using ``getConfigurationSection()`` and ``setConfigurationSection()``.

CLI Commands
------------

Please refer to SiteSupra :doc:`supra_cli` for more information. All Doctrine commands known by Symfony are available via SiteSupra CLI.

Standard event listeners
------------------------

By default ``SupraPackageFramework`` defines and initializes Doctrine using it's own ``config.yml``:

.. code-block:: yaml
    :linenos:

    doctrine:
        event_managers:
            public:
                subscribers:
                    - supra.doctrine.event_subscriber.table_name_prefixer
                    - supra.doctrine.event_subscriber.detached_discriminator_handler
                    - supra.doctrine.event_subscriber.nested_set_listener
                    - supra.doctrine.event_subscriber.timestampable

.. TODO unclear

``subscribers`` array references the following classes, also defined in ``config.yml``, ``services`` section:

.. code-block:: yaml
    :linenos:

    services:
        supra.doctrine.event_subscriber.table_name_prefixer:
            class: \Supra\Core\Doctrine\Subscriber\TableNamePrefixer
            parameters: ['su_', '']
        supra.doctrine.event_subscriber.detached_discriminator_handler:
            class: \Supra\Core\Doctrine\Subscriber\DetachedDiscriminatorHandler
        supra.doctrine.event_subscriber.timestampable:
            class: \Supra\Package\Framework\Doctrine\Subscriber\TimestampableListener
        supra.doctrine.event_subscriber.nested_set_listener:
            class: \Supra\Core\NestedSet\Listener\NestedSetListener

They serve for the following purposes:

* ``TableNamePrefixer`` adds prefixes to SiteSupra database tables (currently not-changeable, default ``su_``);
* ``DetachedDiscriminatorHandler`` is internal SiteSupra feature. Quite probably we'll tune it up and document later;
* ``TimestampableListener`` listens to changes in entities implementing ``Supra\Package\Cms\Entity\Abstraction\TimestampableInterface``, calls ``setCreationTime()`` and ``setModificationTime`` if needed;
* ``NestedSetListener`` handles changes in SiteSupra's NestedSet implementation.

If some other package must add other event subscribers, this can be done by overriding ``SupraPackageFramework`` configuration
like it is done in ``SupraPackageCms``:

.. code-block:: php
    :linenos:

    <?php

    public function inject(ContainerInterface $container)
    {
        //setting up doctrine
        $frameworkConfiguration = $container->getApplication()->getConfigurationSection('framework');

        $frameworkConfiguration['doctrine']['event_managers']['public'] = array_merge_recursive(
            $frameworkConfiguration['doctrine']['event_managers']['public'],
            array(
                'subscribers' => array(
                    'supra.cms.file_storage.event_subscriber.file_path_change_listener',
                    'supra.cms.pages.event_subscriber.page_path_generator',
                    'supra.cms.pages.event_subscriber.image_size_creator_listener',
                )
            )
        );

        $container->getApplication()->setConfigurationSection('framework', $frameworkConfiguration);
    }

You can freely alter any configurations during package injection phase (since actual entity managers and
subscribers are set up only in finishing phase).

Internal Entities and SupraId
-----------------------------

.. TODO check whether versioning is already enabled

Doctrine, by itself, is a very sensitive system. For example, it does not like when you are trying to persist entity that
already has id or restore entities with pre-set foreign keys. However, SiteSupra's versioning, based on
EntityAudit, does exactly that! Therefore, we are using:

* A custom type, called ``supraId20`` (use ``@Column(type="supraId20")``). That's currently just a 20 characters length string;
* A custom base entity ``Supra\Package\Cms\Entity\Abstraction\Entity``, which is a ``@MappedSuperclass``, and provides base methods like ``regenerateId``, ``__clone`` etc.

SiteSupra Id contains twenty symbols and looks like "018dusx9903wosockckg", where:

* First 9 symbols are reserved for timestamp converted to base36. Tto be honest, we do not use standard unix timestamps. Our base date is 16 Dec 2011, 11:33:05. That's the day when supraId was introduced;
* Next two symbols are reserved for internal counter of entities persisted in current session;
* Trailing 9 symbols are just a randomly generates suffix.

.. note::

    This is expected to be refactored to @GeneratedValue(strategy="CUSTOM") and @CustomIdGenerator(class="...") soon

EntityAudit and Versioning
--------------------------

SiteSupra's versioning is almost completely based on EntityAudit library. For more inforamtion refer to `respective documentation <https://github.com/simplethings/EntityAudit>`_.
We do not override anything there, so this should be enough if you need to implement auditing of your project entities.
