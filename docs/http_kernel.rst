.. index::
    single: HTTP kernel
    single: Internals; HTTP kernel

HTTP Kernel and Bootstrap Process
=================================

.. note::

    SiteSupra does not use `HttpKernel <https://github.com/symfony/HttpKernel>`_.
    We had eight versions of SiteSupra before moving to Symfony. Unfortuantely, no all of the Symfony concepts do suit our needs well.
    Our implementation is still a bit incomplete, especially ``RequestStack`` and forwarding, so expect refactoring soon.

SiteSupra uses plain ``HttpFoundation`` component (see `documentation of HttpFoundation <https://github.com/symfony/HttpFoundation>`_
and `HTTP requests in symfony <http://symfony.com/doc/current/book/http_fundamentals.html#requests-and-responses-in-symfony>`_
for more information).

SiteSupra mimics Symfony behaviour as close as possible - a ``Request`` object is created, every controller returns
``Response`` (read more on controllers :doc:`here <controllers>`).
Basically, request processing happens in the following order:

.. TODO review the below structure to ensure better readability

* Web server hits entry point ``webroot/index.php``;
* SiteSupra builds :doc:`container <di_container>`, ``buildContainer()`` is called;
* SiteSupra boots, ``boot()`` is called. Two events are fired at that moment - ``Supra::EVENT_BOOT_START`` and ``Supra::EVENT_BOOT_END``. Method ``boot()`` is called for every registered package, allowing early initialization;
* Request handling starts by calling ``handleRequest($request)``. This method loads ``Supra\Core\Kernel\HttpKernel`` and calls ``handle()``. Request handling by HttpKernel traverses through stages below:

  1. ``KernelEvent::REQUEST`` is thrown. Request processing stops when there is at least one event configured for response to ``RequestResponseEvent``. Request object is returned;
  2. Kernel tries to resolve controller and action by checking ``_controller`` and ``_action`` parameters of request ``AttributeBag``; if found, controller is instantiated, action is called and ``HttpKernel`` expects ``Controller`` to return a ``Response``. This is used when forwarding requests or instantiating requests without a route;
  3. If controller is not resolved yet kernel loads routing and tries to find current route. ``AttributeBag`` is overwritten by one created from route configuration and controller is actually executed. ``KernelEvent::CONTROLLER_START`` and ``KernelEvent::CONTROLLER_END`` events are fired, and any listener can override response during  ``KernelEvent::CONTROLLER_END`` event. If any exception (generic or ``ResourceNotFoundException``) is thrown request processing moves to exceptions processing (see stage 5 below);
  4. Kernel fires final ``KernelEvent::RESPONSE`` event and returns resulting ``Response`` object. Exception is thrown when the object is not an instance of ``Response``;
  5. If any exception is caught during request processing, then the exception is processed in the following way:

    1. Kernel fires ``KernelEvent::EXCEPTION`` event (again, if any listener provides ``Response`` inside this exception event, then the response is returned);
    2. If exception is instance of ``ResourceNotFoundException``, a special event is fired - ``KernelEvent::ERROR404``, which allows, for example, on-the-fly compilation of assets. Finally, if no response is available after the event is processed, the exception is re-thrown (in debug mode) or ``exception404Action`` of default exception controller (stored in container under ``exception.controller`` key) is called;
    3. If there's still and exception and no response, an exception is re-thrown (in debug mode) or ``exception500Action`` of default exception controller (stored in container under ``exception.controller`` key) is called.

* Resulting response is sent to the browser (by calling ``send()`` method)). Any unhandled exception is caught by `Debug <http://symfony.com/doc/current/components/debug/introduction.html>`_ component;
* SiteSupra shuts down firing ``Supra::EVENT_SHUTDOWN_START`` and  ``Supra::EVENT_SHUTDOWN_END`` events and calls ``shutdown()`` method of all registered packages, thus allowing some late cleanup.

As you can see, this process is pretty simple and transparent. Last thing to note must be a ``SupraJsonResponse`` class
that is used throughout CMS backend for passing messages, warnings, and errors to frontend in a common way.
See the class source code to learn more on messaging process.