.. include:: include/replace.rst

#############
Configuration
#############

.. _configuration-quickstart:

***********
Quick Start
***********

To run an application in Unit, first set up an :ref:`application
<configuration-applications>` object.  Let's store it in a file to :samp:`PUT`
it into the :samp:`config/applications` section of Unit's control API,
available via the :ref:`control socket <installation-startup>` at
:samp:`http://localhost/`:

.. code-block:: console

   # cat << EOF > config.json

       {
           "type": "php",
           "root": "/www/blogs/scripts"
       }
       EOF

   # curl -X PUT --data-binary @config.json --unix-socket \
          /path/to/control.unit.sock http://localhost/config/applications/blogs

       {
           "success": "Reconfiguration done."
       }

Unit starts the application process.  Next, reference the application object
from a :ref:`listener <configuration-listeners>` object, comprising an IP (or a
wildcard to match any IPs) and a port number, in the :samp:`config/listeners`
section of the API:

.. code-block:: console

   # cat << EOF > config.json

       {
           "pass": "applications/blogs"
       }
       EOF

   # curl -X PUT --data-binary @config.json --unix-socket \
          /path/to/control.unit.sock http://localhost/config/listeners/127.0.0.1:8300

       {
           "success": "Reconfiguration done."
       }

Unit accepts requests at the specified IP and port, passing them to the
application process.  Your app works!

Finally, check the resulting configuration:

.. code-block:: console

   # curl --unix-socket /path/to/control.unit.sock http://localhost/config/

       {
           "listeners": {
               "127.0.0.1:8300": {
                   "pass": "applications/blogs"
               }
           },

           "applications": {
               "blogs": {
                   "type": "php",
                   "root": "/www/blogs/scripts/"
               }
           }
       }

You can upload the entire configuration at once or update it in portions.  For
details of configuration techniques, see :ref:`below <configuration-mgmt>`.
For a full configuration sample, see :ref:`here <configuration-full-example>`.

.. _configuration-mgmt:

************************
Configuration Management
************************

Unit's configuration is JSON-based, accessed via the :ref:`control socket
<installation-startup>`, and entirely manageable over HTTP.

.. note::

   Here, we use :program:`curl` to query Unit's control API, prefixing URIs
   with :samp:`http://localhost` as expected by this utility.  You can use any
   tool capable of making HTTP requests; also, the hostname is irrelevant for
   Unit.

To address parts of the configuration, query the control socket over HTTP; URI
path segments of your requests to the API must be names of its `JSON object
members <https://tools.ietf.org/html/rfc8259#section-4>`_ or indexes of its
`array elements <https://tools.ietf.org/html/rfc8259#section-5>`_.

You can manipulate the API with the following HTTP methods:

.. list-table::
   :header-rows: 1

   * - Method
     - Action

   * - :samp:`GET`
     - Returns the entity at the request URI as JSON value in the HTTP response
       body.

   * - :samp:`POST`
     - Updates the *array* at the request URI, appending the JSON value
       from the HTTP request body.

   * - :samp:`PUT`
     - Replaces the entity at the request URI and returns status message in the
       HTTP response body.

   * - :samp:`DELETE`
     - Deletes the entity at the request URI and returns status message in the
       HTTP response body.

Before a change, Unit evaluates the difference it causes in the entire
configuration; if there's none, nothing is done. For example, you can't restart
an app by uploading the same configuration it already has.

Unit performs actual reconfiguration steps as gracefully as possible: running
tasks expire naturally, connections are properly closed, processes end
smoothly.

Any type of update can be done with different URIs, provided you supply the
right JSON:

.. code-block:: console

   # curl -X PUT -d '{ "pass": "applications/blogs" }' --unix-socket \
          /path/to/control.unit.sock http://localhost/config/listeners/127.0.0.1:8300

   # curl -X PUT -d '"applications/blogs"' --unix-socket /path/to/control.unit.sock \
          http://localhost/config/listeners/127.0.0.1:8300/pass

However, mind that the first command replaces the *entire* listener, dropping
any other options you could have configured, whereas the second one replaces
only the :samp:`pass` value and leaves other options intact.

========
Examples
========

To minimize typos and effort, avoid embedding JSON payload in your commands;
instead, consider storing your configuration snippets for review and reuse.
Suppose you save your application object as :file:`wiki.json`:

.. code-block:: json

   {
       "type": "python",
       "module": "wsgi",
       "user": "www-wiki",
       "group": "www-wiki",
       "path": "/www/wiki/"
   }

Use it to set up an application called :samp:`wiki-prod`:

.. code-block:: console

   # curl -X PUT --data-binary @/path/to/wiki.json \
          --unix-socket /path/to/control.unit.sock http://localhost/config/applications/wiki-prod

Use it again to set up a development version of the same app called
:samp:`wiki-dev`:

.. code-block:: console

   # curl -X PUT --data-binary @/path/to/wiki.json \
          --unix-socket /path/to/control.unit.sock http://localhost/config/applications/wiki-dev

Toggle the :samp:`wiki-dev` app to another source code directory:

.. code-block:: console

   # curl -X PUT -d '"/www/wiki-dev/"' \
          --unix-socket /path/to/control.unit.sock http://localhost/config/applications/wiki-dev/path

Next, boost the process count for the production app to warm it up a bit:

.. code-block:: console

   # curl -X PUT -d '5' \
          --unix-socket /path/to/control.unit.sock http://localhost/config/applications/wiki-prod/processes

Add a listener for the :samp:`wiki-prod` app to accept requests at all host
IPs:

.. code-block:: console

   # curl -X PUT -d '{ "pass": "applications/wiki-prod" }' \
          --unix-socket /path/to/control.unit.sock 'http://localhost/config/listeners/*:8400'

Plug the :samp:`wiki-dev` app into the listener to test it:

.. code-block:: console

   # curl -X PUT -d '"applications/wiki-dev"' --unix-socket /path/to/control.unit.sock \
          'http://localhost/config/listeners/*:8400/pass'

Then rewire the listener, adding a URI-based route to the development version
of the app:

.. code-block:: console

   # cat << EOF > config.json

       [
           {
               "match": {
                   "uri": "/dev/*"
               },

               "action": {
                   "pass": "applications/wiki-dev"
               }
           }
       ]
       EOF

   # curl -X PUT --data-binary @config.json --unix-socket \
          /path/to/control.unit.sock http://localhost/config/routes

   # curl -X PUT -d '"routes"' --unix-socket \
          /path/to/control.unit.sock 'http://localhost/config/listeners/*:8400/pass'

Next, let's change the :samp:`wiki-dev`'s URI prefix in the :samp:`routes`
array using its index (0):

.. code-block:: console

   # curl -X PUT -d '"/development/*"' --unix-socket=/path/to/control.unit.sock \
          http://localhost/config/routes/0/match/uri

Let's add a route to the prod app: :samp:`POST` always adds to the array end,
so there's no need for an index:

.. code-block:: console

   # curl -X POST -d '{"match": {"uri": "/production/*"}, \
          "action": {"pass": "applications/wiki-prod"}}'  \
          --unix-socket=/path/to/control.unit.sock        \
          http://localhost/config/routes/

Otherwise, use :samp:`PUT` with the array's last index (0 in our sample) *plus
one* to add the new item at the end:

.. code-block:: console

   # curl -X PUT -d '{"match": {"uri": "/production/*"}, \
          "action": {"pass": "applications/wiki-prod"}}' \
          --unix-socket=/path/to/control.unit.sock       \
          http://localhost/config/routes/1/

To get the complete :samp:`config` section:

.. code-block:: console

   # curl --unix-socket /path/to/control.unit.sock http://localhost/config/

       {
           "listeners": {
               "*:8400": {
                   "pass": "routes"
               }
           },

           "applications": {
               "wiki-dev": {
                   "type": "python",
                   "module": "wsgi",
                   "user": "www-wiki",
                   "group": "www-wiki",
                   "path": "/www/wiki-dev/"
               },

               "wiki-prod": {
                   "type": "python",
                   "processes": 5,
                   "module": "wsgi",
                   "user": "www-wiki",
                   "group": "www-wiki",
                   "path": "/www/wiki/"
               }
           },

           "routes": [
               {
                   "match": {
                       "uri": "/development/*"
                   },

                   "action": {
                       "pass": "applications/wiki-dev"
                   }
               },
               {
                   "action": {
                       "pass": "applications/wiki-prod"
                   }
               }
           ]
       }

To obtain the :samp:`wiki-dev` application object:

.. code-block:: console

   # curl --unix-socket /path/to/control.unit.sock \
          http://localhost/config/applications/wiki-dev

       {
           "type": "python",
           "module": "wsgi",
           "user": "www-wiki",
           "group": "www-wiki",
           "path": "/www/wiki-dev/"
       }

You can save JSON returned by such requests as :file:`.json` files for update
or review:

.. code-block:: console

   # curl --unix-socket /path/to/control.unit.sock \
          http://localhost/config/ > config.json

To drop the listener on :samp:`\*:8400`:

.. code-block:: console

   # curl -X DELETE --unix-socket /path/to/control.unit.sock \
          'http://localhost/config/listeners/*:8400'

Mind that you can't delete objects that other objects rely on, such as a route
still referenced by a listener:

.. code-block:: console

   # curl -X DELETE --unix-socket /var/run/unit/control.sock \
          http://localhost/config/routes

       {
           "error": "Invalid configuration.",
           "detail": "Request \"pass\" points to invalid location \"routes\"."
       }

.. _configuration-listeners:

*********
Listeners
*********

To start serving HTTP requests with Unit, define a listener in the
:samp:`config/listeners` section of the API.  A listener uniquely combines a
host IP (or a wildcard to match all host IPs) and a port that Unit binds to.

.. note::

   On Linux-based systems, wildcard listeners can't overlap with other
   listeners on the same port due to kernel-imposed limitations; for example,
   :samp:`*:8080` conflicts with :samp:`127.0.0.1:8080`.

Unit dispatches the requests it receives to :ref:`applications
<configuration-applications>` or :ref:`routes <configuration-routes>`
referenced by listeners; it also can serve requests for :ref:`static files
<configuration-static>` directly.  You can plug several listeners into one app
or route, or use a single listener for hot-swapping during testing or staging.

Available options:

.. list-table::
    :header-rows: 1

    * - Option
      - Description

    * - :samp:`pass`
      - Listener's target; possible values and respective actions:

        .. list-table::

           * - App name
             - :samp:`applications/qwk2mart`
             - Listener passes incoming requests to the
               :ref:`app <configuration-applications>`.

           * - Route name
             - :samp:`routes/route66`, :samp:`routes`
             - Listener relays incoming requests to the
               :ref:`route <configuration-routes>`.

    * - :samp:`tls`
      - SSL/TLS configuration object.  Its only option, :samp:`certificate`,
        enables secure communication via the listener; it must name a
        certificate chain that you have :ref:`configured <configuration-ssl>`
        earlier.

Here, local requests at port :samp:`8300` are passed to the :samp:`blogs` app;
all requests at :samp:`8400` follow the :samp:`main` route:

.. code-block:: json

    {
        "127.0.0.1:8300": {
            "pass": "applications/blogs",
            "tls": {
                "certificate": "blogs-cert"
            }
        },

        "*:8400": {
            "pass": "routes/main"
        }
    }


.. _configuration-routes:

******
Routes
******

Unit configuration offers a :samp:`routes` object to enable elaborate internal
routing between listeners and apps.  Listeners :samp:`pass` requests to routes
or directly to apps.  Requests are matched against route step conditions; a
request matching all conditions of a step is passed to the app or the route
that the step specifies.

In its simplest form, :samp:`routes` can be a single route array:

.. code-block:: json

   {
        "listeners": {
            "*:8300": {
                "pass": "routes"
            }
        },

        "routes": [ "simply referred to as routes" ]
   }

Another form is an object with one or more named route arrays as members:

.. code-block:: json

   {
        "listeners": {
            "*:8300": {
                "pass": "routes/main"
            }
        },

        "routes": {
            "main": [ "named route, qualified name: routes/main" ],
            "route66": [ "named route, qualified name: routes/route66" ]
        }
   }

============
Route Object
============

A route array contains step objects as elements; a request passed to a route
traverses them sequentially.

Steps have the following options:

.. list-table::
   :header-rows: 1

   * - Option
     - Description

   * - :samp:`action/pass`
     - Route's target in case of a match, identical to :samp:`pass` in a
       :ref:`listener <configuration-listeners>`.

   * - :samp:`action/share`
     - Static asset path: :samp:`/www/files/`.  In case of a match, Unit serves
       the :ref:`files <configuration-static>` at this path.

   * - :samp:`action/proxy`
     - Socket address of an HTTP server where the request is
       :ref:`proxied <configuration-routes-proxy>` upon match.

   * - :samp:`match`
     - Object that defines the step conditions.

       - If the request fits all :samp:`match` conditions in a step, the step's
         :samp:`action` is performed.

       - If the request doesn't match a condition, Unit proceeds to the next
         step of the route.

       - If the request doesn't match any steps, a 404 "Not Found" response is
         returned.

       If you omit :samp:`match`, requests are passed unconditionally; to avoid
       issues, use no more than one such step per route, placing it last.  See
       :ref:`below <configuration-routes-matching>` for condition matching
       details.

.. note::

   A route step must define exactly one of the following: :samp:`pass`,
   :samp:`share`, or :samp:`proxy`.

An example:

.. code-block:: json

   {
       "routes": [
           {
               "match": {
                   "host": "example.com",
                   "scheme": "https",
                   "uri": "/php/*"
               },

               "action": {
                   "pass": "applications/php_version"
                }
           },
           {
               "action": {
                   "share": "/www/static_version/"
                }
           }
        ]
   }

A more elaborate example with chained routes and proxying:

.. code-block:: json

   {
       "routes": {
           "main": [
               {
                   "match": {
                       "scheme": "http"
                   },

                   "action": {
                       "pass": "routes/http_site"
                   }
               },
               {
                   "match": {
                       "host": "blog.example.com"
                   },

                   "action": {
                       "pass": "applications/blog"
                   }
               },
               {
                   "action": {
                       "share": "/www/static_fallthrough/"
                   }
               }
           ],

           "http_site": [
               {
                   "match": {
                       "uri": "/v2_site/*"
                   },

                   "action": {
                       "pass": "applications/v2_site"
                   }
               },
               {
                   "action": {
                       "proxy": "http://127.0.0.1:9000"
                   }
               }
           ]
       }
   }

.. _configuration-routes-proxy:

========
Proxying
========

Unit routes support HTTP proxying; provide the target socket address in the
step's :samp:`action/proxy` option:

.. code-block:: json

    {
       "routes": [
           {
               "match": {
                   "uri": "/ipv4/*"
               },

               "action": {
                   "proxy": "http://127.0.0.1:8080"
                }
           },

           {
               "match": {
                   "uri": "/ipv6/*"
               },

               "action": {
                   "proxy": "http://[::1]:8090"
                }
           },

           {
               "match": {
                   "uri": "/unix/*"
               },

               "action": {
                   "proxy": "http://unix:/path/to/unix.sock"
                }
           }
        ]
    }

As the example above suggests, you can use Unix, IPv4, and IPv6 socket
addresses.

.. _configuration-routes-matching:

==================
Condition Matching
==================

To route incoming requests, Unit applies pattern-based conditions to individual
request properties:

.. list-table::
   :header-rows: 1

   * - Option
     - Description
     - Case |-| Sensitive
   * - :samp:`arguments`
     - Parameter arguments supplied in the request URI.
     - Yes
   * - :samp:`cookies`
     - Cookies supplied with the request.
     - Yes
   * - :samp:`destination`
     - Target IP address and optional port of the request.
     - No
   * - :samp:`headers`
     - Header fields supplied with the request.
     - No
   * - :samp:`host`
     - Host from the :samp:`Host` header field without port number, normalized
       by removing the trailing period (if any).
     - No
   * - :samp:`method`
     - Method from the request line.
     - No
   * - :samp:`scheme`
     - URI `scheme
       <https://www.iana.org/assignments/uri-schemes/uri-schemes.xhtml>`_.
       Currently, :samp:`http` and :samp:`https` are supported.
     - No
   * - :samp:`source`
     - Source IP address and optional port of the request.
     - No
   * - :samp:`uri`
     - URI path without arguments, normalized by decoding the "%XX" sequences,
       resolving relative path references ("." and ".."), and compressing
       adjacent slashes into one.
     - Yes

For :samp:`host`, :samp:`method`, and :samp:`uri`, :ref:`simple matching
<configuration-routes-simple>` is used; :samp:`destination` and :samp:`source`
employ :ref:`address matching <configuration-routes-address>`, accepting single
values or arrays of values; other properties rely on :ref:`compound matching
<configuration-routes-compound>`.

.. note::

   The :samp:`scheme` option value can either be :samp:`http` or :samp:`https`.

.. _configuration-routes-simple:

Simple Matching
***************

A simple property in a :samp:`match` object is matched against a string pattern
or an array of patterns:

.. code-block:: json

   {
       "match": {
           "simple_property1": "pattern",
           "simple_property2": ["pattern", "pattern", "..." ]
       },

       "action": {
           "pass": "..."
        }
   }

To be a match against the condition, the property must meet two requirements:

- If there are patterns without negation, at least one of them matches the
  property value.

- No negation-based patterns match the property value.

Patterns must match the value symbol by symbol, except for wildcards
(:samp:`*`) and negations (:samp:`!`):

- A wildcard matches zero or more arbitrary characters; wildcards can only
  :samp:`*prefix` exact patterns, :samp:`suffix*` them, :samp:`*enclose*` them,
  or :samp:`split*them` in two.

- A negation rejects all matches to the remainder of the pattern; pattern can
  only start with it.

.. note::

   This type of matching can be explained with set operations.  Suppose set *U*
   comprises all possible values of a property; set *P* comprises strings that
   match any patterns without negation; set *N* comprises strings that match
   any negation-based patterns.  In this scheme, the matching set will be:

   | *U* ∩ *P* \\ *N* if *P* ≠ ∅
   | *U* \\ *N* if *P* = ∅

A few examples:

.. code-block:: json

   {
       "host": "*.example.com"
   }

Only subdomains of :samp:`example.com` will match.

.. code-block:: json

   {
       "host": ["eu-*.example.com", "!eu-5.example.com"]
   }

Here, any :samp:`eu-` subdomains of :samp:`example.com` will match except
:samp:`eu-5.example.com`.

.. code-block:: json

   {
       "method": ["!HEAD", "!GET"]
   }

Any methods will match except :samp:`HEAD` and :samp:`GET`.

You can also combine special characters in a pattern:

.. code-block:: json

   {
       "uri": "!*/api/*"
   }

Here, any URIs will match except the ones containing :samp:`/api/`.

.. _configuration-routes-address:

Address Matching
****************

Unit uses special logic to match IP addresses and address ranges against the
:samp:`source` and :samp:`destination` options in routes.  Supported matching
types, including wildcards, exact addresses, CIDR notation, and address ranges,
are outlined below.  Both :samp:`source` and :samp:`destination` accept
individual patterns and arrays of patterns; any matching pattern in an array
makes the entire condition a match.

#. Single wildcard: matches any IPs, defines an individual port number or a
   port range:

   .. code-block:: json

      {
          "match": {
              "source": "*:0-65535",
              "destination": "*:8000"
          }
      }

   No other wildcard combinations (partial matches, multiple wildcards) are
   supported.

#. Exact address: matches specific IP addresses, optionally defining matching
   ports or port ranges.

   IPv4:

   .. code-block:: json

      {
          "match": {
              "source": [
                  "127.0.0.1",
                  "192.168.0.10:8080",
                  "192.168.0.11:8080-8090"
              ]
          }
      }

   IPv6:

   .. code-block:: json

      {
          "match": {
              "destination": [
                  "2001::",
                  "[2002::]:8000",
                  "[2003::]:8080-8090"
              ]
          }
      }

#. CIDR address range: matches IP ranges in CIDR notation, optionally defining
   matching ports or port ranges.

   IPv4:

   .. code-block:: json

      {
          "match": {
              "destination": [
                  "10.0.0.0/8",
                  "10.0.0.0/7:1000",
                  "10.0.0.0/32:8080-8090"
              ]
          }
      }

   IPv6:

   .. code-block:: json

      {
          "match": {
              "source": [
                   "2001::/16",
                   "[0ff::/64]:8000",
                   "[fff0:abcd:ffff:ffff:ffff::/128]:8080-8090"
               ]
          }
      }

#. Address range: matches ranges of IP addresses, optionally defining matching
   ports or port ranges.

   IPv4:

   .. code-block:: json

      {
          "match": {
              "destination": [
                   "10.0.0.0-10.0.0.10",
                   "10.0.0.100-11.0.0.100:1000",
                   "127.0.0.100-127.0.0.255:8080-8090"
              ]
          }
      }

   IPv6:

   .. code-block:: json

      {
          "match": {
              "source": [
                   "2001::-200f:ffff:ffff:ffff:ffff:ffff:ffff:ffff",
                   "[fe08::-feff::]:8000",
                   "[fff0::-fff0::10]:8080-8090"
              ]
          }
      }

Finally, prepend an address-based pattern with the '!' character to negate it:

   .. code-block:: json

      {
          "match": {
              "destination": "!10.0.0.0-10.0.0.100",
              "source": "!*:8000"
          }
      }

The condition above excludes requests that target any IPs within the
:samp:`10.0.0.0-10.0.0.100` range *or* issued from port 8000.

.. _configuration-routes-compound:

Compound Matching
*****************

This type of matching is used for :samp:`arguments`, :samp:`cookies`, and
:samp:`headers` properties.

A compound property is matched against an object with names and patterns or an
array of such objects:

.. code-block:: json

   {
       "match": {
           "compound_property1": {
               "name1": "pattern",
               "name2": ["pattern", "..."]
           },

           "compound_property2": [
               {
                   "name1": "pattern",
                   "name2": ["pattern", "pattern", "..."]
               },

               {
                   "name1": "pattern",
                   "name3": ["pattern", "pattern", "..."]
               }
           ]
       },

       "action": {
           "pass": "..."
       }
   }

To match a single condition object, the request must contain *all* items
explicitly named in the object; their values are matched against patterns in
the same manner as property values during :ref:`simple matching
<configuration-routes-simple>`.

To match an object array, it's sufficient to match *any* single one of its
objects.

A few examples:

.. code-block:: json

   {
       "arguments": {
           "mode": "strict",
           "access": "!full"
       }
   }

This requires :samp:`mode=strict` and any :samp:`access` argument other than
:samp:`access=full` in the URI.

.. code-block:: json

   {
       "headers": [
           {
               "Accept-Encoding": "*gzip*",
               "User-Agent": "Mozilla/5.0*"
           },

           {
               "User-Agent": "curl*"
           }
       ]
   }

This matches all requests that either use :samp:`gzip` and identify as
:samp:`Mozilla/5.0` or list :program:`curl` as the user agent.

