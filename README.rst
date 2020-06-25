What is peg?
============

peg stands for PostgreSQL Environment Generator.  It's a fairly complicated
script that automates most of the repetitive work required to install test
instances of PostgreSQL.  It's primarily aimed at developers who are building
from source control (git, cvs) and toward reviewers who are trying out a
source code package (.tar.gz), including release source snapshots and the
alpha/beta builds.  It works on UNIX-like systems that already have
the rest of the build tools necessary for compiling PostgreSQL installed.

Motivation
==========

peg usage relies heavily on environment variables.  As much as possible, it
mirrors the environment setups of two common PostgreSQL setups:  psql and
the RPM packaging of the program.

The psql client and other programs that use the libpq library look for
environment variables such as PGPORT to change their behavior.  Server
startup does some of this as well, such as using PGDATA to set the database
location.

The RPM packaging used by RedHat's PostgreSQL server uses a sourced
environment stored in /etc/sysconfig/pgsql/postgresql[version] which
can define settings like PGDATA.  One useful idiom when working with
postgresql is to have the login profile of the postgres user and others
using Postgres to source this file, and therefore have the exact same
settings used to start the server.  That file might look like this::

  PGENGINE=/usr/pgsql-9.1/bin
  PGPORT=5432
  PGDATA=/var/lib/pgsql/9.1/data
  PGLOG=/var/lib/pgsql/9.1/pgstartup.log

To make these available to the user running the database, typically
postgres, that user's login profile would execute a shell command like
this::

  source /etc/sysconfig/pgsql/postgresql-9.1
   
The idea is that once you've setup an environment with peg, the environment
variables you'll have available to you will match what you'd get if you
were logging into a production RHEL server running the standard PostgreSQL
RPM set.  Once peg has setup your environment, commands like the following
will work, exactly the same as if you'd sourced the sysconfig file on
a production RHEL server::

  pg_ctl start -l $PGLOG
  $EDITOR $PGDATA/postgresql.conf

Whether or not you find that useful, you'll also get source code builds and
cluster creation, setup, and control automated by using peg.

peg takes commands similarly to common version control systems:  you call
the one executable script with a subcommand, followed by what you want it to
operate on, which right now is always a project name.

Installation
============

peg is so far a single script; its only dependency is Bash.  You
can just copy or link it to any directory that's in your path, such
as /usr/local/bin for a system-wide install.  Here's a sample
installation that checks out a working copy and then
links to that copy in a standard directory found in the PATH
on Linux systems::

  $ cd $HOME
  $ git clone git://github.com/gregs1104/peg.git
  $ sudo ln -s $HOME/peg/peg /usr/local/bin/

Creating a first project
========================

The easiest way to make peg work is to create a $HOME/pgwork directory and
let peg manage the whole thing for you::

  # git repo setup example (you must be running bash)
  cd
  mkdir -p pgwork
  peg init test
  . peg build
  psql

At this point your sole git repository will be switched to a new branch that
matches the name of the project.

Directory layout
----------------

peg assumes you are installing your Postgres database as a regular user,
typically in your home directory.  It allows you have to multiple "projects"
installed in this structure, each of which gets its own source code, binaries,
and database.

The example above will get you a directory tree that looks like this
(this is simplified to only include the basic outline, there will be
more directories below these)::

  pgwork/
  |-- data
  |   `-- test
  |-- inst
  |   `-- test
  |       `-- bin
  |-- repo
  |   `-- git
  `-- src
      `-- test

There will only be one repo directory for each of the projects hosted in
this work area, and each project will checkout its own unique copy of that
repo into its work area--unless you are using git; see below.

The top level directories are intended to work like this:

 * pgwork/repo:  The one copy of the master repo all projects share
 * pgwork/src/project:  Checked out copy of the repo with this project's source
 * pgwork/inst/project:  Binaries build by the "make install" step
 * pgwork/data/project:  Hold the database cluster created by initdb

Shortcuts
---------

Once you've gotten an environment setup, using it again can be as easy as::

  . peg switch
  
That will use whatever project name you last gave the script and start the
server up.  If you shutdown your system, that should be all that's needed
to bring things back up again.

When you source the peg script, along with getting settings like PGDATA
available it also creates a couple of quick aliases you can use instead
of calling peg for those functions:

 * start:  If you already have a database cluster setup, start it
 * stop:  Stop a running database with the slow shutdown code
 * immediate:  Stop a running database immediately

