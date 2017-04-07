Using Rootbox
=============

Restrictions
************

Rootbox requires two things:

- A Linux system. Some others Unixes (like the BSDs) may work, but they're
  untested.
- An ext4 file system to store the workspace directory (see below). Rootbox
  relies on the magic of ext4's sparse files to easily create boxes and images.

Also, here's something import:

**ROOTBOX IS NOT SECURE.**

Let me repeat:

**ROOTBOX IS NOT SECURE.**

Third time's the charm:

**ROOTBOX IS NOT SECURE.**

Rootbox's "boxes" are not intended to be used like you would use containers.
They *are* designed to be created and easily distributed and portable
development environments, but they are *not* designed for running web servers or
untrusted code. Use containers for that!

Getting Started
***************

Just run::

  curl -L https://goo.gl/H3OpCL | sudo sh

If you're not confortable with running random internet scripts, you can also
install it from GitHub instead::

  git clone https://github.com/project-rootbox/rootbox.git
  cd rootbox
  sudo sh install.sh

Once that's done, you need to initialize the Rootbox workspace directory, which
is where all your images and boxes will be stored. You can do so using
``rootbox init``::

  rootbox init

This will initialize Rootbox in ``$HOME/.rootbox``. If you want to use a
different directory, just pass ``-d``::

  rootbox init my_directory

This will create the directory and symlink it to ``$HOME/.rootbox``.

Note that, as already mentioned above, **this directory must be on an ext4 file
system**!

If at any point in the future, you want to use a different workspace directory,
you'll need to pass ``-f`` to force the reinitialization.

.. warning::

  Do **NOT** tamper with the workspace directory! By that, I mean don't go
  touching it in any way. Remember when I said that Rootbox uses ext4's sparse
  files? Well, that means that, if you try to play with the directory or
  copy/move files outside of it, there's a chance all the sparse files will
  inflate to their full 128 GB size. Ouch!!

Concepts
********

Before we continue, I'd like to introduce some terms you'll see floating around
this document:

- **workspace**: As mention above, this is the directory where Rootbox will save
  all your images and boxes (see below) into.

- **boxes**: Boxes are essentially Rootbox's version of containers. They hold
  an `Alpine Linux <https://alpinelinux.org/>`_ installation, as well as
  whatever else you want to put in them. Note that, unlike containers, boxes are
  intentionally insecure, and they aren't intended to be used for everything
  that one might use containers for.

- **images**: Every box is base on an *image*. Images are just vanilla,
  unmodified Alpine Linux installations.

- **bind mounts**: These allow you to "mount" external directories inside your
  boxes.

Got that? Let's go on!

Creating an image
*****************

As already mentioned, images are unmodified installations of Alpine Linux, ready
to be converted to a box when desired. They're easy to create::

  rootbox image.add -v 3.5

Here, we're using Alpine Linux 3.5, as passed to the version argument. Note that
the resulting image will be around 210 MB in size. This seems rather big, but
it also contains several development tools and libraries, such as GCC,
libstdc++, git, and more. If you want just a plain Alpine Linux installation,
then you can also pass ``-s``::

  rootbox image.add -v 3.5 -s

This will create a *slim* or *nodev* image, which is significantly lighter
(around ~13 MB in size) but doesn't contain development tools inside. When you
create a slim image, you can later reference using ``VERSION-nodev``, like
``3.5-nodev``.

(You can also use ``image.list`` to list all your installed images and
``image.remove -v VERSION`` to remove one of your images.)

Creating a box
**************

Creating a new box is easy::

  rootbox box.new -n mybox -v 3.5

Here, we're creating a box called *mybox*, using the Alpine Linux 3.5 image that
was created earlier. (To use the slim/nodev image, pass ``-v 3.5-nodev``
instead.)

You can also add a list of bind mounts to be mounted whenever the box is run.
For example::

  rootbox box.new -n mybox -v 3.5 -b outside_directory///inside_directory

Now, whenever this box is run ``outside_directory`` will appear inside it as
``/inside_directory``. For instance, if you always want the current directory
to be mounted inside the box as ``/cwd``, you could run::

  rootbox box.new -n mybox -v 3.5 -b .///cwd

Absolute paths can be used, too::

  rootbox box.new -n mybox -v 3.5 -b /home/$USER///external_home

Running your boxes
******************

Now that a box has been created, let's run it! ::

  rootbox box.run -n mybox

This will put you inside an ``ash`` shell inside your box. Take a look around
for a bit! Once you're done, you can Ctrl-D out of it.

Just like above, bind mounts can be created when the box is run::

  rootbox box.run -n mybox -b .///cwd

These will be mounted in addition to any specified when creating the box for the
first time.

While inside the box, you can also install packages using the Alpine package
manager,
`apk <https://wiki.alpinelinux.org/wiki/Alpine_Linux_package_management>`_, like
this::

  sudo apk add clang

A command can also be passed via the command line::

  rootbox box.run -n mybox -c 'echo 123'

Box factories
*************

Factories are an imporant concept in Rootbox! A box factory is just a shell
script that's run inside your box upon creation to set things up. For instance,
you could create a factory ``clang.sh`` containing:

.. code-block:: shell

  sudo apk add clang

To create a box using your factory, you can just run::

  rootbox box.new -n mybox -v 3.5 -f clang.sh

``-f`` takes a path to your box factory. However, things get fancier than that!

You can have one factory depend on another one. For instance, you might have
``llvm.sh`` to install llvm:

.. code-block:: shell

  sudo apk add llvm

Then, ``clang.sh`` could be modified to read:

