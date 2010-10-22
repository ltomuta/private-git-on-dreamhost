

=======================================
Private GIT repositories (in DreamHost)
=======================================

:Author: Tiago Alves Macambira
:Licence: Creative Commons By-SA


.. contents:: Table of Contents



Introduction
============

This is *yet another* guide describing how to setup private
HTTP-accessible Git repositories in Dreamhost_ using Git's
``git-http-backend`` (a.k.a git's `Smart HTTP protocol`__). While
similar guides can easily be found by the thousands in the Web (I've
listed some of them in the Refereces section), I've found that some
guides have outdated information and that the setup described in
others could be improved. Thus, this guide tries to update and
consolidate the information dispersed in such sources.

__ GitSmartHTTP_

Some might ask "Why on Earth would someone opt to create its on
private Git hosting solution while better offerings are available from
sites such as, let's say, GitHub_?". As the guy `from RailsTips
pointed out in one of his articles`__, sometimes you don't need or
don't want to "share" a project with anyone but with yourself and
paying for a GitHub_-like service might just not make sense.  If
that's your case, than this guide is for you.

__ RailsTipsArticle_

While aimed at a Dreamhost_-hosted accounts and the environment such
accounts have as of 2010-10-17, I believe the process described here
can be used in other hosting providers as well.


It is important to highlight that one of the objectives of this guide
is to describe a process that:

* should be easy to perform by just renaming or editing this guide's
  companion files and
* once complete, can be easily be reused to generate other "collection
  of repositories", with different URLs and passwords.


Assumptions and requirements
----------------------------

* No WebDav or SSH support is needed nor used for serving Git repositories.
  
  Once again, the focus is on HTTP access using
  ``git-http-backend``. As for SSH, guides describing how to setup a
  similar environment for SSH-accessible repositories can be found
  easily on the web.

* We will stick to a DRY_ (Don't Repeat Yourself) philosophy.

  Thus, we want configuration options to be repeated in as few places
  as possible. We will use environment variables in Apache
  configuration files to do this. If this makes you uncomfortable,
  well, you can always manually spread configuration options all over
  the place. :-) Your call.


* The web server being used is Apache.

  Well, that is what Dreamhost_ allows me to use so that is going to
  be the focus of this guide. While it should not be that difficult to
  port the settings here to something suitable for another web server,
  describing how to do it is out of the scope of this guide.

* We are able to run CGI scripts.

  `DreamHosts puts some restrictions`__
  on how a CGI script can be executed and the environment where it
  runs. We will abide to those restrictions.

__ DreamHostWikiCGI_

* ``ScriptAlias`` is not allowed by the web server.

  The instructions given in git-http-backend_ manpage will not work as
  they use ``ScriptAlias``. The idea is to use common CGI scripts and
  ``mod_rewrite`` instead, roughly following the ideas presented in
  http://wiki.dreamhost.com/Git#Smart_HTTP .

* SuExec_ is used to run CGI scripts.

  Notice that, as stated in DreamHost page on CGI, ``SuExec`` "*wipes out all
  environment variables that don't start with HTTP\_"*. All our
  env. vars. will have this prefix.

* Your private git repositories will be accessible in a subpath of your
  domain.

  The idea is that your private git repos will be available in an
  address such as **http://www.example.tld/corporate-git/**. Adapting
  the instructions bellow so you can serve them from the root of a
  domain of its own, say **http://corporate-git.example.tld** should
  be fairly simple.


Installation
============

All the commands and instructions given bellow should be performed on
the machine in Dreamhost where your account is installed. So ssh to it
and let's start.

Install Git
-----------

Well, this should probably be a non-issue since git comes
pre-installed on most Dreamhost machines. To verify it::

    $ git --version
    git version 1.7.1.1

As you see, the box that serves my domain in dreamhost has git version
1.7.1.1 installed. `Anything greater than 1.6.6 shall do`__.

__ GitSmartHTTP_



If you don't have git installed in you box, have an old version or if
for some other reason your need to compile git, follow Craig's instructions in
|CraigJolicoerArticle|_.



Create the directory where your repositories will live
------------------------------------------------------


It should reside somewhere not accessible from the web or directly
served by the web server. We will tell Apache and ``git-http-backend``
how to properly and securely serve those repositories latter. For now,
we want them protected from third parties.

Say we decided to store them in ``~/private_repos/``. We will refer to
this directly by ``GIT_REPOS_ROOT`` in the rest of this guide. Create
this directory and protect it against filesystem access from others::

    export GIT_REPOS_ROOT="~/private_repos/"
    mkdir ${GIT_REPOS_ROOT}
    chmod 700 ${GIT_REPOS_ROOT}


Setup the bare repository creation script
-----------------------------------------



We will use the script ``newgit.sh``, presented bellow, to create new
repositories [1]_ [2]_ . Remember to modify
the value of the GIT_REPOS_ROOT variable in it to match our setup:

.. include:: newgit.sh
   :literal:

Move or copy this file to an appropriate path (say, your home
directory would be fine) and turn it into an executable::

    chmod u+x ~/newgit.sh

.. [1] This script is based in http://gist.github.com/73622

.. [2] Other guides prefer to use something similar wrapped as a Bash
       function but I'd rather have it as a script


Apache Setup
------------

Now, let's configure Apache to securely serve those repositories.


Setup your .htaccess
~~~~~~~~~~~~~~~~~~~~

As we stated in `Assumptions and requirements`_, we want to serve our files from
**http://www.example.tld/corporate-git/**. So, go to the directory
holding your domain files (``~/www.example.tld``, in our exemple),
create a ``corporate-git`` directory in it if it doesn't exist yet and create
a ``.htaccess`` file in it::

    cd ~/www.example.tld
    mkdir corporate-git
    cd corporate-git
    export GIT_WEB_DIR=`pwd` # we will use it in latter steps
    touch .htaccess
    chmod 700 .htaccess


Now, edit this ``.htaccess`` contents to match the text presented
bellow or just copy the contents of the file ``model-htaccess`` into
it and adapt it to match your config:


.. include:: model-htaccess
   :literal:

For now we will focus on the area between the ``# GIT BEGIN`` and ``#
GIT END`` blocks.  Modify ``HTTP_GIT_PROJECT_ROOT`` to match you setup:
it should point to the **full path** where you store your private
repositories. Just expand the value of ``GIT_REPOS_ROOT`` to get this
information::

    $ (cd ${GIT_REPOS_ROOT}; pwd)
    /home/user/private_repos/

So, in our example, ``HTTP_GIT_PROJECT_ROOT`` value should be set to
``/home/user/private_repos/``, as presented in the example above.

Setup git-http-web for you repositories
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Not we will create a CGI script that will invoke
``git-http-backend``. In your ``.htaccess`` this script is referred as
``git-http-backend-private.cgi``. Create it in the same directory
where you ``.htaccess`` is by coping the one that comes with this guide
to that directory or by creating an empty file with the following
contents:

.. include:: git-http-backend-private.cgi
   :literal:

Turn it into an executable file::

    chmod 755 git-http-backend-private.cgi


.. attention::
    You may need to update the path to ``git-http-backend`` executable
    if git was installed in a non-default location.

And that's it. No need to setup anything: all the settings this
scripts are passed to it through environment variables set by Apache
and defined in the ``.htaccess`` file.

From this point on you should be able to create repositories and
access them through HTTP.


Password-protect you repository
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

We are almost set. Let's configure password protection for this whole
thing.  We will focus on the latter part of your ``.htaccess``, the one
between ``# AUTHENTICATION BEGIN`` and ``# AUTHENTICATION END`` that we
reproduce bellow::

    # AUTHENTICATION BEGIN ########################
    AuthType Digest
    AuthName "Private Git Repository Access"
    AuthUserFile /home/user/private_repos/.htpasswd
    Require valid-user
    # AUTHENTICATION END  #########################

You will have to create the password file pointed by ``AuthUserFile``
with the ``htdigest`` tool::

    htdigest -c /home/user/private_repos/.htpasswd

Now add a user to this file::

    htdigest /home/user/private_repos/.htpasswd "Private Git Repository Access" username


You will be prompted for a password. And that's it.


Notice:

* we are using `Digest Authentication
  <http://httpd.apache.org/docs/2.2/mod/mod_auth_digest.html#using>`_. Is
  is supposed to be more secure than plain authentication.
* The password file should be keep in a place not directly accessible
  from the web. Ideally it should not even be placed in the directory
  to be served by ``git-http-backend`` but I'm lazy and I hope this
  will be enough. :) 
