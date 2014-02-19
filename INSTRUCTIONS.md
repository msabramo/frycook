Frycook
=======

Frycook is a system for installing and maintaining software on Linux
computers.  It consists of a framework to build systems with and a
program you run to apply the things you built with the framework to your
computers.

At the highest level Frycook depends on recipes and cookbooks to define
how systems are built.  Recipes and cookbooks in turn use settings,
environments, and packages to do their work.  The settings and
environments are passed around within the framework as json-translatable
dictionaries.  Packages live on disk as directories and files.

Settings and environments exist as json-like dictionaries because they
are easy to turn into json files, and then read back from them, and
because most templating engines expect dictionaries to be passed to them
in order to render their contents.  This means that these dictionaries
map nicely to both the storage and rendering functions and keep frycook
simple.

Recipes and cookbooks are python code that get executed when building
and updating servers.  Each recipe and cookbook lives in its own file.

Setup
=====

Before you can do anything with frycook, you'll need to install its
python package from pypi.  I usually create a virtualenv to run it in,
but it can also be installed globally.

    pip install frycook

Globules
========

All the files necessary for frycook to function are usually arranged in
a directory structure that I call a *globule*.  The rest of these
instructions will walk you through this globule and explain all the
pieces and how they work together.

Example:

    awesome_recipes           # root directory
      packages                # directory for the package files
        hosts                 # root for hosts package files
          etc                 # corresponds to /etc on the target server
            hosts.tmplt       # template that becomes /etc/hosts on the target server
        nginx                 # root for nginx package files
          etc                 # corresponds to /etc directory on the target server
            default           # corresponds to /etc/default directory on the target server
              nginx           # corresponds to /etc/default/nginx file on the target server
            nginx             # corresponds to /etc/nginx directory on target server
              conf.d          # corresponds to /etc/nginx/conf.d directory on target server
              nginx.conf      # corresponds to /etc/nginx/nginx.conf file on target server
              sites-available # corresponds to /etc/nginx/sites-available directory on target server
                default       # corresponds to /etc/nginx/sites-available/default directory on target server
              sites-enabled   # corresponds to /etc/nginx/sites-enable directory on target server
          srv                 # corresponds to /srv directory on the target server
            www               # corresponds to /srv/www directory on the target server
              50x.html        # corresponds to /srv/www/50x.html file on target server
              index.html      # corresponds to /srv/www/index.html file on target server
      setup                   # directory for non-package files
        comp_test1.json       # included environment file for computer test1
        environment.json      # environment file
        runner.sh             # wrapper for frycooker that sets PYTHONPATH
        settings.json         # settings file
        cookbooks             # directory to hold the cookbooks package
          __init__.py         # define the cookbook list here and import all cookbook classes
          base.py             # cookbook referencing all the recipes for a base server setup
          web.py              # cookbook for make a base server into a web server
        recipes               # directory to hold the recipes package
          __init__.py         # define the recipe list here and import all recipe classes
          example_com.py      # recipe for setting up example.com on a web server
          hosts.py            # recipe for setting up the /etc/hosts file
          nginx.py            # recipe for setting up nginx
          root.py             # recipe for setting the root user's authorized_keys file

Recipes
=======

Recipes define subsystems that are distinct parts of larger systems.
They are the basic units of setup in frycook.  Generally a recipe
corresponds to a package that needs to be installed or configured.

example recipe
--------------

This example sets up the hosts file on a computer.

    import cuisine

    from frycook import Recipe

    class RecipeHosts(Recipe):
        def apply(self, computer):
            group = self.environment["computers"][computer]["host_group"]
            computers = self.environment["groups"][group]["computers"]
            sibs = [comp for comp in computers if comp != computer]
            tmp_env = {"host": computer,
                       "sibs": sibs,
                       "computers": self.environment["computers"]}
            self.push_package_file_set('hosts', computer, tmp_env)

            cuisine.sudo("service hostname restart")

recipe list
-----------

There should be a recipe list in the __init__.py file for the packge.

Here's the sample init file:

    from fail2ban import RecipeFail2ban
    from hosts import RecipeHosts
    from nginx import RecipeNginx
    from postfix import RecipePostfix
    from root_user import RecipeRootUser
    from example_com import RecipeExampleCom
    from shorewall import RecipeShorewall
    from ssh import RecipeSSH

    recipes = {
        'fail2ban': RecipeFail2ban,
        'hosts': RecipeHosts,
        'nginx': RecipeNginx,
        'postfix': RecipePostfix,
        'root_user': RecipeRootUser,
        'example_com': RecipeExampleCom,
        'shorewall': RecipeShorewall,
        'ssh': RecipeSSH
        }