.. code-block:: shell

  #:DEPENDS llvm.sh
  sudo apk add clang

The ``#:DEPENDS`` means that ``llvm.sh`` must be run first. Now, when you use
``clang.sh`` as your box factory, ``llvm.sh`` will be run, too!

Box factories can also specify Alpine Linux versions that they work on:

.. code-block:: shell

  #:VERSION 3.5 3.5-nodev

This factory will run under 3.5 and 3.5-nodev, but if you try to use it on an
Alpine 3.4 box, Rootbox won't let you.

Using factories from the internet
*********************************

If things weren't already awesome enough, you can load your factories straight
from the web or Git. If you have a factory up at GitHub, you could use it via::

  rootbox box.new -n mybox -v 3.5 -f git:myuser/myrepo@@mybranch///myfactory.sh

If ``@@mybranch`` is ommited, it defaults to *master*. If ``///myfactory.sh``
is ommited, it defaults to ``factory.sh``. For instance, to load ``factory.sh``
from ``CoolRootboxScripts/cool_scripts_set_1``::

  rootbox box.new -n mybox -v 3.5 -f git:CoolRootboxScripts/cool_scripts_set_1

To use ``my_other_factory.sh``::

  rootbox box.new -n mybox -v 3.5 -f git:CoolRootboxScripts/cool_scripts_set_1///my_other_factory.sh

To use it from the branch ``devel``::

  rootbox box.new -n mybox -v 3.5 -f git:CoolRootboxScripts/cool_scripts_set_1@@devel///my_other_factory.sh

GitLab is supported, too::

  rootbox box.new -n mybox -v 3.5 -f gitlab:MyGitlabUser/my_gitlab_repo@@branch///factory_name.sh

as well as any other plain old Git repository::

  rootbox box.new -n mybox -v 3.5 -f git:https://whatever.com/my_repo.git@@branch///factory_name.sh

In fact, factories can be pulled from anywhere on the internet::

  rootbox box.new -n mybox -v 3.5 -f url:https://mysite.com/some_cool_factory.sh

The syntax this time is a bit different: the url must point to an absolute URL
to the factory. If the URL ends with a slash (``/``), then ``factory.sh`` will
be appended to it.

These location formats can be used inside the ``DEPENDS`` section of a script,
too. You could have something like this:

.. code-block:: shell

  # This is my cool factory script!
  #:DEPENDS git:myuser/myrepo///myfactory.sh
  #:DEPENDS url:rootbox_factories.com/myotherfactory.sh

Managing, exporting, and importing your boxes
*********************************************

If you want to see a list of all the boxes that have been installed, just run::

  rootbox box.list

Boxes can be cloned::

  rootbox box.clone -s source_box -n new_box

and deleted::

  rootbox box.remove -n mybox

More importantly, they can also be exported using ``box.dist``. It works much
like you'd expect by now::

  rootbox box.dist -n mybox

The default file name is ``<your_box_name>.box``. In this case, it'll be
``mybox.box``. That can be overriden, of course::

  rootbox box.dist -n mybox -o my_custom_name.box

In addition, you can apply compression using ``-c`` to make it a bit smaller::

  rootbox box.dist -n mybox -o gzip_compressed.box.gz -c gzip
  rootbox box.dist -n mybox -o bzip2_compressed.box.bz2 -c bzip2

Boxes can also be imported::

  rootbox box.import -n mybox -l mybox.box.gz

Here, ``mybox.box.gz`` is being imported using the name ``mybox``. In fact,
boxes can be imported from virtually anywhere, using the exact same syntax as
used with box factories::

  rootbox box.import -n mybox -l url:rootbox_storage.com/1234567
  rootbox box.import -n mybox -l git:Cooluser101/myboxes///cool_stuff.box

An example
**********

Building C
^^^^^^^^^^

Donald wants to create a box that can be used to statically compile his C
programs. Alpine Linux is great for static linking::

  rootbox box.new -n static_c -b .///cwd
  rootbox box.run -n static_c

  # Inside the box...
  cd /cwd
  echo 'int main() {}' > x.c
  gcc -static -o x x.c
  exit

  # Back outside again...
  ldd x  # statically linked

Building Nim
^^^^^^^^^^^^

Mary wants to create a box designed for building Nim programs. She can use
factories to automate...everything:

.. code-block:: shell

  sudo apk add xz linenoise-dev libexecinfo-dev

  curl -L https://nim-lang.org/download/nim-0.16.0.tar.xz -o nim.txz
  tar xvf nim.txz
  cd nim-0.16.0
  ./build.sh

  bin/nim c koch
  ./koch boot -d:release -d:useLinenoise
  sudo ./koch install /usr/local/bin

She can save this to ``nim-factory.sh``. Then, to create her image, she can
just run::

  rootbox box.new -n nim -b .///cwd -f nim-factory.sh

If she uploads her Nim factory to GitHub in ``mary123/factories``, someone else
can use it, too::

  rootbox box.new -n nim -b .///cwd -f git:mary123/factories///nim-factory.sh

If Robby wants to build a Nim program using Rootbox, he can create a factory,
too:

.. code-block:: shell

  #:DEPENDS git:mary123/factories///nim-factory.sh
  git clone https://github.com/bobby456/my-nim-program.git
  cd my-nim-program
  nim c my-program.nim
  sudo cp my-program /usr/local/bin

Closing thoughts
****************

Rootbox is still in the beta stages. If you notice anything isn't working quite
correctly, feel free to report it to the
`GitHub repo <https://github.com/project-rootbox/rootbox/issues/new>`_.

Have fun playing with your boxes!!!
