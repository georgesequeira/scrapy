.. _topics-spider-middleware:

=================
Spider Middleware
=================

The spider middleware is a framework of hooks into Scrapy's spider processing
mechanism where you can plug custom functionality to process the requests that
are sent to :ref:`topics-spiders` for processing and to process the responses
and items that are generated from spiders. 

.. _topics-spider-middleware-setting:

Activating a spider middleware
==============================

To activate a spider middleware component, add it to the
:setting:`SPIDER_MIDDLEWARES` setting, which is a dict whose keys are the
middleware class path and their values are the middleware orders.

Here's an example::

    SPIDER_MIDDLEWARES = {
        'myproject.middlewares.CustomSpiderMiddleware': 543,
    }

The :setting:`SPIDER_MIDDLEWARES` setting is merged with the
:setting:`SPIDER_MIDDLEWARES_BASE` setting defined in Scrapy (and not meant to
be overridden) and then sorted by order to get the final sorted list of enabled
middlewares: the first middleware is the one closer to the engine and the last
is the one closer to the spider.

To decide which order to assign to your middleware see the
:setting:`SPIDER_MIDDLEWARES_BASE` setting and pick a value according to where
you want to insert the middleware. The order does matter because each
middleware performs a different action and your middleware could depend on some
previous (or subsequent) middleware being applied.

If you want to disable a builtin middleware (the ones defined in
:setting:`SPIDER_MIDDLEWARES_BASE`, and enabled by default) you must define it
in your project :setting:`SPIDER_MIDDLEWARES` setting and assign `None` as its
value.  For example, if you want to disable the off-site middleware::

    SPIDER_MIDDLEWARES = {
        'myproject.middlewares.CustomSpiderMiddleware': 543,
        'scrapy.contrib.spidermiddleware.offsite.OffsiteMiddleware': None,
    }

Finally, keep in mind that some middlewares may need to be enabled through a
particular setting. See each middleware documentation for more info.

Writing your own spider middleware
==================================

Writing your own spider middleware is easy. Each middleware component is a
single Python class that defines one or more of the following methods:

.. module:: scrapy.contrib.spidermiddleware