* If you update the value of the ``AuthName`` setting you **must**
  also change the 2nd. parameter passed to ``htdigest``, i.e., the
  *Realm*, as `they must match
  <http://www.freebsdwiki.net/index.php/Apache,_Digest_Authentication>`_!
  Odd, I know. But that's the way it is.


Setup GitWeb
~~~~~~~~~~~~

If you followed this guide up to this point than you are able to use
your repositories with git with no major issues. But you will not be
able to browse them with a web browser, retrieve the list of
repositories you have, see diffs, commit messages nor nothing like
that. To make things better, let's install GitWeb, another CGI
interface that will provide a web interface that allows to do all
those things I just said you couldn't.

.. note::
   Most of the content in this section comes from  Kang's |KangArticle|_.


Retrieving and installing
+++++++++++++++++++++++++

GitWeb comes in the same source package as git itself. Unfortunately,
Dreamhost doesn't install it by default so we will have to install it
manually ourselves. Do your remember what is your git version? No?
Find it all::

    git --version

Go to `git homepage`_ and download the corresponding source
package. In my example, in which my git version is 1.7.1.1, I would
need to grab the ``git-1.7.1.1.tar.gz`` source package::

    cd ~ # Yep, we will download it in our home directory
    wget http://www.kernel.org/pub/software/scm/git/git-1.7.1.1.tar.gz

