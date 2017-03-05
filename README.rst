==================================
nginx proxy_pass resolver pitfalls
==================================

If you're using proxy_pass and your endpoint's IPs can vary in time, please read it to avoid misunderstandings about how nginx works.

.. contents::

Example
=======

.. code:: nginx

  location /api/ {
      proxy_pass http://api.com/;
  }

In this case nginx will resolve ``api.com`` only once at nginx startup. You should keep in mind some cases when your endpoint can be resolved to the any IP, e.g. you're using load balancer which doing magic failover via DNS mapping. If ``api.com`` will point to another IP your proxying will fail in the example above.

Trying to find solution
=======================

Add a resolver directive
------------------------

You can check `official nginx docs <http://nginx.org/en/docs/>`_ and find `resolver <http://nginx.org/en/docs/http/ngx_http_core_module.html#resolver>`_ directive.

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

It's because of nginx doesn't respect resolver directive in this case. It will resolve ``api.com`` only at startup by system resolver (``/etc/resolv.com``) anyway, even if TTL of ``api.com`` is 1s.

Add variables
-------------

You can google a bit and find that `nginx only tries to resolve proxy endpoint if it uses variables <https://trac.nginx.org/nginx/ticket/723>`_. Also `official doc for proxy_pass notices this too <http://nginx.org/en/docs/http/ngx_http_proxy_module.html#proxy_pass>`_. Hmmm.

Let's try:

.. code:: nginx

  location /api/ {
      set $endpoint api.com;
      resolver 8.8.8.8 valid=1s;
      proxy_pass https://$endpoint/;
  }

Works as expected. These configurations will works:

.. code:: nginx

  set $endpoint api.com;
  location /api/ {
      resolver 8.8.8.8 valid=60s;
      proxy_pass https://$endpoint/;
  }
  
.. code:: nginx

  location ~ ^/(?<dest_proxy>[\w-]+)(/(?<path_proxy>.*))? {
      resolver 8.8.8.8 ipv6=off valid=60s;
      proxy_pass https://${dest_proxy}.example.com${path_proxy}$is_args$args;
  }
  
Notice that without resolver directive in these cases nginx will start, but will fail with 502 at runtime, because "no resolver defined to resolve".

Caveats
-------

Imagine configuration:

.. code:: nginx

  location /api_version/ {
      proxy_pass https://api.com/stats/dev1/;
  }

  location /api/ {
      set $endpoint api.com;
      resolver 8.8.8.8 valid=60s;
      proxy_pass https://$endpoint/;
  }

In this case nginx will resolve ``api.com`` once at startup with system resolver and then never do re-resolve. 

Use variables everywhere to make it work as expected:

.. code:: nginx

  location /api_version/ {
      set $endpoint api.com;
      proxy_pass https://$endpoint/stats/dev1/;
  }

  location /api/ {
      set $endpoint api.com;
      resolver 8.8.8.8 valid=60s;
      proxy_pass https://$endpoint/;
  }
  

Perfomance
==========

TODO (anycast)
TODO (http)
TODO (upstream)
write about resolve