.. note::

   You can combine simple, compound, and address matching in a :samp:`match`
   condition.

================
Passing Requests
================

To match a step, the request must fit *all* property conditions listed in it.

If all properties match or :samp:`match` is omitted, Unit routes the request
using the respective :samp:`action`:

.. code-block:: json

   {
      "routes": [
          {
              "match": {
                  "host": [ "*.example.com", "!static.example.com" ],
                  "uri": [ "/admin/*", "/store/*" ],
                  "scheme": "https",
                  "source": "*:8000-9000",
                  "method": "POST"
              },

              "action": {
                  "pass": "applications/php5_app"
              }
          },
          {
              "action": {
                  "share": "/www/static_site/"
              }
          }
      ]
   }

Here, all :samp:`POST` requests issued from ports 8000-9000 for HTTPS-schemed
URIs prefixed with :samp:`/admin/` or :samp:`/store/` within subdomains of
:samp:`example.com` (except for :samp:`static.example.com`) are routed to
:samp:`php5_app`; any other requests are served with static content at
:file:`/www/static_site/`.

.. _configuration-static:

************
Static Files
************

Unit is capable of acting as a standalone web server, serving requests for
static assets from directories you configure; to use the feature, supply the
directory path in the :samp:`share` option of a route step:

.. code-block:: json

   {
       "listeners": {
           "127.0.0.1:8300": {
               "pass": "routes"
           }
        },

       "routes": [
           {
               "action": {
                   "share": "/www/data/static/"
                }
           }
       ]
   }

Suppose the :file:`/www/data/static/` directory has the following structure:

.. code-block:: none

   /www/data/static/
   ├── stylesheet.css
   ├── html
   │   └──index.html
   └── js files
       └──page.js

In the above configuration, you can request specific files by these URIs:

.. code-block:: console

   $ curl 127.0.0.1:8300/html/index.html
   $ curl 127.0.0.1:8300/stylesheet.css
   $ curl '127.0.0.1:8300/js files/page.js'

.. note::

   Unit supports encoded symbols in URIs as the last query above suggests.

If your query specifies only the directory name, Unit will attempt to serve
:file:`index.html` from this directory:

.. subs-code-block:: console

   $ curl -vL 127.0.0.1:8300/html/

    ...
    < HTTP/1.1 200 OK
    < Last-Modified: Fri, 20 Sep 2019 04:14:43 GMT
    < ETag: "5d66459d-d"
    < Content-Type: text/html
    < Server: Unit/|version|
    ...

.. note::

   Unit's ETag response header fields use the following format:
   :samp:`%MTIME_HEX%-%FILESIZE_HEX%`.

Unit maintains a number of :ref:`built-in MIME types <configuration-mime>` like
:samp:`text/plain` or :samp:`text/html`; also, you can add extra types and
override built-ins in the :samp:`/config/settings/http/static/mime_types`
section.

========
Examples
========

One common use case that this feature enables is the separation of requests for
static and dynamic content into independent routes.  The following
example relays all requests that target :file:`.php` files to an application
and uses a static :samp:`share` as the catch-all fallback:

.. code-block:: json

   {
       "routes": [
           {
               "match": {
                   "uri": "*.php"
               },
               "action": {
                   "pass": "applications/php-app"
               }
           },
           {
               "action": {
                   "share": "/www/php-app/assets/files/"
               }
           }

       ],

       "applications": {
           "php-app": {
               "type": "php",
               "root": "/www/php-app/scripts/"
           }
       }
   }

You can reverse this scheme for apps that avoid filenames in dynamic URIs,
listing all types of static content to be served from a :samp:`share` in a
:samp:`match` condition and adding an unconditional fallback application path:

.. code-block:: json

   {
       "routes": [
           {
               "match": {
                   "uri": [
                       "*.css",
                       "*.ico",
                       "*.jpg",
                       "*.js",
                       "*.png",
                       "*.xml"
                   ]
               },
               "action": {
                   "share": "/www/php-app/assets/files/"
               }
           },
           {
               "action": {
                   "pass": "applications/php-app"
               }
           }

       ],

       "applications": {
           "php-app": {
               "type": "php",
               "root": "/www/php-app/scripts/"
           }
       }
   }

.. _configuration-applications:

************
Applications
************

Each app that Unit runs is defined as an object in the
:samp:`config/applications` section of the control API; it lists the app's
language and settings, its runtime limits, process model, and various
language-specific options.

.. note::

   Our official :ref:`language support packages <installation-precomp-pkgs>`
   include end-to-end examples of application configuration, available for your
   reference at :file:`/usr/share/doc/<module name>/examples/` after package
   installation.

Here, Unit runs 20 processes of a PHP app called :samp:`blogs`, stored in
the :file:`/www/blogs/scripts/` directory:

.. code-block:: json

   {
       "blogs": {
           "type": "php",
           "processes": 20,
           "root": "/www/blogs/scripts/"
       }
   }

.. _configuration-apps-common:

App objects have a number of options shared between all application languages:

.. list-table::
    :header-rows: 1

    * - Option
      - Description

    * - :samp:`type` (required)
      - Application type: :samp:`external` (Go and Node.js), :samp:`java`,
        :samp:`perl`, :samp:`php`, :samp:`python`, or :samp:`ruby`.

        Except with :samp:`external`, you can detail the runtime version:
        :samp:`"type": "python 3"`, :samp:`"type": "python 3.4"`, or even
        :samp:`"type": "python 3.4.9rc1"`.  Unit searches its modules and uses
        the latest matching one, reporting an error if none match.

        For example, if you have only one PHP module, 7.1.9, it matches
        :samp:`"php"`, :samp:`"php 7"`, :samp:`"php 7.1"`, and :samp:`"php
        7.1.9"`.  If you have modules for versions 7.0.2 and 7.0.23, set
        :samp:`"type": "php 7.0.2"` to specify the former; otherwise, PHP |_|
        7.0.23 will be used.

    * - :samp:`limits`
      - Object that accepts two integer options, :samp:`timeout` and
        :samp:`requests`.  Their values govern the life cycle of an
        application process.  For details, see
        :ref:`here <configuration-proc-mgmt-lmts>`.

    * - :samp:`processes`
      - Integer or object.  Integer sets a static number of app processes;
        object options :samp:`max`, :samp:`spare`, and :samp:`idle_timeout`
        enable dynamic management.  For details, see :ref:`here
        <configuration-proc-mgmt-prcs>`.

        The default value is 1.

    * - :samp:`working_directory`
      - The app's working directory.  If not set, the Unit daemon's working
        directory is used.

    * - :samp:`user`
      - Username that runs the app process.  If not set, :samp:`nobody` is
        used.

    * - :samp:`group`
      - Group name that runs the app process.  If not set, the user's primary
        group is used.

    * - :samp:`environment`
      - Environment variables to be passed to the application.

Also, you need to set :samp:`type`-specific :ref:`options
<configuration-languages>` to run the app.  This :ref:`Python app
<configuration-python>` uses :samp:`path` and :samp:`module`:

.. code-block:: json

   {
       "type": "python 3.6",
       "processes": 16,
       "working_directory": "/www/python-apps",
       "path": "blog",
       "module": "blog.wsgi",
       "user": "blog",
       "group": "blog",
       "environment": {
           "DJANGO_SETTINGS_MODULE": "blog.settings.prod",
           "DB_ENGINE": "django.db.backends.postgresql",
           "DB_NAME": "blog",
           "DB_HOST": "127.0.0.1",
           "DB_PORT": "5432"
       }
   }

==================
Process Management
==================

Unit supports three per-app options that control the app's processes:
:samp:`isolation`, :samp:`limits`, and :samp:`processes`.

.. _configuration-proc-mgmt-isolation:

Process Isolation
*****************

