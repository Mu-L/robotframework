Creating Robot Framework releases
=================================

These instructions cover steps needed to create new Robot Framework releases.
Many individual steps are automated, but we don't want to automate
the whole procedure because it would be hard to react if something goes
terribly wrong. When applicable, the steps are listed as commands that can
be copied and executed on the command line.

.. contents::
   :depth: 1

Preconditions
-------------

Operating system and Python requirements
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Generating releases has only been tested on Linux, but it ought to work the
same way also on OSX and other unixes. Generating releases on Windows may
work but is not tested, supported, or recommended.

Creating releases is only supported with Python 3.6 or newer. If you are
using Ubuntu or one of its derivatives and don't have Python 3.6 in the
official package repository, you may consider using the
`Dead Snakes PPA <https://launchpad.net/~deadsnakes/+archive/ubuntu/ppa>`_.

The ``pip`` and ``invoke`` commands below are also expected to run on Python
3.6+. Alternatively, it's possible to use the ``python3.6 -m pip`` approach
to run these commands.

Python dependencies
~~~~~~~~~~~~~~~~~~~

Many steps are automated using the generic `Invoke <http://pyinvoke.org>`_
tool with a help by our `rellu <https://github.com/robotframework/rellu>`_
utilities, but also other tools and modules are needed. A pre-condition is
installing all these, and that's easiest done using `pip
<http://pip-installer.org>`_ and the provided `<requirements-dev.txt>`_ file::

    pip install -r requirements-dev.txt

Using Invoke
~~~~~~~~~~~~

Invoke tasks are defined in the `<tasks.py>`_ file and they are executed from
the command line like::

    inv[oke] task [options]

Run ``invoke`` without arguments for help. All tasks can be listed using
``invoke --list`` and each task's usage with ``invoke --help task``.

Different Git workflows
~~~~~~~~~~~~~~~~~~~~~~~

Git commands used below always expect that ``origin`` is the project main
repository. If that's not the case, and instead ``origin`` is your personal
fork, you probably still want to push to the main repository. In that case
you need to add ``upstream`` or similar to ``git push`` commands before
running them.

Testing
-------

Make sure that adequate tests are executed before releases are created.
See `<atest/README.rst>`_ for details.

If output.xml `schema <doc/schema/README.rst>`_ has changed, remember to
run tests also with `full schema validation`__ enabled::

    atest/run.py --schema-validation

__ https://github.com/robotframework/robotframework/tree/master/atest#schema-validation

Preparation
-----------

1. Check that you are on the right branch and have nothing left to commit,
   pull, or push::

      git branch
      git status
      git pull --rebase
      git push

2. Make sure code is formatted properly::

      invoke format
      git status

3. Clean up::

      invoke clean

4. Set version information to a shell variable to ease copy-pasting further
   commands. Add ``aN``, ``bN`` or ``rcN`` postfix if creating a pre-release::

      VERSION=<version>

   For example, ``VERSION=7.1.1`` or ``VERSION=7.2a2``. No ``v`` prefix!

Release notes
-------------

1. Create a personal `GitHub access token`__ to be able to access issue tracker
   programmatically. The token needs only the ``repo/public_repo`` scope.

2. Set GitHub user information into shell variables to ease running the
   ``invoke release-notes`` command in the next step::

      GITHUB_USERNAME=<username>
      GITHUB_ACCESS_TOKEN=<token>

   ``<username>`` is your normal GitHub user name and ``<token>`` is the personal
   access token generated in the previous step. Alternatively this information can
   be given when running the command in the next step.

3. Generate a template for the release notes::

      invoke release-notes -w -v $VERSION -u $GITHUB_USERNAME -p $GITHUB_ACCESS_TOKEN

   The ``-v $VERSION`` option can be omitted if `version is already set
   <Set version_>`__. Omit the ``-w`` option if you just want to get release
   notes printed to the console, not written to a file.

   When generating release notes for a preview release like ``7.2rc1``,
   the list of issues is only going to contain issues with that label
   (e.g. ``rc1``) or with a label of an earlier preview release (e.g.
   ``alpha1``, ``beta2``).

4. Fill the missing details in the generated release notes template.

