Writing Plugins for NodeBB
==========================

So you want to write a plugin for NodeBB, that's fantastic! There are a couple of things you need to know before starting that will help you out.

Like WordPress, NodeBB's plugins are built on top of a hook system in NodeBB. This system exposes parts of NodeBB to plugin creators in a controlled way, and allows them to alter content while it passes through, or execute certain behaviours when triggered.

See the full `list of hooks <https://github.com/NodeBB/NodeBB/wiki/Hooks/>`_ for more information.

Filters and Actions
------------------

There are three types of hooks: **filters**, **actions**, and **static** hooks.

**Filters** act on content, and can be useful if you want to alter certain pieces of content as it passes through NodeBB. For example, a filter may be used to alter posts so that any occurrences of "apple" gets changed to "orange". Likewise, filters may be used to beautify content (i.e. code filters), or remove offensive words (profanity filters).

**Actions** are executed at certain points of NodeBB, and are useful if you'd like to *do* something after a certain trigger. For example, an action hook can be used to notify an admin if a certain user has posted. Other uses include analytics recording, or automatic welcome posts on new user registration.

**Static** hooks are executed and wait for you to do something before continuing. They're similar to action hooks, but whereas execution in NodeBB continues immediately after an action is fired, static hooks grant you a bit of time to run your own custom logic, before resuming execution.

When you are writing your plugin, make sure a hook exists where you'd like something to happen. If a hook isn't present, `file an issue <https://github.com/NodeBB/NodeBB/issues>`_ and we'll include it in the next version of NodeBB.

Configuration
------------------

Each plugin package contains a configuration file called ``plugin.json``. Here is a sample:

.. code:: json

    {
        "url": "Absolute URL to your plugin or a Github repository",
        "library": "./my-plugin.js",
        "staticDirs": {
            "images": "public/images"
        },
        "less": [
            "assets/style.less"
        ],
        "hooks": [
            { "hook": "filter:post.save", "method": "filter" },
            { "hook": "action:post.save", "method": "emailme" }
        ],
        "scripts": [
            "public/src/client.js"
        ],
        "acpScripts": [
            "public/src/admin.js"
        ],
        "languages": "path/to/languages",
        "templates": "path/to/templates"
    }

The ``library`` property is a relative path to the library in your package. It is automatically loaded by NodeBB (if the plugin is activated).