idempotence
-----------

One thing to keep in mind when creating recipes and cookbooks is
idempotency.  By keeping idempotency in mind in general you can create
recipes that you can run again and again to push out minor changes to a
package.  This way your recipes become the only way that you modify your
servers and can be a single chokepoint that you can monitor to make sure
things happen properly.

Lots of the cuisine functions you'll use have an "ensure" version that
first checks to see if a condition is true before applying it, such as
checking if a package is installed before trying to install it.  This is
nice when things could cause undesired configuration changes or
expensive operations that you don't want to happen every time.  These
functions are a huge aid in writing idempotent recipes and cookbooks.

rudeness
--------

Another thing to keep in mind is that some actions performed in recipes
can affect the end users of the systems, in effect being rude to them.
This might cause an outage or otherwise mess them up.  The recipe class
keeps track of whether or not this is ok in its 'ok_to_be_rude' variable
so you can know what actions are acceptable.  Consult this before doing
rude things.

file set copying
----------------

The Recipe class defines a few helper functions for handling templates
and copying files to servers.  It runs files with a .tmplt extension
through Mako, using the dictionary you pass to it.  Regular files just
get copied.  You can specify owner, group, and permissions on a
per-directory and per-file basis.

git repo checkouts
------------------

The Recipe class also defines some helper functions for working with git
repos.  You can checkout a git repo onto the remote machine, or check it
out locally and copy it to the remote machine if you don't want to setup
the remote machine to be able to do checkouts.

apply process
-------------

This is where you apply a recipe to a server.  There are two class
methods that get called during the apply process, and possibly two
messages that get displayed.  Generally you'll just override the apply
method and sometimes add pre_apply or post_apply messages.  If you
override pre_apply_checks, remember to call the base class method.
Here's the order that things happen in:

pre_apply_message -> pre_apply_checks() -> apply() -> post_apply_message

Cookbooks
=========

Cookbooks are sets of recipes to apply to a server to create systems
made up of subsystems.

Example:

    from frycook import Cookbook

    from recipes import RecipeHosts
    from recipes import RecipeRootUser
    from recipes import RecipeShorewall
    from recipes import RecipeSSH
    from recipes import RecipeFail2ban
    from recipes import RecipePostfix

    class CookbookBase(Cookbook):
        recipe_list = [RecipeRootUser,
                       RecipeHosts,
                       RecipeShorewall,
                       RecipeSSH,
                       RecipeFail2ban,
                       RecipePostfix]

There should be a cookbook list in the __init__.py file for the
cookbooks packge.

Here's the init file for the sample cookbooks module:

    from base import CookbookBase
    from web import CookbookWeb

    cookbooks = {
        'base': CookbookBase,
        'web': CookbookWeb
        }

Packages Directory
==================

The packages directory contains all the files needed by the recipes.
There is one sub-directory per package, and each package generally
corresponds to a recipe.  Within each package the directories and files
are laid out the exact same as they will be on the target systems.  Any
files with .tmplt extensions will be processed as mako templates before
being copied out to computers.  The fck_metadata.txt files define the
ownership and permissions for the files and directories when they're
copied to the target system.  The fck_delete.txt files list files that
should be deleted in that directory on the target systems.

Here's the packages directory layout from our sample globule:

    packages                # directory for the package files
      hosts                 # root for hosts package files
        etc                 # corresponds to /etc on the target server
          hosts.tmplt       # template that becomes /etc/hosts on the target server
      nginx                 # root for nginx package files
        etc                 # corresponds to /etc directory on the target server
          default           # corresponds to /etc/default directory on the target server
            nginx           # corresponds to /etc/default/nginx file on the target server
          nginx             # corresponds to /etc/nginx directory on target server
            conf.d          # corresponds to /etc/nginx/conf.d directory on target server
            nginx.conf      # corresponds to /etc/nginx/nginx.conf file on target server
            sites-available # corresponds to /etc/nginx/sites-available directory on target server
              default       # corresponds to /etc/nginx/sites-available/default directory on target server
            sites-enabled   # corresponds to /etc/nginx/sites-enable directory on target server
        srv                 # corresponds to /srv directory on the target server
          www               # corresponds to /srv/www directory on the target server
            50x.html        # corresponds to /srv/www/50x.html file on target server
            index.html      # corresponds to /srv/www/index.html file on target server

regular files
-------------

template files
--------------

fck_delete.txt files
--------------------

fck_metadata.txt files
----------------------

Settings
========

There are a few settings that frycook depends on.  They are read in from
a JSON file called settings.json and are passed around as a dictionary
to the constructors for Cookbooks and Recipes.  The settings dicionary
has the following keys:

*package_dir*: root of the packages hierarchy for the file copying
routines to look in

*module_dir*: temporary directory for holding compiled mako modules
during template handling

*file_ignores*: regex pattern for filenames to ignore while copying
package files

*tmp_dir*: directory used to temporary hold files during some of the git
 repo manipulation routines in recipes

For any key containing 'dir' or 'path', if you include a tilde (~) in
the value, it will be replaced with the home directory of the user
running frycooker, just like in bash.  For this example, that would be
the "package_dir" and "module_dir" keys.

Example settings.json file:

    {
      "package_dir": "~/bigawesomecorp/admin/packages/",
      "module_dir": "/tmp/bigawesomecorp/mako_modules",
      "tmp_dir": "/tmp",
      "file_ignores": ".*~"
    }

Environment
===========

Frycook depends on having detailed knowledge of the metadata needed by
all the components when software is being setup on the computers.  It is
read in from a set of files into a single dictionary that is passed
around to the parts of the frycook framework.  The environment
dictionary contains all the metadata about the computers and the
environment they live in that the recipes and cookbooks need to
function.  Most of its data is directly relevant to specific recipes and
is filled in depending on the recipes' needs.  It's a dictionary with
three main sections that should always be there:

*users*: a list of the users that recipes could reference, with such
things as public ssh keys

*computers*: a list of computers in the system, with such things as ip
addresses

*groups*: groups of computers that will be addressed as a unit

Just like for the settings dictionary, any key containing 'dir' or
'path' and including a tilde (~) in the value will have the tilde
replaced with the home directory of the user running frycooker.

Each computer in the *computers* section is also expected to have a
*components* section listing all the cookbooks and recipes that that
computer uses.  This is used by frycooker to easily apply all relevant
cookbooks and recipes to computers.  It's also a good way to keep track
of what components make up that computer when it's fully functional.

Each component in the list has a *type* key identifying if it's a
cookbook or a recipe, and a *name* key identifying the name of the
cookbook or recipe.

Example:

    {
      "users": {
        "root": {
          "ssh_public_key": "ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQDYK8U9Isp+Ih+THCj2ohCo6nLY1R5Sn7oPzxM8ZBwH3ik/2EF3v0ibNezruja1I3OwF8W1QyWOdooIwTYJ8HXH9+Gyxcq/PseXbFWqg3k/lL50d5AawyRQZndOaNcFG6B8ULXJDksA6oQccXRzzxmnXpwGR8XEfSBCo2cdWDF1CXKvKXDZ4sqvGTVJIKshUAVbmfi4wH0LTtGIlV4IxslKUbfsErIU8kSyZNLLslq9XRvlqVK3iSabomKUY14MTbc3sefQzIctTtlmBpZw2mMBS49k4HYo1UwhUNiLbFBS7QhcivbJwFqGPj0N5pAx0oPUj1m96GGsqpiqu1eNp/yb jay@Jamess-MacBook-Air.local"
        },
        "example_com": {
          "ssh_public_key": "ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQDYK8U9Isp+Ih+THCj2ohCo6nLY1R5Sn7oPzxM8ZBwH3ik/2EF3v0ibNezruja1I3OwF8W1QyWOdooIwTYJ8HXH9+Gyxcq/PseXbFWqg3k/lL50d5AawyRQZndOaNcFG6B8ULXJDksA6oQccXRzzxmnXpwGR8XEfSBCo2cdWDF1CXKvKXDZ4sqvGTVJIKshUAVbmfi4wH0LTtGIlV4IxslKUbfsErIU8kSyZNLLslq9XRvlqVK3iSabomKUY14MTbc3sefQzIctTtlmBpZw2mMBS49k4HYo1UwhUNiLbFBS7QhcivbJwFqGPj0N5pAx0oPUj1m96GGsqpiqu1eNp/yb jay@Jamess-MacBook-Air.local"
        }
      },

      "computers": {
        "test1": {
          "domain_name": "fubu.example",
          "host_group": "test",
          "public_ifaces": ["eth0", "eth1"],
          "public_ips": {"192.168.56.10": "test1.fubu.example",
                         "192.168.56.11": "test2.fubu.example"},
          "private_ifaces": ["eth2"],
          "private_ips": {"192.168.1.126": "test1"},
          "components": [{"type": "cookbook",
                          "name": "base"},
                         {"type": "cookbook",
                          "name": "web"}]
        }
      },

      "groups": {
        "test" : {
          "computers": ["test1"]
        }
      }
    }

This dictionary is created by processing thee environment JSON files.
The main file is called environment.json, and it can have include
directives that pull in additonal json files so that you can split up
large environments into multiple files.