You can use namespace isolation `features
<http://man7.org/linux/man-pages/man7/namespaces.7.html>`_ for your apps if
Unit's underlying OS supports them:

.. code-block:: console

   $ ls /proc/self/ns/

       cgroup  ipc  mnt  net  pid  ...  user  uts

The :samp:`isolation` application option has the following members:

.. list-table::
   :header-rows: 1

   * - Option
     - Description

   * - :samp:`namespaces`
     - Object that configures namespace isolation scheme for the application.

       Available options (system-dependent; check your OS manual for guidance):

       .. list-table::

          * - :samp:`cgroup`
            - Creates a new `cgroup
              <http://man7.org/linux/man-pages/man7/cgroup_namespaces.7.html>`_
              namespace for the app.

          * - :samp:`credential`
            - Creates a new `user
              <http://man7.org/linux/man-pages/man7/user_namespaces.7.html>`_
              namespace for the app.

          * - :samp:`mount`
            - Creates a new `mount
              <http://man7.org/linux/man-pages/man7/mount_namespaces.7.html>`_
              namespace for the app.

          * - :samp:`network`
            - Creates a new `network
              <http://man7.org/linux/man-pages/man7/network_namespaces.7.html>`_
              namespace for the app.

          * - :samp:`pid`
            - Creates a new `PID
              <http://man7.org/linux/man-pages/man7/pid_namespaces.7.html>`_
              namespace for the app.

          * - :samp:`uname`
            - Creates a new `UTS
              <http://man7.org/linux/man-pages/man7/namespaces.7.html>`_
              namespace for the app.

       All options listed above are Boolean; to isolate the app, set the
       corresponding namespace option to :samp:`true`; to disable isolation,
       set the option to :samp:`false` (default).

   * - :samp:`uidmap`
     - Array of `ID mapping
       <http://man7.org/linux/man-pages/man7/user_namespaces.7.html>`_ objects;
       each array item must define the following:

       .. list-table::

          * - :samp:`container`
            - Integer that starts the user ID mapping range in the app's
              namespace.

          * - :samp:`host`
            - Integer that starts the user ID mapping range in the OS
              namespace.

          * - :samp:`size`
            - Integer size of the ID range in both namespaces.

   * - :samp:`gidmap`
     - Same as :samp:`uidmap`, but configures group IDs instead of user IDs.

.. note::

   The :samp:`uidmap` and :samp:`gidmap` options are available only if the OS
   supports user namespaces.

Example of an :samp:`isolation` object that enables all namespaces and sets
mappings for user and group IDs:

.. code-block:: json

    {
        "namespaces": {
            "cgroup": true,
            "credential": true,
            "mount": true,
            "network": true,
            "pid": true,
            "uname": true
        },

        "uidmap": [
            {
                "host": 1000,
                "container": 0,
                "size": 1000
            }
        ],

        "gidmap": [
            {
                "host": 1000,
                "container": 0,
                "size": 1000
            }
        ]
    }

.. _configuration-proc-mgmt-lmts:

Request Limits
**************

The :samp:`limits` object controls request handling by the app process and has
two integer options:

.. list-table::
   :header-rows: 1

   * - Option
     - Description

   * - :samp:`timeout`
     - Request timeout in seconds.  If an app process exceeds this limit while
       handling a request, Unit alerts it to cancel the request and returns an
       HTTP error to the client.

   * - :samp:`requests`
     - Maximum number of requests Unit allows an app process to serve.  If the
       limit is reached, the process is restarted; this helps to mitigate
       possible memory leaks or other cumulative issues.

Example:

.. code-block:: json

   {
       "type": "python",
       "working_directory": "/www/python-apps",
       "module": "blog.wsgi",
       "limits": {
           "timeout": 10,
           "requests": 1000
       }
   }

.. _configuration-proc-mgmt-prcs:

Process Management
******************

The :samp:`processes` option offers a choice between static and dynamic process
management.  If you set it to an integer, Unit immediately launches the given
number of app processes and keeps them without scaling.

To enable dynamic prefork model for your app, supply a :samp:`processes` object
with the following options:

.. list-table::
    :header-rows: 1

    * - Option
      - Description

    * - :samp:`max`
      - Maximum number of application processes that Unit will maintain
        (busy and idle).

        The default value is 1.

    * - :samp:`spare`
      - Minimum number of idle processes that Unit tries to reserve for an app.
        When the app is started, :samp:`spare` idle processes are launched;
        Unit assigns incoming requests to existing idle processes, forking new
        idles to maintain the :samp:`spare` level if :samp:`max` allows.  As
        processes complete requests and turn idle, Unit terminates extra ones
        after :samp:`idle_timeout`.

    * - :samp:`idle_timeout`
      - Time in seconds that Unit waits before terminating an idle process
        which exceeds :samp:`spare`.

If :samp:`processes` is omitted entirely, Unit creates 1 static process.  If
an empty object is provided: :samp:`"processes": {}`, dynamic behavior with
default option values is assumed.

Here, Unit allows 10 processes maximum, keeps 5 idles, and terminates extra
idles after 20 seconds:

.. code-block:: json

   {
       "max": 10,
       "spare": 5,
       "idle_timeout": 20
   }

.. _configuration-languages:
.. _configuration-external:

==========
Go/Node.js
==========

To run your Go or Node.js applications in Unit, you need to configure them
`and` modify their source code as suggested below.  Let's start with the app
configuration; besides :ref:`common options <configuration-apps-common>`, you
have the following:

.. list-table::
    :header-rows: 1

    * - Option
      - Description

    * - :samp:`executable` (required)
      - Pathname of the application, absolute or relative to
        :samp:`working_directory`.

        For Node.js, supply your :file:`.js` pathname and start the file itself
        with a proper shebang:

        .. code-block:: javascript

           #!/usr/bin/env node

        .. note::

           Make sure to :command:`chmod +x` the file you list here so Unit can
           start it.

    * - :samp:`arguments`
      - Command line arguments to be passed to the application.
        The example below is equivalent to
        :samp:`/www/chat/bin/chat_app --tmp-files /tmp/go-cache`.

Example:

.. code-block:: json

   {
       "type": "external",
       "working_directory": "/www/chat",
       "executable": "bin/chat_app",
       "user": "www-go",
       "group": "www-go",
       "arguments": ["--tmp-files", "/tmp/go-cache"]
   }

Before applying the configuration, update the application itself.

.. _configuration-external-go:

Modifying Go Sources
********************

In the :samp:`import` section, reference the :samp:`"unit.nginx.org/go"` package
that you have :ref:`installed <installation-precomp-pkgs>` or :ref:`built
<installation-go>` earlier:

.. code-block:: go

   import (
       ...
       "unit.nginx.org/go"
       ...
   )

.. note::

   The package is required only to build the app; there's no need to install it
   in the target environment.

In the :samp:`main()` function, replace the :samp:`http.ListenandServe` call
with :samp:`unit.ListenAndServe`:

.. code-block:: go

   func main() {
       ...
       http.HandleFunc("/", handler)
       ...
       //http.ListenAndServe(":8080", nil)
       unit.ListenAndServe(":8080", nil)
       ...
   }

The resulting application works as follows:

- When you run it standalone, the :samp:`unit.ListenAndServe` call falls back
  to :samp:`http` functionality.
- When Unit runs it, :samp:`unit.ListenAndServe` communicates with Unit's
  router process directly, ignoring the address supplied as its first argument
  and relying on the :ref:`listener's settings <configuration-listeners>`
  instead.

.. note::

   For an example of Go app configuration, see our :doc:`Grafana
   <howto/grafana>` howto.  Also, see a :ref:`sample <sample-go>` in Go.

.. _configuration-external-nodejs:

Modifying Node.js Sources
*************************