The ``staticDirs`` property is an object hash that maps out paths (relative to your plugin's root) to a directory that NodeBB will expose to the public at the route ``/plugins/{YOUR-PLUGIN-ID}``.

* e.g. The ``staticDirs`` hash in the sample configuration maps ``/path/to/your/plugin/public/images`` to ``/plugins/my-plugin/images``

The ``less`` property contains an array of paths (relative to your plugin's directory), that will be precompiled into the CSS served by NodeBB.

The ``hooks`` property is an array containing objects that tell NodeBB which hooks are used by your plugin, and what method in your library to invoke when that hook is called. Each object contains the following properties (those with a * are required):

* ``hook``, the name of the NodeBB hook
* ``method``, the method called in your plugin
* ``priority``, the relative priority of the method when it is eventually called (default: 10)

The ``scripts`` property is an array containing files that will be compiled into the minified javascript payload served to users.

The ``acpScripts`` property is similar to ``scripts``, except these files are compiled into the minified payload served in the Admin Control Panel (ACP)

The ``languages`` property is optional, which allows you to set up your own internationalization for your plugin (or theme). Set up a similar directory structure as core, for example: ``language/en_GB/myplugin.json``.

The ``templates`` property is optional, and allows you to define a folder that contains template files to be loaded into NodeBB. Set up a similar directory structure as core, for example: ``partials/topic/post.tpl``.

The ``nbbpm`` property is an object containing NodeBB package manager info.

Writing the plugin library
------------------

The core of your plugin is your library file, which gets automatically included by NodeBB if your plugin is activated.

Each method you write into your library takes a certain number of arguments, depending on how it is called:

* Filters send a single argument through to your method, while asynchronous methods can also accept a callback.
* Actions send a number of arguments (the exact number depends how the hook is implemented). These arguments are listed in the :doc:`list of hooks <hooks>`.

Example library method
------------------

If we were to write method that listened for the ``action:post.save`` hook, we'd add the following line to the ``hooks`` portion of our ``plugin.json`` file:

.. code:: json

    { "hook": "action:post.save", "method": "myMethod" }

Our library would be written like so:

.. code:: javascript

    var MyPlugin = {
            myMethod: function(postData) {
                // do something with postData here
            }
        };

Getting a reference to `req`, `res`, `socket` and `uid` within any plugin hook
------------------

.. code:: javascript
    
    var cls = module.parent.require('./middleware/cls'); // require cls once in your plugin.

    var MyPlugin = {
            myMethod: function(postData, callback) {
                var req = cls.get('http').req; // current http request object.
                var res = cls.get('http').res; // current http response object.
                var uid = req.user.uid; // current user id
                // ...
            },
            
            // let's say this one occurs on a websocket event,
            myOtherMethod: function(somethingData) {
                var socket = cls.get('ws').socket; // current socket object.
                
                var uid = socket.uid; // current user id, if available
                
                var payload = cls.get('ws').payload; // socket payload data, if available
                var event = cls.get('ws').event; // socket last event, if available
                
                // ...
            }
        };

Using NodeBB libraries to enhance your plugin
------------------

Occasionally, you may need to use NodeBB's libraries. For example, to verify that a user exists, you would need to call the ``exists`` method in the ``User`` class. To allow your plugin to access these NodeBB classes, use ``module.parent.require``:

.. code:: javascript

    var User = module.parent.require('./user');
    User.exists('foobar', function(err, exists) {
        // ...
    });

Installing the plugin
------------------

In almost all cases, your plugin should be published in `npm <https://npmjs.org/>`_, and your package's name should be prefixed "nodebb-plugin-". This will allow users to install plugins directly into their instances by running ``npm install``.

When installed via npm, your plugin **must** be prefixed with "nodebb-plugin-", or else it will not be found by NodeBB.

Listing your plugin in the NodeBB Package Manager (nbbpm)
------------------

All NodeBB's grab a list of downloadable plugins from the NodeBB Package Manager, or nbbpm for short.

When you create your plugin and publish it to npm, it will be picked up by nbbpm, although it will not show up in installs until you specify a compatibility string in your plugin's ``package.json``.

To add this data to ``package.json``, create an object called ``nbbpm``, with a property called ``compatibility``. This property's value is a semver range of NodeBB versions that your plugin is compatible with.

You may not know which versions your plugin is compatible with, so it is best to stick with the version range that your NodeBB is using. For example, if you are developing a plugin against NodeBB v0.8.0, the simplest compatibility string would be:

.. code::

    {
        ...
        "nbbpm": {
            "compatibility": "^0.8.0"
        }
    }

To allow your plugin to be installed in multiple versions of NodeBB, use this type of string:

.. code::

    {
        ...
        "nbbpm": {
            "compatibility": "^0.7.0 || ^0.8.0"
        }
    }

Any valid semver string will work. You can confirm the validity of your semver string at this website: http://jubianchi.github.io/semver-check/

Testing
------------------

Run NodeBB in development mode:

.. code::

    ./nodebb dev

This will expose the plugin debug logs, allowing you to see if your plugin is loaded, and its hooks registered. Activate your plugin from the administration panel, and test it out.

Disabling Plugins
-------------------

You can disable plugins from the ACP, but if your forum is crashing due to a broken plugin you can reset all plugins by executing

.. code::

    ./nodebb reset -p

Alternatively, you can disable a single plugin by running

.. code::

    ./nodebb reset -p nodebb-plugin-im-broken
