.. _developing-against-rpms:

Developing against RPMs
-----------------------

Install and start the server
^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Create a development environment as per `<vm-setup.html>`_
instructions and deploy your service as usual. We will use SiteDB
as an example server for this guide::

    (VER=HG1303b REPO="-r comp=comp.pre" A=/data/cfg/admin;
     PKGS="admin/devtools frontend sitedb/legacy";
     cd /data;
     $A/InstallDev -R cmsweb@$VER -s image -v $VER -a $PWD/auth $REPO -p "$PKGS")

    (A=/data/cfg/admin; cd /data; $A/InstallDev -s start)

Note the commands run within braces "``( )``" to avoid changing your
environment.

Since SiteDB uses an Oracle database, this assumes you have somehow acquired
the secrets information and have put them into ``$PWD/auth`` as described in
the vm-setup instructions. For SiteDB specifically this means installing the
right ``$PWD/auth/sitedb/SiteDBAuth.py``; other servers will require other
kinds of secrets, some require none at all. You can point the authentication
directory whereever you want, for example in a subdirectory under your AFS
home directory ``~/private``, it does not have to be under ``/data``.

Above $VER is the base configuration you are developing against. Usually this
should be the current pre-production release, which is always documented in the
monthly "CMSDIST validation" ticket and announced in the webInterfaces forum.

For this particular application, you'd normally deploy the ``legacy`` variant
as shown above. Not all applications have variants, you need to know the
application to know what should normally be deployed.

As usual you can choose a different RPM repository with
"``REPO='-r comp=comp.pre.me'``", or override the version to be installed
by using ``sitedb@2.4.1-rc1/legacy`` to install 2.4.1-rc1 instead of whatever
had been included in the release series $VER. This feature will be used later
in this development cycle to test your candidate RPMs.


Create development area
^^^^^^^^^^^^^^^^^^^^^^^

Once you have installed the server, create a development area as per `CMS
DM/WM instructions. <dev-git.html>`_.
Clone SiteDB from github; the following assumes you forked the repo and
are going to use your own github account to access it::

    # example dev area, can be anywhere
    cd /data/user
    git clone git@github.com:mygit/sitedb.git
    cd sitedb
    stg init

    git branch my-new-feature
    stg init


Modify code and rebuild
^^^^^^^^^^^^^^^^^^^^^^^

Modify the source code, creating your stg patch stack. For each patch you
can rebuild the server and apply your changes to the server you installed
earlier::

    (cd /data/user/sitedb;
     . /data/current/config/admin/init.sh;
     . /data/current/apps/sitedb/etc/profile.d/env.sh;
     wmc-dist-patch --skip-docs --compress)

Here we speed up the build by skipping sphinx documentation generation, and
generate a production-like server by compressing HTML, CSS and javascript
assets. Dropping the ``--compress`` option installs assets uncompressed and
automatically puts the javascript side into a debug mode, which is useful
for being able to debug the server.