First, you need to have the :program:`unit-http` module :ref:`installed
<installation-nodejs-package>`.  If it's global, symlink it in your project
directory:

.. code-block:: console

   # npm link unit-http

Do the same if you move a Unit-hosted application to a new system where
:program:`unit-http` is installed globally.

Next, use :samp:`unit-http` instead of :samp:`http` in your code:

.. code-block:: javascript

   var http = require('unit-http');

Unit also supports the WebSocket protocol; your Node.js app only needs to
replace the default :samp:`websocket`:

.. code-block:: javascript

  var webSocketServer = require('unit-http/websocket').server;

.. note::

   For examples of Node.js app configuration, see our :doc:`Express
   <howto/express>` and :ref:`Docker <docker-apps>` howtos or a
   :ref:`sample <sample-nodejs>` in Node.js.

.. _configuration-java:

====
Java
====

First, make sure to install Unit along with the :ref:`Java language module
<installation-precomp-pkgs>`.

Besides :ref:`common options <configuration-apps-common>`, you have the
following:

.. list-table::
    :header-rows: 1

    * - Option
      - Description

    * - :samp:`webapp` (required)
      - Pathname of the application's packaged or unpackaged :file:`.war` file.

    * - :samp:`classpath`
      - Array of paths to your app's required libraries (may point to
        directories or :file:`.jar` files).

    * - :samp:`options`
      - Array of strings defining JVM runtime options.

Example:

.. code-block:: json

   {
       "type": "java",
       "classpath": ["/www/qwk2mart/lib/qwk2mart-2.0.0.jar"],
       "options": ["-Dlog_path=/var/log/qwk2mart.log"],
       "webapp": "/www/qwk2mart/qwk2mart.war"
   }

.. note::

   For an example of Java app configuration, see our :doc:`Jira <howto/jira>`
   howto.  Also, see a JSP :ref:`sample <sample-java>`.

.. _configuration-perl:

====
Perl
====

First, make sure to install Unit along with the :ref:`Perl language module
<installation-precomp-pkgs>`.

Besides :ref:`common options <configuration-apps-common>`, you have the
following:

.. list-table::
    :header-rows: 1

    * - Option
      - Description

    * - :samp:`script` (required)
      - PSGI script path.

Example:

.. code-block:: json

   {
       "type": "perl",
       "script": "/www/bugtracker/app.psgi",
       "working_directory": "/www/bugtracker",
       "processes": 10,
       "user": "www",
       "group": "www"
   }

.. note::

   For an example of Perl app configuration, see our :doc:`Bugzilla
   <howto/bugzilla>` and :doc:`Catalyst <howto/catalyst>` howtos or a
   :ref:`sample <sample-perl>` in Perl.

.. _configuration-php:

===
PHP
===

First, make sure to install Unit along with the :ref:`PHP language module
<installation-precomp-pkgs>`.

Besides :ref:`common options <configuration-apps-common>`, you have the
following:

.. list-table::
    :header-rows: 1

    * - Option
      - Description

    * - :samp:`root` (required)
      - Base directory of your PHP app's file structure.  All URI paths are
        relative to this value.

    * - :samp:`index`
      - Filename appended to any URI paths ending with a slash; applies if
        :samp:`script` is omitted.

        The default value is :samp:`index.php`.

    * - :samp:`options`
      - Object that defines :file:`php.ini` location and options.  For details,
        see below.

    * - :samp:`script`
      - Filename of a :samp:`root`-based PHP script that Unit uses to serve all
        requests to the app.

The :samp:`index` and :samp:`script` options enable two modes of operation:

- If :samp:`script` is set, all requests to the application are handled by
  the script you provide.

- Otherwise, the requests are served according to their URI paths; if script
  name is omitted, :samp:`index` is used.

You can customize :file:`php.ini` via the :samp:`options` object:

.. list-table::
    :header-rows: 1

    * - Option
      - Description

    * - :samp:`file`
      - Pathname of the :file:`php.ini` file with `PHP configuration directives
        <https://php.net/manual/en/ini.list.php>`_.

    * - :samp:`admin`, :samp:`user`
      - Objects for extra directives.  Values in :samp:`admin` are set in
        :samp:`PHP_INI_SYSTEM` mode, so the app can't alter them; :samp:`user`
        values are set in :samp:`PHP_INI_USER` mode and may `be updated
        <https://php.net/manual/en/function.ini-set.php>`_ in runtime.

Directives from :file:`php.ini` are overridden by settings supplied in
:samp:`admin` and :samp:`user` objects.

.. note::

   Values in :samp:`options` must be strings (for example,
   :samp:`"max_file_uploads": "4"`, not :samp:`"max_file_uploads": 4`); for
   boolean flags, use :samp:`"0"` and :samp:`"1"` only.  For details about
   :samp:`PHP_INI_*` modes, see the `PHP docs
   <https://php.net/manual/en/configuration.changes.modes.php>`_.

Example:

.. code-block:: json

   {
       "type": "php",
       "processes": 20,
       "root": "/www/blogs/scripts/",
       "user": "www-blogs",
       "group": "www-blogs",

       "options": {
           "file": "/etc/php.ini",
           "admin": {
               "memory_limit": "256M",
               "variables_order": "EGPCS",
               "expose_php": "0"
           },
           "user": {
               "display_errors": "0"
           }
       }
   }

.. note::

   For an example of PHP app configuration, see our :doc:`WordPress
   <howto/wordpress>` howto.  Also, see a :ref:`sample
   <sample-php>` in PHP.

.. _configuration-python:

======
Python
======

First, make sure to install Unit along with the :ref:`Python language module
<installation-precomp-pkgs>`.

Besides :ref:`common options <configuration-apps-common>`, you have the
following:

.. list-table::
    :header-rows: 1

    * - Option
      - Description

    * - :samp:`module` (required)
      - `WSGI <https://www.python.org/dev/peps/pep-3333/>`_ module name.  To
        run the app, Unit looks for an :samp:`application` callable in the
        module you supply; the :samp:`module` itself is `imported
        <https://docs.python.org/3/reference/import.html>`_ just like in
        Python.

    * - :samp:`path`
      - Additional lookup path for Python modules; this string is inserted into
        :samp:`sys.path`.

    * - :samp:`home`
      - Path to Python's `virtual environment <https://packaging.python.org/
        tutorials/installing-packages/#creating-virtual-environments>`_ for the
        app.  Absolute or relative to :samp:`working_directory`.


        .. note::

           The Python version used to run the app depends on the :samp:`type`
           value; Unit ignores the command-line interpreter from the virtual
           environment for performance considerations.

Example:

.. code-block:: json

   {
       "type": "python",
       "processes": 10,
       "working_directory": "/www/store/",
       "path": "/www/store/cart/",
       "home": "/www/store/.virtualenv/",
       "module": "wsgi",
       "user": "www",
       "group": "www"
   }

.. note::

   For examples of Python app configuration, see our :doc:`Django
   <howto/django>` and :doc:`Flask <howto/flask>` howtos or a
   :ref:`sample <sample-python>` in Python.

.. _configuration-ruby:

====
Ruby
====

First, make sure to install Unit along with the :ref:`Ruby language module
<installation-precomp-pkgs>`.

.. note::

   Unit uses the `Rack <https://rack.github.io>`_ interface to run Ruby
   scripts; you need to have it installed as well:

   .. code-block:: console

      $ gem install rack

Besides :ref:`common options <configuration-apps-common>`, you have the
following:

.. list-table::
    :header-rows: 1

    * - Option
      - Description

    * - :samp:`script` (required)
      - Rack script pathname, including the :file:`.ru` extension:
        :file:`/www/rubyapp/script.ru`.

Example:

.. code-block:: json

   {
       "type": "ruby",
       "processes": 5,
       "user": "www",
       "group": "www",
       "script": "/www/cms/config.ru"
   }

