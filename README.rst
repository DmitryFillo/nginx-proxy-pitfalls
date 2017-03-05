========================================
NGINX proxy pitfalls with DNS resolving
========================================

If you're using ``proxy_pass`` and your endpoint's IPs can vary in time, please read this article to avoid misunderstandings about how nginx works. 

.. contents::

TL;DR
=====

If you want to force nginx resolving your endpoints, you should:

* Use variables with proxy_pass, e.g. ``proxy_pass https://$endpoint/;``, where ``$endpoint`` can be manually setted or from location regexp.
* Make sure that your endpoint isn't used in the another locations, because it will break resolving. Move it to the ``upstream`` for that location to make resolving work as expected.

But I recommend to read all article, because it's interesting.

Explanatory example
===================

.. code:: nginx

  location /api/ {
      proxy_pass http://api.com/;
  }

In this case nginx will resolve api.com only once at startup (or reload). But there are some cases when your endpoint can be resolved to any IP, e.g. if you're using load balancer which doing magic failover via DNS mapping. If api.com will point to another IP your proxying will fail.

Trying to find solution
=======================

Add a resolver directive
------------------------

You can check `official nginx documentation <http://nginx.org/en/docs/>`_ and find `resolver <http://nginx.org/en/docs/http/ngx_http_core_module.html#resolver>`_ directive:

.. code:: nginx

  location /api/ {
      resolver 8.8.8.8;
      proxy_pass https://api.com/;
  }

No, it will not work. Even this will not work:

.. code:: nginx

  location /api/ {
      resolver 8.8.8.8 valid=1s;
      proxy_pass https://api.com/;
  }

It's because of nginx doesn't respect ``resolver`` directive in this case. It will resolve api.com only at startup (or reload) by system resolver (/etc/resolv.conf) anyway, even if real TTL of A/AAAA record api.com is 1s.

Add variables
-------------

You can google a bit and find that `nginx try to resolve proxy endpoint with variables <https://trac.nginx.org/nginx/ticket/723>`_. Also `official documentation for proxy_pass directive notices this too <http://nginx.org/en/docs/http/ngx_http_proxy_module.html#proxy_pass>`_. Hmmm, I think this should be noticed in the ``resolver`` description, but let's try anyway:

.. code:: nginx

  location /api/ {
      set $endpoint api.com;
      resolver 8.8.8.8 valid=10s;
      proxy_pass https://$endpoint/;
  }

Works as expected. These configurations works too:

.. code:: nginx

  set $endpoint api.com;
  location /api/ {
      resolver 8.8.8.8 valid=60s;
      proxy_pass https://$endpoint/;
  }
  
.. code:: nginx

  location ~ ^/(?<dest_proxy>[\w-]+)(?:/(?<path_proxy>.*))? {
      resolver 8.8.8.8 ipv6=off valid=60s;
      proxy_pass https://${dest_proxy}.example.com/${path_proxy}$is_args$args;
  }
  
Notice that nginx will start even without ``resolver`` directive, but will fail with 502 at runtime, because "no resolver defined to resolve".

Caveats
-------

.. code:: nginx

  location /api_version/ {
      proxy_pass https://api.com/version/;
  }

  location /api/ {
      set $endpoint api.com;
      resolver 8.8.8.8 valid=60s;
      proxy_pass https://$endpoint/;
  }

In this case nginx will resolve api.com once at startup with system resolver and then will never do re-resolve even for /api/ requests. *Example with /api_version/ is just synthetic example, you can use more complex scenarios with headers set, etc.*

Use variables everywhere to make it work as expected:

.. code:: nginx

  location /api_version/ {
      set $endpoint api.com;
      resolver 8.8.8.8 valid=60s;
      proxy_pass https://$endpoint/version/;
  }

  location /api/ {
      set $endpoint api.com;
      resolver 8.8.8.8 valid=60s;
      proxy_pass https://$endpoint/;
  }

You can move ``set`` and ``resolver`` to the ``server`` or ``http`` (or use ``include``) directives to avoid copy-paste (also I assume that it will increase perfomance a bit, but I haven't tested it).

Will ``upstream`` save my life?
-------------------------------

If you're using nginx plus, you can use ``resolve`` parameter, `check out documentation <http://nginx.org/en/docs/http/ngx_http_upstream_module.html#server>`_. I assume that it will be efficient, because documentation says "monitors changes of the IP addresses that correspond to a domain name of the server", while solutions listed above will query DNS on the particular request. But if you're using open source nginx, no honey is available for you. No money — no honey.

Interesting ``upstream`` behaviour
----------------------------------

.. code:: nginx
  
  server {
    listen      80;
    server_name fillo.me;

    location = /api_version/ {
        proxy_pass https://version.api.com/version/;
    }

    location ~ ^/api/(?<dest_proxy>[\w-]+)(/(?<path_proxy>.*))? {
        resolver 8.8.8.8 valid=60s;
        proxy_pass https://${dest_proxy}.api.com/${path_proxy}$is_args$args;
    }
  }

.. list-table::
   :header-rows: 1

   * - Proxy from
     - Proxy to
   * - http://fillo.me/api/[name]/[something]/[else]/
     - https://[name].api.com/[something]/[else]/
   * - http://fillo.me/api_version/
     - https://api.com/version/

* If you will open http://fillo.me/api_version/ then no resolve will be done, because of nginx resolved version.api.com at startup.
* If you will open http://fillo.me/api/version/version/ then NO resolve will be done, because of nginx resolved version.api.com at startup (described above).
* If you will open http://fillo.me/api/checkout/items/ then it will work as expected (resolve occurs).

But it can be fixed with ``upstream``:

.. code:: nginx

  upstream api_version {
      server version.example.com:443;
  }

  server {
      listen      80;
      server_name fillo.me;

      location = /api_version/ {
         proxy_pass https://api_version/version/;
         proxy_set_header Host version.api.com;
      }

      location ~ ^/api/(?<dest_proxy>[\w-]+)(?:/(?<path_proxy>.*))? {
          resolver 8.8.8.8 valid=60s;
          proxy_pass https://${dest_proxy}.example.com/${path_proxy}$is_args$args;
      }
  }

* If you will open http://fillo.me/api_version/ then no resolve will be done, because of nginx resolved version.api.com at startup.
* If you will open http://fillo.me/api/version/version/ then it will work as expected, nginx will resolve version.api.com every 60s on particular request.
* If you will open http://fillo.me/api/checkout/items/ then it will work as expected, nginx will resolve checkout.api.com every 60s on particular request.

So, very interesting can be done with ``upstream``:

.. code:: nginx

  upstream api {
      server api.com:443;
  }

  server {
      listen      80;
      server_name fillo.me;

      location /api-with-resolve/ {
         set $endpoint api.com;
         resolver 8.8.8.8 valid=1s;
         proxy_pass https://$endpoint/;
      }

      location /api-without-resolve/ {
         proxy_pass https://api/;
         proxy_set_header Host api.com;
      }
  }

Tested on
=========

* 1.9.6
* 1.10.1 (partially, without upstreams interesting behaviour)

I assume it work for wide range of versions.

Further research
================

* `This issue <https://trac.nginx.org/nginx/ticket/723>`_ says that changing HTTPS to the HTTP helps. Check how protocol changes affects examples above.
* Compare perfomance with different variables scope.