Here again, the names were picked to be similar to the 
"service postgresql start" and stop commands used by the RPM packaging.
start and stop are used on some UNIX systems for job control or system
initialization.  It's unlikely you'll need those while doing PostgreSQL
work too, so re-using those commands for this should save you some typing.

Use peg for performance testing
-------------------------------

The default build method used by peg includes assertions, which will
slow down the speed of the resulting server code considerably.  If you
want to build without assertions and debugging information, you'll need
to set PGDEBUG to a non-empty value.  That will be passed through to
the PostgreSQL "configure" program without turning on any of the
debugging features.  A space works for this, for example:: 
   
  export PGDEBUG=" "  

Before the build step will use the standard build options, rather than
the debugging ones that slow the server down.

peg with git details
====================
 
Source code layout for git
--------------------------

If you are using git for your repo, the src/ directory is just a symbolic link
to the repo itself, so that every project shares the same repo.  Each project
is instead given its own branch when you first use "peg init".  This seems
to match the workflow of git users better than checking out a separate copy
for each project.  This is simple enough to change:  in between the init and
build steps, you can remove the symlink and manually copy the master repo.

TODO:  Provide an example of that

If you intend to work on multiple projects using a single git repo, make
sure you note the "Known Issues" section below for caveats about
common problems, and how to resolve them.

Applying a patch to a git repo project
--------------------------------------

Here's how you might test a patch using git for the base repo::

  peg init test
  cd pgwork/src/test
  git apply some.patch
  . peg build
  psql

Some patches aren't handled by git's apply.  If that fails with errors,
try the following instead::

  patch -p1 < some.patch

The parameter passed to "-p" in this case can vary; 0 is also common.
You'll need to be able to read the patch to predict what it should be.

See http://wiki.postgresql.org/wiki/Working_with_Git for more
information about how to deal with patches, as well as other aspects of
PostgreSQL plus git use.

Sample tgz session
==================

Here's how you might use peg to test out an alpha or beta build
downloaded in source code form.  To do that instead of using a regular
repo, you merely need to create a tgz repo directory and drop the file
into there::

  # Repo setup:  tgz
  cd
  mkdir -p pgwork/repo/tgz
  cp postgresql-9.1alpha1.tar.gz pgwork/repo/tgz

  # Basic install
  peg init test
  . peg build
  psql

cvs or tgz repo with patch
--------------------------

Here's how you might test a patch using CVS for the base repo::

  peg init test
  cd pgwork/src/test
  patch -p 0 < some.patch
  . peg build
  psql

TODO:  Test the above

Sample cvs session
==================

You can clone the postgresql.org cvs repo just by changing your default
PGVCS to be cvs::

  cd
  mkdir -p pgwork

  # Repo setup:  cvs
  export PGVCS=csv
  peg init test
  . peg build
  psql

This will synchronize with the master PostgreSQL git server via rsync, using
the same techniques documented at
http://wiki.postgresql.org/wiki/Working_with_CVS
(The outline given in its "Initial setup" section is actually peg's distant
ancestor)  The main reason why you might want to use CVS is if you
are doing development on an older server where git cannot be installed.

You can easily force this just by creating a repo/cvs directory too::

  cd
  mkdir -p pgwork/repo/cvs
  peg init test
  . peg build
  psql

Sample two-cluster session
==========================

Here is a complicated peg installation.  The intent is to start two database
clusters that shared the same source code and binary set, perhaps for testing
replication with two "nodes" on a single server.  This is relatively easy
to script, using peg to do most of the dirty work normally required here::

  # Two node cluster setup
  peg init master
  peg init slave

  # Make the slave use the same source code and binaries as the slave
  pushd pgwork/inst
  rm -rf slave
  ln -s master slave
  popd

  pushd pgwork/src
  rm -rf slave
  ln -s master slave
  popd

  # Start the master
  peg build master
  # Can't source the above yet, because then PGDATA will be set
  # Start the slave
  export PGPORT=5433 ; peg start slave ; export PGPORT=
  . peg switch master

  psql -p 5432 -c "show data_directory"
  psql -p 5433 -c "show data_directory"

Note that if you now try to stop the slave like this::

  peg stop slave

This won't actually work, because it will be still using the PGDATA
environment variable you sourced in.  Instead you need to do this::

  unset PGDATA PGLOG
  . peg switch slave
  peg stop