environment.json:

    {
      "users": {
        "root": {
          "ssh_public_key": "ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQDYK8U9Isp+Ih+THCj2ohCo6nLY1R5Sn7oPzxM8ZBwH3ik/2EF3v0ibNezruja1I3OwF8W1QyWOdooIwTYJ8HXH9+Gyxcq/PseXbFWqg3k/lL50d5AawyRQZndOaNcFG6B8ULXJDksA6oQccXRzzxmnXpwGR8XEfSBCo2cdWDF1CXKvKXDZ4sqvGTVJIKshUAVbmfi4wH0LTtGIlV4IxslKUbfsErIU8kSyZNLLslq9XRvlqVK3iSabomKUY14MTbc3sefQzIctTtlmBpZw2mMBS49k4HYo1UwhUNiLbFBS7QhcivbJwFqGPj0N5pAx0oPUj1m96GGsqpiqu1eNp/yb jay@Jamess-MacBook-Air.local"
        },
        "example_com": {
          "ssh_public_key": "ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQDYK8U9Isp+Ih+THCj2ohCo6nLY1R5Sn7oPzxM8ZBwH3ik/2EF3v0ibNezruja1I3OwF8W1QyWOdooIwTYJ8HXH9+Gyxcq/PseXbFWqg3k/lL50d5AawyRQZndOaNcFG6B8ULXJDksA6oQccXRzzxmnXpwGR8XEfSBCo2cdWDF1CXKvKXDZ4sqvGTVJIKshUAVbmfi4wH0LTtGIlV4IxslKUbfsErIU8kSyZNLLslq9XRvlqVK3iSabomKUY14MTbc3sefQzIctTtlmBpZw2mMBS49k4HYo1UwhUNiLbFBS7QhcivbJwFqGPj0N5pAx0oPUj1m96GGsqpiqu1eNp/yb jay@Jamess-MacBook-Air.local"
        }
      },

      "computers": {
        "imports": ["comp_test1.json"]
      },

      "groups": {
        "test" : {
          "computers": ["test1"]
        }
      }
    }

comp_test1.json:

    "test1": {
      "domain_name": "fubu.example",
      "host_group": "test",
      "public_ifaces": ["eth0", "eth1"],
      "public_ips": {"192.168.56.10": "test1.fubu.example",
                     "192.168.56.11": "test2.fubu.example"},
      "private_ifaces": ["eth2"],
      "private_ips": {"192.168.1.126": "test1"},
      "components": [{"type": "cookbook",
                      "name": "base"},
                     {"type": "cookbook",
                      "name": "web"}]
    }

Frycooker
=========

Frycooker is the program that takes all your carefully coded recipes and
cookbooks and applies them to computers.

The recipes and cookbooks modules should be accessible via the
PYTHONPATH so they can be imported.

explain messages

just print messages

dry run

----------------
Copyright (c) James Yates Farrimond. All rights reserved.

Redistribution and use in source and binary forms, with or without
Modification, are permitted provided that the following conditions are
met:

Redistributions of source code must retain the above copyright notice,
this list of conditions and the following disclaimer.

Redistributions in binary form must reproduce the above copyright
notice, this list of conditions and the following disclaimer in the
documentation and/or other materials provided with the distribution.

THIS SOFTWARE IS PROVIDED BY JAMES YATES FARRIMOND ''AS IS'' AND ANY
EXPRESS OR IMPLIED WARRANTIES, INCLUDING, BUT NOT LIMITED TO, THE
IMPLIED WARRANTIES OF MERCHANTABILITY AND FITNESS FOR A PARTICULAR
PURPOSE ARE DISCLAIMED. IN NO EVENT SHALL JAMES YATES FARRIMOND OR
CONTRIBUTORS BE LIABLE FOR ANY DIRECT, INDIRECT, INCIDENTAL, SPECIAL,
EXEMPLARY, OR CONSEQUENTIAL DAMAGES (INCLUDING, BUT NOT LIMITED TO,
PROCUREMENT OF SUBSTITUTE GOODS OR SERVICES; LOSS OF USE, DATA, OR
PROFITS; OR BUSINESS INTERRUPTION) HOWEVER CAUSED AND ON ANY THEORY OF
LIABILITY, WHETHER IN CONTRACT, STRICT LIABILITY, OR TORT (INCLUDING
NEGLIGENCE OR OTHERWISE) ARISING IN ANY WAY OUT OF THE USE OF THIS
SOFTWARE, EVEN IF ADVISED OF THE POSSIBILITY OF SUCH DAMAGE.

The views and conclusions contained in the software and documentation
are those of the authors and should not be interpreted as representing
official policies, either expressed or implied, of James Yates
Farrimond.