.. class:: SpiderMiddleware

    .. method:: process_spider_input(response, spider)

        This method is called for each response that goes through the spider
        middleware and into the spider, for processing.

        :meth:`process_spider_input` should return ``None`` or raise an
        exception.

        If it returns ``None``, Scrapy will continue processing this response,
        executing all other middlewares until, finally, the response is handled
        to the spider for processing.

        If it raises an exception, Scrapy won't bother calling any other spider
        middleware :meth:`process_spider_input` and will call the request
        errback.  The output of the errback is chained back in the other
        direction for :meth:`process_spider_output` to process it, or
        :meth:`process_spider_exception` if it raised an exception.

        :param reponse: the response being processed
        :type response: :class:`~scrapy.http.Response` object

        :param spider: the spider for which this response is intended
        :type spider: :class:`~scrapy.spider.BaseSpider` object


    .. method:: process_spider_output(response, result, spider)

        This method is called with the results returned from the Spider, after
        it has processed the response.

        :meth:`process_spider_output` must return an iterable of
        :class:`~scrapy.http.Request` or :class:`~scrapy.item.Item` objects.

        :param response: the response which generated this output from the
          spider
        :type response: class:`~scrapy.http.Response` object

        :param result: the result returned by the spider
        :type result: an iterable of :class:`~scrapy.http.Request` or
          :class:`~scrapy.item.Item` objects

        :param spider: the spider whose result is being processed
        :type spider: :class:`~scrapy.item.BaseSpider` object


    .. method:: process_spider_exception(response, exception, spider)

        This method is called when when a spider or :meth:`process_spider_input`
        method (from other spider middleware) raises an exception.

        :meth:`process_spider_exception` should return either ``None`` or an
        iterable of :class:`~scrapy.http.Response` or
        :class:`~scrapy.item.Item` objects.

        If it returns ``None``, Scrapy will continue processing this exception,
        executing any other :meth:`process_spider_exception` in the following
        middleware components, until no middleware components are left and the
        exception reaches the engine (where it's logged and discarded).

        If it returns an iterable the :meth:`process_spider_output` pipeline
        kicks in, and no other :meth:`process_spider_exception` will be called.

        :param response: the response being processed when the exception was
          raised
        :type response: :class:`~scrapy.http.Response` object

        :param exception: the exception raised
        :type exception: `Exception`_ object

        :param spider: the spider which raised the exception
        :type spider: :class:`scrapy.spider.BaseSpider` object

.. _Exception: http://docs.python.org/library/exceptions.html#exceptions.Exception


.. _topics-spider-middleware-ref:

Built-in spider middleware reference
====================================

This page describes all spider middleware components that come with Scrapy. For
information on how to use them and how to write your own spider middleware, see
the :ref:`spider middleware usage guide <topics-spider-middleware>`.

For a list of the components enabled by default (and their orders) see the
:setting:`SPIDER_MIDDLEWARES_BASE` setting.

DepthMiddleware
---------------

.. module:: scrapy.contrib.spidermiddleware.depth
   :synopsis: Depth Spider Middleware

.. class:: DepthMiddleware

   DepthMiddleware is a scrape middleware used for tracking the depth of each
   Request inside the site being scraped. It can be used to limit the maximum
   depth to scrape or things like that.

   The :class:`DepthMiddleware` can be configured through the following
   settings (see the settings documentation for more info):

      * :setting:`DEPTH_LIMIT` - The maximum depth that will be allowed to
        crawl for any site. If zero, no limit will be imposed.
      * :setting:`DEPTH_STATS` - Whether to collect depth stats.
      * :setting:`DEPTH_PRIORITY` - Whether to prioritize the requests based on
        their depth.

HttpErrorMiddleware
-------------------

.. module:: scrapy.contrib.spidermiddleware.httperror
   :synopsis: HTTP Error Spider Middleware

.. class:: HttpErrorMiddleware

    Filter out unsuccessful (erroneous) HTTP responses so that spiders don't
    have to deal with them, which (most of the time) imposes an overhead,
    consumes more resources, and makes the spider logic more complex.

According to the `HTTP standard`_, successful responses are those whose
status codes are in the 200-300 range.

.. _HTTP standard: http://www.w3.org/Protocols/rfc2616/rfc2616-sec10.html

If you still want to process response codes outside that range, you can
specify which response codes the spider is able to handle using the
``handle_httpstatus_list`` spider attribute.

For example, if you want your spider to handle 404 responses you can do
this::

    class MySpider(CrawlSpider):
        handle_httpstatus_list = [404]

.. reqmeta:: handle_httpstatus_list

The ``handle_httpstatus_list`` key of :attr:`Request.meta
<scrapy.http.Request.meta>` can also be used to specify which response codes to
allow on a per-request basis.

Keep in mind, however, that it's usually a bad idea to handle non-200
responses, unless you really know what you're doing.

For more information see: `HTTP Status Code Definitions`_.

.. _HTTP Status Code Definitions: http://www.w3.org/Protocols/rfc2616/rfc2616-sec10.html

OffsiteMiddleware
-----------------

.. module:: scrapy.contrib.spidermiddleware.offsite
   :synopsis: Offsite Spider Middleware

.. class:: OffsiteMiddleware

   Filters out Requests for URLs outside the domains covered by the spider.

   This middleware filters out every request whose host names aren't in the
   spider's :attr:`~scrapy.spider.BaseSpider.allowed_domains` attribute.

   When your spider returns a request for a domain not belonging to those
   covered by the spider, this middleware will log a debug message similar to
   this one::

      DEBUG: Filtered offsite request to 'www.othersite.com': <GET http://www.othersite.com/some/page.html>

   To avoid filling the log with too much noise, it will only print one of
   these messages for each new domain filtered. So, for example, if another
   request for ``www.othersite.com`` is filtered, no log message will be
   printed. But if a request for ``someothersite.com`` is filtered, a message
   will be printed (but only for the first request filtred).

   If the spider doesn't define an
   :attr:`~scrapy.spider.BaseSpider.allowed_domains` attribute, or the
   attribute is empty, the offsite middleware will allow all requests.

   If the request has the :attr:`~scrapy.http.Request.dont_filter` attribute
   set, the offsite middleware will allow the request even if its domain is not
   listed in allowed domains.


RefererMiddleware
-----------------

.. module:: scrapy.contrib.spidermiddleware.referer
   :synopsis: Referer Spider Middleware

.. class:: RefererMiddleware

   Populates Request referer field, based on the Response which originated it.

RefererMiddleware settings
~~~~~~~~~~~~~~~~~~~~~~~~~~

.. setting:: REFERER_ENABLED

REFERER_ENABLED
^^^^^^^^^^^^^^^

.. versionadded:: 0.15

Default: ``True``

Whether to enable referer middleware will be enabled.

UrlLengthMiddleware
-------------------

.. module:: scrapy.contrib.spidermiddleware.urllength
   :synopsis: URL Length Spider Middleware

.. class:: UrlLengthMiddleware 

   Filters out requests with URLs longer than URLLENGTH_LIMIT

   The :class:`UrlLengthMiddleware` can be configured through the following
   settings (see the settings documentation for more info):

      * :setting:`URLLENGTH_LIMIT` - The maximum URL length to allow for crawled URLs.