.. note::

   For an example of Ruby app configuration, see our :doc:`Redmine
   <howto/redmine>` howto.  Also, see a :ref:`sample
   <sample-ruby>` in Ruby.

.. _configuration-stngs:

********
Settings
********

Unit has a global :samp:`settings` configuration object that stores
instance-wide preferences.  Its :samp:`http` option fine-tunes the handling of
HTTP requests from the clients:

.. list-table::
    :header-rows: 1

    * - Option
      - Description

    * - :samp:`header_read_timeout`
      - Maximum number of seconds to read the header of a client's request.
        If Unit doesn't receive the entire header from the client within this
        interval, it responds with a 408 Request Timeout error.

        The default value is 30.

    * - :samp:`body_read_timeout`
      - Maximum number of seconds to read data from the body of a client's
        request.  It limits the interval between consecutive read operations,
        not the time to read the entire body.  If Unit doesn't receive any
        data from the client within this interval, it responds with a 408
        Request Timeout error.

        The default value is 30.

    * - :samp:`send_timeout`
      - Maximum number of seconds to transmit data in the response to a client.
        It limits the interval between consecutive transmissions, not the
        entire response transmission.  If the client doesn't receive any data
        within this interval, Unit closes the connection.

        The default value is 30.

    * - :samp:`idle_timeout`
      - Maximum number of seconds between requests in a keep-alive connection.
        If no new requests arrive within this interval, Unit responds with a
        408 Request Timeout error and closes the connection.

        The default value is 180.

    * - :samp:`max_body_size`
      - Maximum number of bytes in the body of a client's request.  If the body
        size exceeds this value, Unit responds with a 413 Payload Too Large
        error and closes the connection.

        The default value is 8388608 (8 MB).
    * - :samp:`static`
      - An object that configures static asset handling, containing a single
        object named :samp:`mime_types`.  In turn, :samp:`mime_types`
        defines specific MIME types as options.  An option's value can be a
        string or an array of strings; each string must specify a filename
        extension or a specific filename that is included in the MIME type.

Example:

.. code-block:: json

   {
       "settings": {
           "http": {
               "header_read_timeout": 10,
               "body_read_timeout": 10,
               "send_timeout": 10,
               "idle_timeout": 120,
               "max_body_size": 6291456,
               "static": {
                   "mime_types": {
                       "text/plain": [
                            ".log",
                            "README",
                            "CHANGES"
                       ]
                   }
               }
           }
       }
   }

.. _configuration-mime:

.. note::

   Built-in support for MIME types includes :file:`.aac`, :file:`.atom`,
   :file:`.avi`, :file:`.bin`, :file:`.css`, :file:`.deb`, :file:`.dll`,
   :file:`.exe`, :file:`.flac`, :file:`.gif`, :file:`.htm`, :file:`.html`,
   :file:`.ico`, :file:`.img`, :file:`.iso`, :file:`.jpeg`, :file:`.jpg`,
   :file:`.js`, :file:`.json`, :file:`.md`, :file:`.mid`, :file:`.midi`,
   :file:`.mp3`, :file:`.mp4`, :file:`.mpeg`, :file:`.mpg`, :file:`.msi`,
   :file:`.ogg`, :file:`.otf`, :file:`.pdf`, :file:`.png`, :file:`.rpm`,
   :file:`.rss`, :file:`.rst`, :file:`.svg`, :file:`.ttf`, :file:`.txt`,
   :file:`.wav`, :file:`.webm`, :file:`.webp`, :file:`.woff2`, :file:`.woff`,
   :file:`.xml`, and :file:`.zip`.  Built-ins can be overridden, and new types
   can be added:

   .. code-block:: console

      # curl -X PUT -d '{"text/x-code": [".c", ".h"]}' /path/to/control.unit.sock \
             http://localhost/config/settings/http/static/mime_types
      {
             "success": "Reconfiguration done."
      }


.. _configuration-access-log:

**********
Access Log
**********

To enable access logging, specify the log file path in the :samp:`access_log`
option of the :samp:`config` object.

In the example below, all requests will be logged to
:file:`/var/log/access.log`:

.. code-block:: console

   # curl -X PUT -d '"/var/log/access.log"' \
          --unix-socket /path/to/control.unit.sock \
          http://localhost/config/access_log

       {
           "success": "Reconfiguration done."
       }

The log is written in the Combined Log Format.  Example of a log line:

.. code-block:: none

   127.0.0.1 - - [21/Oct/2015:16:29:00 -0700] "GET / HTTP/1.1" 200 6022 "http://example.com/links.html" "Godzilla/5.0 (X11; Minix i286) Firefox/42"

.. _configuration-ssl:

************************
SSL/TLS and Certificates
************************

To set up SSL/TLS access for your application, upload a :file:`.pem` file
containing your certificate chain and private key to Unit.  Next, reference the
uploaded bundle in the listener's configuration.  After that, the listener's
application becomes accessible via SSL/TLS.

First, create a :file:`.pem` file with your certificate chain and private key:

.. code-block:: console

   $ cat cert.pem ca.pem key.pem > bundle.pem

.. note::

   Usually, your website's certificate (optionally followed by the
   intermediate CA certificate) is enough to build a certificate chain.  If
   you add more certificates to your chain, order them leaf to root.

Upload the resulting file to Unit's certificate storage under a suitable name:

.. code-block:: console

   # curl -X PUT --data-binary @bundle.pem --unix-socket \
          /path/to/control.unit.sock http://localhost/certificates/<bundle>

       {
           "success": "Certificate chain uploaded."
       }

.. warning::

   Don't use :option:`!-d` for file upload; this option damages :file:`.pem`
   files.  Use the :option:`!--data-binary` option when uploading file-based
   data with :program:`curl` to avoid data corruption.

Internally, Unit stores uploaded certificate bundles along with other
configuration data in its :file:`state` subdirectory; Unit's control API maps
them to a separate configuration section, aptly named :samp:`certificates`:

.. code-block:: json

   {
       "certificates": {
           "<bundle>": {
               "key": "RSA (4096 bits)",
               "chain": [
                   {
                       "subject": {
                           "common_name": "example.com",
                           "alt_names": [
                               "example.com",
                               "www.example.com"
                           ],

                           "country": "US",
                           "state_or_province": "CA",
                           "organization": "Acme, Inc."
                       },

                       "issuer": {
                           "common_name": "intermediate.ca.example.com",
                           "country": "US",
                           "state_or_province": "CA",
                           "organization": "Acme Certification Authority"
                       },

                       "validity": {
                           "since": "Sep 18 19:46:19 2018 GMT",
                           "until": "Jun 15 19:46:19 2021 GMT"
                       }
                   },

                   {
                       "subject": {
                           "common_name": "intermediate.ca.example.com",
                           "country": "US",
                           "state_or_province": "CA",
                           "organization": "Acme Certification Authority"
                       },

                       "issuer": {
                           "common_name": "root.ca.example.com",
                           "country": "US",
                           "state_or_province": "CA",
                           "organization": "Acme Root Certification Authority"
                       },

                       "validity": {
                           "since": "Feb 22 22:45:55 2016 GMT",
                           "until": "Feb 21 22:45:55 2019 GMT"
                       }
                   },
               ]
           }
       }
   }

.. note::

    You can access individual certificates in your chain, as well as specific
    alternative names, by their indexes:

    .. code-block:: console

       # curl -X GET --unix-socket /path/to/control.unit.sock \
              http://localhost/certificates/<bundle>/chain/0/
       # curl -X GET --unix-socket /path/to/control.unit.sock \
              http://localhost/certificates/<bundle>/chain/0/subject/alt_names/0/

Next, add a :samp:`tls` object to the listener configuration, referencing the
uploaded bundle in :samp:`certificate`:

.. code-block:: json

   {
       "listeners": {
           "127.0.0.1:8080": {
               "pass": "applications/wsgi-app",
               "tls": {
                   "certificate": "<bundle>"
               }
           }
       }
   }

The resulting control API configuration may look like this:

.. code-block:: json

   {
       "certificates": {
           "<bundle>": {
               "key": "<key type>",
               "chain": ["<certificate chain, omitted for brevity>"]
           }
       },

       "config": {
           "listeners": {
               "127.0.0.1:8080": {
                   "pass": "applications/wsgi-app",
                   "tls": {
                       "certificate": "<bundle>"
                   }
               }
           },

           "applications": {
               "wsgi-app": {
                   "type": "python",
                   "module": "wsgi",
                   "path": "/usr/www/wsgi-app/"
               }
           }
       }
   }

Now you're solid.  The application is accessible via SSL/TLS:

.. code-block:: console

   $ curl -v https://127.0.0.1:8080
       ...
       * TLSv1.2 (OUT), TLS handshake, Client hello (1):
       * TLSv1.2 (IN), TLS handshake, Server hello (2):
       * TLSv1.2 (IN), TLS handshake, Certificate (11):
       * TLSv1.2 (IN), TLS handshake, Server finished (14):
       * TLSv1.2 (OUT), TLS handshake, Client key exchange (16):
       * TLSv1.2 (OUT), TLS change cipher, Client hello (1):
       * TLSv1.2 (OUT), TLS handshake, Finished (20):
       * TLSv1.2 (IN), TLS change cipher, Client hello (1):
       * TLSv1.2 (IN), TLS handshake, Finished (20):
       * SSL connection using TLSv1.2 / AES256-GCM-SHA384
       ...

Finally, you can :samp:`DELETE` a certificate bundle that you don't need
anymore from the storage:

.. code-block:: console

   # curl -X DELETE --unix-socket /path/to/control.unit.sock \
          http://localhost/certificates/<bundle>

       {
           "success": "Certificate deleted."
       }

.. note::

   You can't delete certificate bundles still referenced in your
   configuration, overwrite existing bundles using :samp:`PUT`, or (obviously)
   delete non-existent ones.

Happy SSLing!

.. _configuration-full-example:

************
Full Example
************

.. code-block:: json

   {
       "certificates": {
           "bundle": {
               "key": "RSA (4096 bits)",
               "chain": [
                   {
                       "subject": {
                           "common_name": "example.com",
                           "alt_names": [
                               "example.com",
                               "www.example.com"
                           ],

                           "country": "US",
                           "state_or_province": "CA",
                           "organization": "Acme, Inc."
                       },

                       "issuer": {
                           "common_name": "intermediate.ca.example.com",
                           "country": "US",
                           "state_or_province": "CA",
                           "organization": "Acme Certification Authority"
                       },

                       "validity": {
                           "since": "Sep 18 19:46:19 2018 GMT",
                           "until": "Jun 15 19:46:19 2021 GMT"
                       }
                   },

                   {
                       "subject": {
                           "common_name": "intermediate.ca.example.com",
                           "country": "US",
                           "state_or_province": "CA",
                           "organization": "Acme Certification Authority"
                       },

                       "issuer": {
                           "common_name": "root.ca.example.com",
                           "country": "US",
                           "state_or_province": "CA",
                           "organization": "Acme Root Certification Authority"
                       },

                       "validity": {
                           "since": "Feb 22 22:45:55 2016 GMT",
                           "until": "Feb 21 22:45:55 2019 GMT"
                       }
                   }
               ]
           }
       },

       "config": {
           "settings": {
               "http": {
                   "header_read_timeout": 10,
                   "body_read_timeout": 10,
                   "send_timeout": 10,
                   "idle_timeout": 120,
                   "max_body_size": 6291456,
                   "static": {
                       "mime_types": {
                           "text/plain": [
                                ".log",
                                "README",
                                "CHANGES"
                           ]
                       }
                   }
               }
           },

           "listeners": {
               "*:8000": {
                   "pass": "routes",
                   "tls": {
                       "certificate": "bundle"
                   }
               },

               "127.0.0.1:8001": {
                   "pass": "applications/drive"
               }
           },

           "routes": [
               {
                   "match": {
                       "uri": "/admin/*",
                       "scheme": "https",
                       "arguments": {
                           "mode": "strict",
                           "access": "!raw"
                       },

                       "cookies": {
                           "user_role": "admin"
                       }
                   },

                   "action": {
                       "pass": "applications/cms"
                   }
               },

               {
                   "match": {
                       "host": ["blog.example.com", "blog.*.org"],
                       "source": "*:8000-9000"
                   },

                   "action": {
                       "pass": "applications/blogs"
                   }
               },

               {
                   "match": {
                       "host": "example.com",
                       "source": "127.0.0.0-127.0.0.255:8080-8090",
                       "uri": "/chat/*"
                   },

                   "action": {
                       "pass": "applications/chat"
                   }
               },

               {
                   "match": {
                       "host": "example.com",
                       "source": [
                           "10.0.0.0/7:1000",
                           "10.0.0.0/32:8080-8090"
                       ]
                   },

                   "action": {
                       "pass": "applications/store"
                   }
               },

               {
                   "match": {
                       "host": "wiki.example.com"
                   },

                   "action": {
                       "pass": "applications/wiki"
                   }
               },

               {
                   "match": {
                       "scheme": "http",
                   },

                   "action" {
                       "proxy": "http://127.0.0.1:8080"
                   }
               },

               {
                   "action" {
                       "share": "/www/not_found/"
                   }
               }
           ],

           "applications": {
               "blogs": {
                   "type": "php",
                   "root": "/www/blogs/scripts/",
                   "limits": {
                       "timeout": 10,
                       "requests": 1000
                   },

                   "options": {
                       "file": "/etc/php.ini",
                       "admin": {
                           "memory_limit": "256M",
                           "variables_order": "EGPCS",
                           "expose_php": "0"
                       },

                       "user": {
                           "display_errors": "0"
                       }
                   },

                   "processes": 4
               },

               "chat": {
                   "type": "external",
                   "executable": "bin/chat_app",
                   "group": "www-chat",
                   "user": "www-chat",
                   "working_directory": "/www/chat/",
                   "isolation": {
                       "namespaces": {
                           "cgroup": false,
                           "credential": true,
                           "mount": false,
                           "network": false,
                           "pid": false,
                           "uname": false
                       },

                       "uidmap": [
                           {
                               "host": 1000,
                               "container": 0,
                               "size": 1000
                           }
                       ],

                       "gidmap": [
                           {
                               "host": 1000,
                               "container": 0,
                               "size": 1000
                           }
                       ]
                   }
               },

               "cms": {
                   "type": "ruby",
                   "script": "/www/cms/main.ru",
                   "working_directory": "/www/cms/"
               },

               "drive": {
                   "type": "perl",
                   "script": "app.psgi",
                   "processes": {
                       "max": 10,
                       "spare": 5,
                       "idle_timeout": 20
                   },

                   "working_directory": "/www/drive/"
               },

               "store": {
                   "type": "java",
                   "webapp": "/www/store/store.war",
                   "classpath": ["/www/store/lib/store-2.0.0.jar"],
                   "options": ["-Dlog_path=/var/log/store.log"]
               },

               "wiki": {
                   "type": "python",
                   "module": "wsgi",
                   "environment": {
                       "DJANGO_SETTINGS_MODULE": "wiki.settings.prod",
                       "DB_ENGINE": "django.db.backends.postgresql",
                       "DB_NAME": "wiki",
                       "DB_HOST": "127.0.0.1",
                       "DB_PORT": "5432"
                   },

                   "path": "/www/wiki/",
                   "processes": 10
               }
           },

           "access_log": "/var/log/access.log"
       }
   }