5. Make sure that issues have correct information:

   - All issues should have type (bug, enhancement or task) and priority set.
     Notice that issues with the task type are automatically excluded from
     the release notes.
   - Issue priorities should be consistent.
   - Issue titles should be informative. Consistency is good here too, but
     no need to overdo it.

   If information needs to be added or edited, its better to edit it in the
   issue tracker than in the generated release notes. This allows re-generating
   the list of issues later if more issues are added.

6. Commit and push changes::

      git add doc/releasenotes/rf-$VERSION.rst
      git commit -m "Release notes for $VERSION" doc/releasenotes/rf-$VERSION.rst
      git push

7. Update later if necessary. Writing release notes is typically the biggest
   task when generating releases, and getting everything done in one go is
   often impossible.

__ https://docs.github.com/en/free-pro-team@latest/github/authenticating-to-github/creating-a-personal-access-token

Update Libdoc templates
-----------------------

1. Prerequisites are listed in `<src/web/README.md>`_. This step can be skipped
   if there are no changes to Libdoc.

2. Regenerate HTML template and update the list of supported localizations in
   the ``--help`` text::

      invoke build-libdoc

3. Commit and push changes::

      git commit -m "Update Libdoc templates" src/robot/htmldata/libdoc/libdoc.html src/robot/libdocpkg/languages.py
      git push

Set version
-----------

1. Set version information in `<src/robot/version.py>`_ and `<setup.py>`_::

      invoke set-version $VERSION

2. Commit and push changes::

      git commit -m "Updated version to $VERSION" src/robot/version.py setup.py
      git push

Tagging
-------

1. Create an annotated tag and push it::

      git tag -a v$VERSION -m "Release $VERSION"
      git push --tags

2. Add short release notes to GitHub's `releases page
   <https://github.com/robotframework/robotframework/releases>`_
   with a link to the full release notes.

Creating distributions
----------------------

1. Checkout the earlier created tag if necessary::

      git checkout v$VERSION

   This isn't necessary if continuing right after tagging_.

2. Cleanup (again). This removes temporary files as well as ``build`` and
   ``dist`` directories::

      invoke clean

4. Create and validate source distribution and `wheel <https://pythonwheels.com>`_::

      python setup.py sdist bdist_wheel
      ls -l dist
      twine check dist/*

   Distributions can be tested locally if needed.

5. Upload distributions to PyPI::

      twine upload dist/*

6. Verify that project pages at `PyPI
   <https://pypi.python.org/pypi/robotframework>`_ look good.

7. Test installation::

      pip install --pre --upgrade robotframework

8. Documentation

   - For a reproducible build, set the ``SOURCE_DATE_EPOCH``
     environment variable to a constant value, corresponding to the
     date in seconds since the Epoch (also known as Epoch time).  For
     more information regarding this environment variable, see
     https://reproducible-builds.org/docs/source-date-epoch/.

   - Generate library documentation::

       invoke library-docs all

   - Create User Guide package::

       doc/userguide/ug2html.py zip

   - Update docs at http://robotframework.org/robotframework/::

        git checkout gh-pages
        invoke add-docs $VERSION --push
        git checkout master    # replace master with v*-maintenance if needed!

Post actions
------------

1. Back to master if needed::

      git checkout master    # replace master with v*-maintenance if needed!

2. Set dev version based on the previous version::

      invoke set-version dev
      git commit -m "Back to dev version" src/robot/version.py setup.py
      git push

   For example, ``1.2.3`` is changed to ``1.2.4.dev1`` and ``2.0.1a1``
   to ``2.0.1a2.dev1``.

3. Close the `issue tracker milestone
   <https://github.com/robotframework/robotframework/milestones>`_.
   Create also new milestone for the next release unless one exists already.

4. Update API doc version at https://readthedocs.org/projects/robot-framework/.

Announcements
-------------

1. ``#announcements`` channel on `Slack <https://slack.robotframework.org/>`_.
   Use ``@channel`` at least with major releases.

2. `Forum <https://forum.robotframework.org/>`_.

3. `LinkedIn group <https://www.linkedin.com/groups/3710899/>`_. A personal
   LinkedIn post is a good idea at least with bigger releases.

4. `robotframework-users <https://groups.google.com/group/robotframework-users>`_