Unpack it, build GitWeb::

    tar zxvf git-1.7.1.1.tar.gz
    cd git-1.7.1.1
    make prefix=/usr/bin gitweb/gitweb.cgi
    rm gitweb/gitweb.perl # we won't need it

We will install it into ``~/gitweb/``::

   export GITWEB_INSTALL_DIR="~/gitweb"
   cp -r gitweb ${GITWEB_INSTALL_DIR}

We are almost there.


Setting up GitWeb
+++++++++++++++++

Now, copy all the GitWeb's media files into the directory
where your ``.htaccess`` is::

    cp ${GITWEB_INSTALL_DIR}/*.{css,png,js} ${GIT_WEB_FOLDER}
    # in this example, GIT_WEB_FOLDER points
    # to ~/www.example.tld/corporate-git

Get back to where your ``.htaccess`` file is
(i.e. ``GIT_WEB_FOLDER``). We will create a wrapper CGI for
GitWeb. Just copy ``gitweb_wrapper.cgi`` or create an empty file with
the contents bellow:

.. include:: gitweb_wrapper.cgi
   :literal:

.. attention::
   If you have installed gitweb files in a different directory, you
   will have to update this file to match the install location.

Once again, we are using settings stored in ``.htaccess`` file and
passing them to a script using environment variables set by Apache. In
this case, we are informing the wrapper script where our repositories
are with ``HTTP_GIT_PROJECT_ROOT``, and informing it where GitWeb
configuration file is with ``HTTP_GITWEB_CONFIG``. The wrapper script,
in turn, will forward these informations to both GitWeb and to its
config file.

Now, let's create GitWeb configuration file. Just
copy ``gitweb_config.perl`` provided with this guide to
``${GIT_REPOS_ROOT}/gitweb_config.perl`` or create an empty file in
that path location with the following contents:

.. include:: gitweb_config.perl
   :literal:

You can customize it a little bit, if you want, but the most important
setting, ``$projectroot``, is set to match the value of
``HTTP_GIT_PROJECT_ROOT``, a env. var. set by Apache.

Notice that this file, ``gitweb_config.perl`` is stored in the same
directory where your repositories are, in ``${GIT_REPOS_ROOT}``. If,
for some reason, you prefer to store it elsewhere, you will have to
update this information in the ``.htaccess`` file.

    

Troubleshooting
---------------

So, something is not working as expected?

First of all, look at your server logs. They will probably give you a
clue of what is going wrong.

Comment out the authentication code. This will ease your "debugging"
process.

A nice way to check if there is something really wrong with your setup
is to use the ``info.cgi``, whose code is presented bellow. This
script is only a minor modification to the one presented in `Dreamhost
wiki page on CGI`__ and allows your to do verify if you are able to
execute CGI scrips and what settings Apache is passing to the other
CGI scripts we use here.

.. include:: info.cgi
    :literal:

Copy it to ``GIT_WEB_FOLDER``, turn it into an executable script
(``chmod 755 ...``) and point your browser to it ( That would be
``http://www.example.tld/corporate-git/`` in our example).


__ DreamHostWikiCGI_



Usage
=====

So everything is ready to use. How do you actually create and use
these new repositories?

Creating new bare repositories
------------------------------

In order to create a new repository, say ``toyproject.git``, all you
have to do is ssh into your Dreamhost account and::

    ~/newgit.sh toyproject.git


That's it: your created and empty repository in you repository
collection. You cannot *clone* it yet cause it is empty::


    $ git clone http://www.example.tld/corporate-git/toyproject.git
    Initialized empty Git repository in /private/tmp/teste/.git/
    fatal: http://www.example.tld/corporate-git/toyproject.git/info/refs not found: did you run git update-server-info on the server?


So, now what? Keep reading.

Pulling and pushing to a new bare repository
--------------------------------------------


What you usually do is creating a local repository, adding file to it and committing this repository history to the new, empty and pristine repository in your web server.


http://help.github.com/creating-a-repo/



XXX TODO


Final remarks
=============

If you need more than one collection of private repositories (say, one
for you and one to share privately with a group of coworkers), all you
need to do is:

 1. Create a directory for each of these collections.
 2. Create copies of ``newgit.sh``, one for each collection, and setup
    the value of GIT_REPOS_ROOT in each of them.
 3. Adapt each .htaccess accordingly.
 4. GitWeb: copy its files too.. Or just sym-link it from a pristine copy.
 

TODOs
=====

* Focus on reusability.
* Write the `Final remarks`_ section properly.


http://httpd.apache.org/docs/2.2/mod/mod_rewrite.html#rewritecond --
serve directly w/ apache if...

Adding project .description directly in the scripts



References
==========

* arvinderkang.com - |KangArticle|_
* craigjolicoeur.com - |CraigJolicoerArticle|_
* |RailsTipsArticle|_
* http://faves.eapen.in/guide-to-hosting-git-repositories-on-dreamhos
* http://gist.github.com/73622
* http://wiki.dreamhost.com/Git#Smart_HTTP
* http://arvinderkang.com/2010/08/25/hosting-git-repositories-on-dreamhost/
* git-http-backend_ manpage
* `Pro Git - Smart HTTP Transport <http://progit.org/2010/03/04/smart-http.html>`_
* http://www.jedi.be/blog/2009/05/06/8-ways-to-share-your-git-repository/

.. _DreamHost: http://www.dreamhost.com
.. _GitHub: http://github.com
.. |RailsTipsArticle| replace:: Git'n Your Shared Host On
.. _RailsTipsArticle: http://railstips.org/blog/archives/2008/11/23/gitn-your-shared-host-on/
.. |CraigJolicoerArticle| replace:: Hosting Git Repositories on Dreamhost
.. _CraigJolicoerArticle: http://craigjolicoeur.com/blog/hosting-git-repositories-on-dreamhost
.. |KangArticle| replace:: Hosting Git repositories on Dreamhost
.. _KangArticle: http://arvinderkang.com/2010/08/25/hosting-git-repositories-on-dreamhost/
.. _SuExec: http://wiki.dreamhost.com/Suexec
.. _DRY: http://en.wikipedia.org/wiki/Don't_repeat_yourself
.. _git-http-backend: http://www.kernel.org/pub/software/scm/git/docs/git-http-backend.html
.. _GitSmartHTTP: http://progit.org/2010/03/04/smart-http.html
.. _Git homepage: http://git-scm.com/
.. _DreamHostWikiCGI: http://wiki.dreamhost.com/CGI
.. 
   .. target-notes::


.. sectnum::