TODO:  The above still doesn't work.  But if you start a whole new shell,
that seems to be fine.

Sample backporting setup
========================

Backporting involves taking code from a newer version of a program and
applying it to an earlier one.  For this example, imagine that the goal
is to apply a patch developed for PostgreSQL 9.2 to version 9.1.  This
only works in peg if you are using the default git repository, where it's
easy to checkout any version of the database code.

A backporting setup works just like a regular git session, just setting
the PGVERSION environment variable first::

  export PGVERSION="9.1"
  cd
  mkdir -p pgwork
  peg init test

Now you can make the changes you have from a later version to the source
code, using something like the patch application example above.  In some
cases, source changes must be applied before compiling PostgreSQL at all.
Many source code patches can be applied and worked on after a build step
too.

If the change you're looking for is a feature added to a later PostgreSQL
version, you might apply it to the older version you have checked out
using "git cherry-pick" instead.

Once all the changes are made, build the database source code and test::

  . peg build
  psql

Base directory detection
========================

The entire peg directory tree is based in a directory recommended to be
named pgwork.  If you use another directory, you can make the script use
it by setting the PGWORK environment variable.  The sequence searched to find
a working area is:

  1. The value passed for PGWORK
  2. The current directory
  3. $HOME/pgwork
  4. $HOME

peg assumes it found a correct working area when there is a "repo"
subdirectory in one of these locations.

Command summary
===============

The following subcommands are accepted by peg:

 * status:  Report on environment variables that would be in place if you were
   to execute a command.  Useful mainly for troubleshooting.
 * init:  Create a repo and a project based on it, if one is named
 * update:  Update your repo and a project based on it, if one is named
 * build-only:  Execute the build steps and create a database cluster, but
   don't start it.  This is typically for if you know you need to modify the
   database configuration before you start it.
 * build:  Build binaries, install them, create a cluster, start the database
 * rebuild:  Rebuild and install just the main binaries for the server in
   the src/backend directory.  When making changes to just the core server
   code, this can save time over doing a full build.
 * initdb:  Create a cluster
 * switch:  Switch to an existing built binary set and cluster
 * start:  Start a cluster
 * stop:  Stop a cluster
 * rm:  Remove all data from a project (but not the repo)

Environment variable reference
==============================

You can see the main environment variables peg uses internally with::

  peg status <project>
  
All of those values are set automatically only if you don't explicitly set
them to something first.  This allows you to do things like use peg to
manage your source and binary builds, while still having a PGDATA that
points to a separate location where you want your database to go.  

 * PGPORT:  Client programs use this to determine what port to connect on;
   if set, any "peg start" commands will start the server on that port.  See
   the multi-cluster example for how this might be used.
 * PGVCS:  Valid options are "cvs", "git", and "tgz".  If you have more
   than one type of source in your repo directory, you can
   use this to specify which of them you should use.
 * PGWORK:  Base directory for working area.  See "Base directory detection".
 * PGPROJECT:  If this is set, it will become the project name used
   for all commands, regardless of what's passed on the command line.
 * PGDEBUG:  By default, peg builds PostgreSQL with the standard flags
   you'd want to use for development and testing.
 * PGMAKE:  Program to run GNU make.  This defaults to "make" but can be
   overridden.
 * PGVERSION:  Stable version of PostgreSQL to checkout, when a new project
   is created with "peg init".  If not specified, the "master" branch of the
   repository is used.  Only useful if peg is running against a git repo.
   This will accept either standard version numbers like "9.1", or version
   names that match the actual stable branch naming conventions like "9_1".

Solaris Use
===========

The defaults for peg are known to have issues building on a typical Solaris
system, where the GNU building toolchain is not necessarily the default one.
Here's a sample configuration you can put into your environment to change
the defaults to work on that platform, assuming you've installed the
Sun Freeware GNU tools in their default location::

  export PGMAKE="/usr/sfw/bin/gmake"
  export PGDEBUG="--enable-cassert --enable-debug --without-readline --with-libedit-preferred"

Known Issues
============

See TODO notes in the peg source code (and even this documentation) for the
open issues in the code.  A few of these turn into functional issues you
should be aware of.

git Branching and Cleanup
-------------------------

When using git, peg links all projects to a single git directory, with each
project treated as a branch.  The program expects that you'll manage
complicated operations here on your own, rather than trying to force git
changes that can potentially be destructive.  One area this can cause
many problems is if you're trying to switch to a new origin branch,
such as when using the PGVERSION variable.  Moving from the default branch
(origin/master) to another version, or the reverse, will usually require
some manual cleanup of the git checkout before running "peg init".