Run the following to restart the server, then observe logs to inspect impact
of your changes::

    (A=/data/cfg/admin; cd /data; $A/InstallDev -s start:sitedb)
    tail -f /data/logs/sitedb/*.log

You always have to restart the server if you've modified the python source
code. If you are only modifying the HTML/CSS/JavaScript assets, you do not
have to restart the server, it's enough to reload the page in a browser.
However whenever you switch between compressed and uncompressed assets, you
do need to restart the web server so it knows to switch file sets.

Note that for the above patch scheme to work, the RPM must generate the
required parallel but empty structure. Basically there are empty *xbin,
xlib, xdata, xdoc,* etc. directories in the RPM, which have been inserted
in the front of $PATH, $LD_LIBRARY_PATH, $PYTHONPATH, etc. The web servers
have also been instructed to prefer HTML, CSS, image and JavaScript assets
from the *xdata* over *data.* The patch step is as simple as installing
*setup.py* products into these parallel directories, and unpatching simply
removes them.

**Note that you never, ever, manually modify the server sources in the RPM
install tree!** You only modify sources in your git checkout area, and let
*setup.py* do the installation into the software area.


Roll back your server
^^^^^^^^^^^^^^^^^^^^^

If you want to revert your changes and reset the server back to what it was
when installed from RPMs, un-patch and restart it. This is necessary at the
very least before you re-deploy the server, as the deployment will otherwise
complain about the extra files::

    (cd /data/user/sitedb;
     . /data/current/config/admin/init.sh;
     . /data/current/apps/sitedb/etc/profile.d/env.sh;
     wmc-dist-unpatch)
    (A=/data/cfg/admin; cd /data; $A/InstallDev -s start:sitedb)


Building RPMs from your changes
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

**Important: at no point in the process do you need to wait
for someone else to complete development or testing.** There are other
reasons you need to synchronise with other developers, but you *never*
need to wait on others to complete development cycle all the way down to
deploying RPMs with standard deployment recipes, and test your servers
with whatever test suite you have.

The steps above would normally cover 75% of your development work, merely
patching the server from the previous cycle with your new changes. For this
part of the development cycle you would not normally rebuild RPMs, which
is very convenient for speeding up development. Sooner or later you do
need to build RPMs to complete your testing. In particular you have to
rebuild RPMs if you modify the external dependencies of your server.

The RPM builds typically get split into two distinct phases for efficiency.
In the first phase one builds strictly private RPMs out of sources still
under development. At this point neither ``cmsdist`` changes nor application
sources are uploaded to a public repository, and the RPMs are not shared
with anyone. This allows near complete test of servers with standard RPMs,
even before pursuing patch review completion. It's not at all uncommon to
test features extensively and submit the pull request only once the new
features or bug fixes have passed thorough testing, increasing the efficiency
of the cycle. The second phase is typically just a final verification
everything still works when pulling RPM sources from official repository.

In the first phase, you replicate your git repository to a build server
and build the RPM from the sources there. You upload those RPMs to your
private RPM repository, then install the RPMs into a dev virtual machine.
If necessary you repeat the patch process as described above, redoing the
RPMs every once in a while. After you're satisfied with the RPMs you've
built, you do any final polish on the patches, push them to your public
git repo, and issue a pull request. You then either wait for patch review
to complete, or change RPMs to build out of your repository on github,
then submit a new deployment request to the standard ``cmsdist`` ticket for
the next pre-production deployment.

When you have the process sorted out, you should typically spend ~75% of
all your development time purely in the patch cycle, without bothering
with any RPM builds at all. About 20% of the time would normally occur
in the RPM polish cycle, and only 4% in the stage of testing final RPMs.
Only 1% of the time should be spent in verifying pre-production servers.
In short, if you run into signficant issues while testing final RPMs, the
process is not working for you and you probably should look to revise
your methods so you're finding problems much earlier in the cycle.

Let's assume you've worked with above instructions and have cloned SiteDB,
our example service, into ``/data/user/sitedb``. Let's further assume you
are using stg to manage a patch stack, with some local patches on top of
the original tree. We'll first make sure all pending changes are included
in our patches, as temporary stg commits at the head::

    # go to your git repository
    cd /data/user/sitedb

    # check there are no pending changes, should say "nothing to commit"
    git status

    # if there were any, include them into top-most patch
    stg refresh

    # review patch stack, patches and commit messages
    stg series -a -d
    stg show -a

    # in my case the stack looks like this:
    + drop-index     | Dropping tables automatically drops indices.
    + drop-types     | Schema types to be dropped are auto-determined, not...
    > unused-schema  | Remove commented out schema which confuses automatic...

    # check history log and commit ids for builds
    git log -4 --abbrev-commit --pretty=oneline --decorate

    # in my case the output looks like this (cf. stg series output)
    0e019b2 (HEAD, refs/patches/master/unused-schema, master) Remove commented out...
    6b0ad92 (refs/patches/master/drop-types) Schema types to be dropped are auto-...
    28a5ba3 (refs/patches/master/drop-index) Dropping tables automatically drops...
    b3fb151 (tag: 2.4.0, origin/master, origin/HEAD, refs/bases/master) Clear DN...

Now push the tree to the build machine; remove the ``-n`` option when
you are comfortable this will do the right thing::

    ssh vocms106.cern.ch mkdir -p /build/$USER/sitedb
    rsync -nzcav --delete ./ vocms106.cern.ch:/build/$USER/sitedb/

On a separate shell window, login to the build server, here vocms106,
and check out ``cmdist`` and ``pkgtools`` according to the $VER you are using.
See the `DMWM builds page
<https://twiki.cern.ch/twiki/bin/view/CMS/DMWMBuilds>`_ to find out
which build server to connect to and the ``pkgtools`` tag to use.::

    cd /build/$USER
    git clone -b V00-21-XX https://github.com/cms-sw/pkgtools.git
    (git clone -b comp https://github.com/cms-sw/cmsdist.git && cd cmsdist && git reset --hard HG1303b)
    head -1 cmsdist/sitedb.spec
      # mine outputs: '### RPM cms sitedb 2.4.0'

We now change the ``cmsdist`` to build an updated RPM. First change the version
tag on the first line; here we use version 2.4.1-rc1 since we're making a
the first release candidate RPM which is a small bug fix to 2.4.0 release.
We also change the ``Source`` line to pull the top-most stg patch from the
git repository we replicated to the build system::

    $ vi cmsdist/sitedb.spec
      -> ### RPM cms sitedb 2.4.1-rc1
      -> Source1: git:/build/lat/sitedb?obj=master/0e019b2&export=%n&output=/%n.tar.gz

Note above the ``obj=master/0e019b2`` which references the commit from our
local git tree's stg patch stack. This is still all work in progress not
committed to the official repository, and we may still change the patches,
but we can already build a RPM from them. Alternatively you could of course
just *tar* up your local source tree and put the tarball somewhere *cmsBuild*
can download it.

Now let's build this against ``comp.pre`` RPM repository::

    pkgtools/cmsBuild -c cmsdist --repository comp.pre \
      -a slc5_amd64_gcc461 --builders 8 -j 5 --work-dir w \
      build cmsweb

If all goes well the output will be like this::

    Package cms+sitedb+2.4.1-rc1 requested. [...]
    [1336684664.68] Starting to build cms+sitedb+2.4.1-rc1, log can be found in ...
    No more packages to build. Waiting for all build threads to complete their job.
    [1336684685.75] Processing cms+sitedb-webdoc+2.4.1-rc1.
    [1336684685.75] Checking repository for previous built cms+sitedb-webdoc+2.4.1-rc1.
    No more packages to build. Waiting for all build threads to complete their job.

Next we upload this RPM to ``comp.pre.me`` private repository, where the *.me*
is your CERN AFS login account::

    pkgtools/cmsBuild -c cmsdist --repository comp.pre \
      -a slc5_amd64_gcc461 --builders 8 -j 5 --work-dir w --upload-user=$USER \
      upload cmsweb

Note that ``comp.pre.me`` is automatically implied from ``--repository comp.pre``.
You should **not** tell it explicitly to the ``--repository`` argument.

If the upload succeeds, the output will be something like::

    Package cms+sitedb+2.4.1-rc1 requested. [...]
    No more packages to build. Waiting for all build threads to complete their job.
    No more packages to build. Waiting for all build threads to complete their job.
    Ready to upload.
    Creating temporary repository.
    Uploading packages.
    Regenerating apt db.

We can now use the private RPM repository and the uploaded RPM for installation.
Just go back to the beginning of this tutorial and override the repository name
with ``REPO="-r comp=comp.pre.$USER"`` and the service version with
``sitedb@2.4.1-rc1/legacy``. That is::

    (A=/data/cfg/admin; cd /data; $A/InstallDev -s stop; crontab -r; killall python)

    (VER=HG1303b REPO="-r comp=comp.pre.lat" A=/data/cfg/admin;
     PKGS="admin/devtools frontend sitedb@2.4.1-rc1/legacy";
     cd /data;
     $A/InstallDev -R cmsweb@$VER -s image -v $VER -a $PWD/auth $REPO -p "$PKGS")

    (A=/data/cfg/admin; cd /data; $A/InstallDev -s start)

You can now test it, maybe iterate back to `Modify code and rebuild`_ to
update the patches and re-patch the server, then rebuild RPMs each time
bumping the release candidate ``-rcN`` number.

Obviously at the same time you may be developing accompanying patch stack
of changes to the ``deployment`` tree. Just make sure you set $A to point
to it, or alternatively rsync your changes to /data/cfg. Then redeploy
with it in order to test them.

Only at the very end of this cycle you need to commit your git repository,
your ``cmsdist`` changes, and deployment script changes, and submit requests to
pull those to official repos and apply tags as appropriate. This helps
leave your change history clean with well-tested modifications, and you can
almost certainly spin a fully functional version of your server at any point
in the process. This in turns considerably reduces the risk in making the
buggy releases.

A slightly more advanced version of this cycle is that you keep your git
repository and cmsdist on your own desktop/laptop system, where you can
edit the sources with a local editor, then rsync the trees to the dev-vm
and build servers. You can actually do almost everything on a laptop,
without any need to use a dev-vm or a build server. That is actually what
I personally do, using the dev-vm only towards the end of the development
cycle.


Submit your changes
^^^^^^^^^^^^^^^^^^^

Once you are happy with your changes, follow the `DM/WM instructions
<dev-git.html>`_ to push your changes to a branch and send a pull request.

You are generally expected to structure the changes as a series of change
sets where each change is an individual atomic modification which makes
sense in its own right, for example adding a new feature, or fixing one
bug. In particular you should not submit the gory details of all the
erring paths of your development as atomic changes. Please review your
change history and squash change sets together where meaningful.

Patches making feature changes should not include formatting changes, and
in particular, should not include pure white space changes. Please verify
"``git diff --check``" does not warn on your patches, and specifically does
not flag trailing white space problems. You can see the latter in red if
you enable git colour diff mode. If you use stg, fixing problems with
patches is trivial, otherwise if it's your topmost patch, you can use
"``git commit --amend``" to update the patch to fix up small issues.

Patches are expected to follow a programming style used elsewhere in the
code, and specifically in any surrounding code. This applies to both
python and javascript. Patches which don't follow conventions for use of
white space, naming of things, and general code structuring or layout
have high probability of getting rejected with request to reformat.