You always want to be careful to commit any working code to your active
branch before trying to change to a new project, and therefore a new git
branch.  Checking the status of the repository checkout is a good habit to
adopt before running "peg init" to try and create a new project.

You can check if your git checkout is completely cleaned up--and therefore
able to accept a branch change with complain--by seeing if its status looks
like this::

  $ git status
  # On branch master
  nothing to commit (working directory clean)

If you see modified or untracked files there, a checkout that tries to change
origin branch version is unlikely to work.  A typical error if cleanup isn't
done correctly before "peg init" is many "needs merge" warnings ending
with::

  error: you need to resolve your current index first

Most files that cause this sort of problem can be cleaned up by going into
src directory--which is just a link to the repo directory when using peg with
git, and you can go there instead--and executing::

  git reset --hard

This will not remove new files that have been added though, which can still
cause you issues.  For example, if the same file names have also been used
in the new branch you're checking out, this will cause you some trouble.
One common way developer checkouts can get this sort of "Untracked files"
is if you build ctags to help navigate the source code.  All new files and
directories can be removed with::

  git clean -f -d

It's very important to move any important files you've added out of the git
directory tree before using "git clean" like this.  It will wipe everything
other than the expected repository files out.

Serious problems
----------------

So far these are serious only in the sense that you are likely to run into
them and the problems they cause are annoying.  But the workarounds to avoid
each are pretty simple.

* If you are running against a project, then create a new one, it's quite
  easy to get into a state where environment variables and other information
  set by the old project continue to linger around.  If you're using a git
  repo for the code, this is particularly likely to happen because
  switching projects only switches branches in the single shared checkout
  of the repo.  That doesn't remove the parts of the source code build
  configuration that refer to the old project:  the ``configure`` stage
  saves where the binaries are going to be stored at for example.
  The suggested workflow when using git is therefore::

     stop
     peg clean
     peg init newproject
     [start a new terminal session to clear all environment variables]
     peg build
     . peg switch

* peg has a notion that you might set PGDATA directly, rather than want that
  particular directory structure to be in the same PGWORK area everything
  else is at.  And when you source peg into your environment to use
  a project, this sets PGDATA.  This combination causes a major issue
  when switching projects that are in fact both hosted in the PGWORK
  structure.  You'll get the PGDATA from the original project, and the
  one you're switching to will believe that's a manually set PGDATA it
  should use.  So everything else will switch to the new project,
  except the database directory, which is confusing.  This problem
  will eventually be addressed in the code.  To work around
  it for now, before doing "peg switch" you should erase PGDATA (and PGLOG,
  which suffers from the same issue)::

     unset PGDATA PGLOG

Trivial bugs
------------

 * peg creates a database matching your name, which is what psql wants for a
   default.  It doesn't check whether it already exist first though, so you'll
   often see an error about that when starting a database.  This is harmless.
 * If you source peg output repeatedly, it will pollute your PATH with
   multiple pointers to the same directory tree.  This is mostly harmless, just
   slowing down how fast commands can be found in your PATH a bit.

Documentation
=============

The documentation ``README.rst`` for the program is in ReST markup.  Tools
that operate on ReST can be used to make versions of it formatted
for other purposes, such as rst2html to make a HTML version.

Contact
=======

The project is hosted at http://github.com/gregs1104/peg

If you have any hints, changes or improvements, please contact:

 * Greg Smith gregs1104@gmail.com

Credits
=======

Copyright (c) 2009-2020, Gregory Smith
All rights reserved.
See COPYRIGHT file for full license details

peg was written by Greg Smith to make all of the PostgreSQL systems he
works on regularly have something closer to a common UI at the console.
peg's directory layout and general design was inspired by several members
of the PostgreSQL community, including:

* Heikki Linnakangas, whose outlined his personal work habits for
  interacting with the CVS repo and inspired Greg to
  write the original "CVS+rsync Solutions" section of 
  http://wiki.postgresql.org/wiki/Working_with_CVS
* Alan Li, Jeff Davis, and other members of Truviso who I may not
  directly remember borrowing ideas from as much as those two.  Alan
  and Jeff both had their own way to organize PostgreSQL installations
  in their respective home directories that I found interesting when we
  worked together on projects.
