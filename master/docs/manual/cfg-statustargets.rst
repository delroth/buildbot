.. -*- rst -*-
.. _Status-Targets:

Status Targets
--------------

The Buildmaster has a variety of ways to present build status to
various users. Each such delivery method is a `Status Target` object
in the configuration's ``status`` list. To add status targets, you
just append more objects to this list::

    c['status'] = []
    
    from buildbot.status import html
    c['status'].append(html.Waterfall(http_port=8010))
    
    from buildbot.status import mail
    m = mail.MailNotifier(fromaddr="buildbot@localhost",
                          extraRecipients=["builds@lists.example.com"],
                          sendToInterestedUsers=False)
    c['status'].append(m)
    
    from buildbot.status import words
    c['status'].append(words.IRC(host="irc.example.com", nick="bb",
                                 channels=[{"channel": "#example1"},
                                           {"channel": "#example2",
                                            "password": "somesecretpassword"}]))

Most status delivery objects take a ``categories=`` argument, which
can contain a list of `category` names: in this case, it will only
show status for Builders that are in one of the named categories.

.. note:: Implementation Note

    Each of these objects should be a :class:`service.MultiService` which will be attached
    to the BuildMaster object when the configuration is processed. They should use
    ``self.parent.getStatus()`` to get access to the top-level :class:`IStatus` object,
    either inside :meth:`startService` or later. They may call
    :meth:`status.subscribe()` in :meth:`startService` to receive notifications of
    builder events, in which case they must define :meth:`builderAdded` and related
    methods. See the docstrings in :file:`buildbot/interfaces.py` for full details.

.. _WebStatus:

WebStatus
~~~~~~~~~

.. py:class:: buildbot.status.web.baseweb.WebStatus

The :class:`buildbot.status.html.WebStatus` status target runs a small
web server inside the buildmaster. You can point a browser at this web
server and retrieve information about every build the buildbot knows
about, as well as find out what the buildbot is currently working on.

The first page you will see is the *Welcome Page*, which contains
links to all the other useful pages. By default, this page is served from
the :file:`status/web/templates/root.html` file in buildbot's library area.
If you'd like to override this page or the other templates found there,
copy the files you're interested in into a :file:`templates/` directory in
the buildmaster's base directory.

One of the most complex resource provided by :class:`WebStatus` is the
*Waterfall Display*, which shows a time-based chart of events. This
somewhat-busy display provides detailed information about all steps of all
recent builds, and provides hyperlinks to look at individual build logs and
source changes. By simply reloading this page on a regular basis, you will see
a complete description of everything the buildbot is currently working on.

A similar, but more developer-oriented display is the `Grid` display.  This
arranges builds by :class:`SourceStamp` (horizontal axis) and builder (vertical axis),
and can provide quick information as to which revisions are passing or failing
on which builders.

There are also pages with more specialized information. For example,
there is a page which shows the last 20 builds performed by the
buildbot, one line each. Each line is a link to detailed information
about that build. By adding query arguments to the URL used to reach
this page, you can narrow the display to builds that involved certain
branches, or which ran on certain :class:`Builder`\s. These pages are described
in great detail below.

.. _WebStatus-Configuration:

Configuration
+++++++++++++

Buildbot now uses a templating system for the web interface. The source
of these templates can be found in the :file:`status/web/templates/` directory
in buildbot's library area. You can override these templates by creating
alternate versions in a :file:`templates/` directory within the buildmaster's
base directory.

The first time a buildmaster is created, the :file:`public_html/`
directory is populated with some sample files, which you will probably
want to customize for your own project. These files are all static:
the buildbot does not modify them in any way as it serves them to HTTP
clients.

Note that templates in :file:`templates/` take precedence over static files in
:file:`public_html/`. ::

    from buildbot.status.html import WebStatus
    c['status'].append(WebStatus(8080))

Note that the initial :file:`robots.txt` file has Disallow lines for all of
the dynamically-generated buildbot pages, to discourage web spiders
and search engines from consuming a lot of CPU time as they crawl
through the entire history of your buildbot. If you are running the
buildbot behind a reverse proxy, you'll probably need to put the
:file:`robots.txt` file somewhere else (at the top level of the parent web
server), and replace the URL prefixes in it with more suitable values.

If you would like to use an alternative root directory, add the
``public_html=`` option to the :class:`WebStatus` creation::

    c['status'].append(WebStatus(8080, public_html="/var/www/buildbot"))

In addition, if you are familiar with twisted.web *Resource
Trees*, you can write code to add additional pages at places inside
this web space. Just use :meth:`webstatus.putChild` to place these
resources.

The following section describes the special URLs and the status views
they provide.

.. _Buildbot-Web-Resources:

Buildbot Web Resources
++++++++++++++++++++++

Certain URLs are `magic`, and the pages they serve are created by
code in various classes in the :file:`buildbot.status.web` package
instead of being read from disk. The most common way to access these
pages is for the buildmaster admin to write or modify the
:file:`index.html` page to contain links to them. Of course other
project web pages can contain links to these buildbot pages as well.

Many pages can be modified by adding query arguments to the URL. For
example, a page which shows the results of the most recent build
normally does this for all builders at once. But by appending
``?builder=i386`` to the end of the URL, the page will show only the
results for the `i386` builder. When used in this way, you can add
multiple ``builder=`` arguments to see multiple builders. Remembering
that URL query arguments are separated *from each other* with
ampersands, a URL that ends in ``?builder=i386&builder=ppc`` would
show builds for just those two Builders.

The ``branch=`` query argument can be used on some pages. This
filters the information displayed by that page down to only the builds
or changes which involved the given branch. Use ``branch=trunk`` to
reference the trunk: if you aren't intentionally using branches,
you're probably using trunk. Multiple ``branch=`` arguments can be
used to examine multiple branches at once (so appending
``?branch=foo&branch=bar`` to the URL will show builds involving
either branch). No ``branch=`` arguments means to show builds and
changes for all branches.

Some pages may include the Builder name or the build number in the
main part of the URL itself. For example, a page that describes Build
#7 of the `i386` builder would live at :file:`/builders/i386/builds/7`.

The table below lists all of the internal pages and the URLs that can
be used to access them.

``/waterfall``
    This provides a chronologically-oriented display of the activity of
    all builders. It is the same display used by the Waterfall display.
    
    By adding one or more ``builder=`` query arguments, the Waterfall is
    restricted to only showing information about the given Builders. By
    adding one or more ``branch=`` query arguments, the display is
    restricted to showing information about the given branches. In
    addition, adding one or more ``category=`` query arguments to the URL
    will limit the display to Builders that were defined with one of the
    given categories.
    
    A ``show_events=true`` query argument causes the display to include
    non-:class:`Build` events, like slaves attaching and detaching, as well as
    reconfiguration events. ``show_events=false`` hides these events. The
    default is to show them.
    
    By adding the ``failures_only=true`` query argument, the Waterfall is
    restricted to only showing information about the builders that
    are currently failing. A builder is considered failing if the
    last finished build was not successful, a step in the current
    build(s) is failing, or if the builder is offline.
    
    The ``last_time=``, ``first_time=``, and  ``show_time=``
    arguments will control what interval of time is displayed. The default
    is to show the latest events, but these can be used to look at earlier
    periods in history. The ``num_events=`` argument also provides a
    limit on the size of the displayed page.
    
    The Waterfall has references to resources many of the other portions
    of the URL space: :file:`/builders` for access to individual builds,
    :file:`/changes` for access to information about source code changes,
    etc.

``/grid``
    This provides a chronologically oriented display of builders, by
    revision.  The builders are listed down the left side of the page,
    and the revisions are listed across the top.
    
    By adding one ore more ``category=`` arguments the grid will be
    restricted to revisions in those categories.
    
    A :samp:`width={N}` argument will limit the number of revisions shown to *N*,
    defaulting to 5.
    
    A :samp:`branch={BRANCHNAME}` argument will limit the grid to revisions on
    branch *BRANCHNAME*.

``/tgrid``
    The Transposed Grid is similar to the standard grid, but, as the name
    implies, transposes the grid: the revisions are listed down the left side
    of the page, and the build hosts are listed across the top.  It accepts
    the same query arguments. The exception being that instead of ``width``
    the argument is named ``length``.

    This page also has a ``rev_order=`` query argument that lets you
    change in what order revisions are shown. Valid values are ``asc``
    (ascending, oldest revision first) and ``desc`` (descending,
    newest revision first).


``/console``
    EXPERIMENTAL: This provides a developer-oriented display of the the last
    changes and how they affected the builders.
    
    It allows a developer to quickly see the status of each builder for the
    first build including his or her change. A green box means that the change
    succeeded for all the steps for a given builder. A red box means that
    the changed introduced a new regression on a builder. An orange box
    means that at least one of the test failed, but it was also failing
    in the previous build, so it is not possible to see if there was any
    regressions from this change. Finally a yellow box means that the test
    is in progress.
    
    By adding one or more ``builder=`` query arguments, the Console view is
    restricted to only showing information about the given Builders.Adding a
    ``repository=`` argument will limit display to a given repository. By
    adding one or more ``branch=`` query arguments, the display is restricted
    to showing information about the given branches. In addition, adding one or
    more ``category=`` query arguments to the URL will limit the display to
    Builders that were defined with one of the given categories.  With the
    ``project=`` query argument, it's possible to restrict the view to changes
    from the given project.
    
    By adding one or more ``name=`` query arguments to the URL, the console view is
    restricted to only showing changes made by the given users.
    
    NOTE: To use this page, your :file:`buildbot.css` file in :file:`public_html`
    must be the one found in
    :file:`master/buildbot/status/web/files/default.css`. This is the
    default for new installs, but upgrades of very old installs of
    Buildbot may need to manually fix the CSS file.

    
    The console view is still in development. At this moment by
    default the view sorts revisions lexically, which can lead to odd
    behavior with non-integer revisions (e.g., git), or with integer
    revisions of different length (e.g., 999 and 1000). It also has
    some issues with displaying multiple braches at the same time. If
    you do have multiple branches, you should use the ``branch=``
    query argument.  The ``order_console_by_time`` option may help
    sorting revisions, although it depends on the date being set
    correctly in each commit::

    
        w = html.WebStatus(http_port=8080, order_console_by_time=True)

``/rss``
    This provides a rss feed summarizing all failed builds. The same
    query-arguments used by 'waterfall' can be added to filter the
    feed output.

``/atom``
    This provides an atom feed summarizing all failed builds. The same
    query-arguments used by 'waterfall' can be added to filter the feed
    output.

``/json``
    This view provides quick access to Buildbot status information in a form that
    is easiliy digested from other programs, including JavaScript.  See
    ``/json/help`` for detailed interactive documentation of the output formats
    for this view.

:samp:`/buildstatus?builder=${BUILDERNAME}&number=${BUILDNUM}`
    This displays a waterfall-like chronologically-oriented view of all the
    steps for a given build number on a given builder.

:samp:`/builders/${BUILDERNAME}`
    This describes the given :class:`Builder` and provides buttons to force a
    build.  A ``numbuilds=`` argument will control how many build lines
    are displayed (5 by default).

:samp:`/builders/${BUILDERNAME}/builds/${BUILDNUM}`
    This describes a specific Build.

:samp:`/builders/${BUILDERNAME}/builds/${BUILDNUM}/steps/${STEPNAME}`
    This describes a specific BuildStep.

:samp:`/builders/${BUILDERNAME}/builds/${BUILDNUM}/steps/${STEPNAME}/logs/${LOGNAME}`
    This provides an HTML representation of a specific logfile.

:samp:`/builders/${BUILDERNAME}/builds/${BUILDNUM}/steps/${STEPNAME}/logs/${LOGNAME}/text`
    This returns the logfile as plain text, without any HTML coloring
    markup. It also removes the `headers`, which are the lines that
    describe what command was run and what the environment variable
    settings were like. This maybe be useful for saving to disk and
    feeding to tools like :command:`grep`.

``/changes``
    This provides a brief description of the :class:`ChangeSource` in use
    (:ref:`Change-Sources`).

:samp:`/changes/{NN}`
    This shows detailed information about the numbered :class:`Change`: who was the
    author, what files were changed, what revision number was represented,
    etc.

``/buildslaves``
    This summarizes each :class:`BuildSlave`, including which `Builder`\s are
    configured to use it, whether the buildslave is currently connected or
    not, and host information retrieved from the buildslave itself.

    A ``no_builders=1`` URL argument will omit the builders column.  This is
    useful if each buildslave is assigned to a large number of builders.

``/one_line_per_build``
    This page shows one line of text for each build, merging information
    from all :class:`Builder`\s [#]_. Each line specifies
    the name of the Builder, the number of the :class:`Build`, what revision it
    used, and a summary of the results. Successful builds are in green,
    while failing builds are in red. The date and time of the build are
    added to the right-hand edge of the line. The lines are ordered by
    build finish timestamp.
    
    One or more ``builder=`` or ``branch=`` arguments can be used to
    restrict the list. In addition, a ``numbuilds=`` argument will
    control how many lines are displayed (20 by default).

``/builders``
    This page shows a small table, with one box for each :class:`Builder`,
    containing the results of the most recent :class:`Build`. It does not show the
    individual steps, or the current status. This is a simple summary of
    buildbot status: if this page is green, then all tests are passing.
    
    As with ``/one_line_per_build``, this page will also honor
    ``builder=`` and ``branch=`` arguments.

``/about``
    This page gives a brief summary of the Buildbot itself: software
    version, versions of some libraries that the Buildbot depends upon,
    etc. It also contains a link to the buildbot.net home page.

There are also a set of web-status resources that are intended for use
by other programs, rather than humans.

``/change_hook``
    This provides an endpoint for web-based source change
    notification. It is used by GitHub and
    contrib/post_build_request.py. See :ref:`Change-Hooks` for more
    details.

.. _WebStatus-Configuration-Parameters:

WebStatus Configuration Parameters
++++++++++++++++++++++++++++++++++

.. _HTTP-Connection:

HTTP Connection
###############

The most common way to run a :class:`WebStatus` is on a regular TCP
port. To do this, just pass in the TCP port number when you create the
:class:`WebStatus` instance; this is called the ``http_port`` argument::

    from buildbot.status.html import WebStatus
    c['status'].append(WebStatus(http_port=8080))

The ``http_port`` argument is actually a `strports specification` for the
port that the web server should listen on. This can be a simple port number, or
a string like ``http_port="tcp:8080:interface=127.0.0.1"`` (to limit
connections to the loopback interface, and therefore to clients running on the
same host) [#]_.

If instead (or in addition) you provide the ``distrib_port``
argument, a twisted.web distributed server will be started either on a
TCP port (if ``distrib_port`` is like ``"tcp:12345"``) or more
likely on a UNIX socket (if ``distrib_port`` is like
``"unix:/path/to/socket"``).

The ``public_html`` option gives the path to a regular directory of HTML
files that will be displayed alongside the various built-in URLs buildbot
supplies.  This is most often used to supply CSS files (:file:`/buildbot.css`)
and a top-level navigational file (:file:`/index.html`), but can also serve any
other files required - even build results!

.. _Authorization:

Authorization
#############

The buildbot web status is, by default, read-only.  It displays lots of
information, but users are not allowed to affect the operation of the
buildmaster.  However, there are a number of supported activities that can
be enabled, and Buildbot can also perform rudimentary username/password
authentication.  The actions are:

``forceBuild``
    force a particular builder to begin building, optionally with a specific revision, branch, etc.

``forceAllBuilds``
    force *all* builders to start building

``pingBuilder``
    "ping" a builder's buildslaves to check that they are alive

``gracefulShutdown``
    gracefully shut down a slave when it is finished with its current build

``stopBuild``
    stop a running build

``stopAllBuilds``
    stop all running builds

``cancelPendingBuild``
    cancel a build that has not yet started

``stopChange``
    cancel builds that include a given change number

``cleanShutdown``
    shut down the master gracefully, without interrupting builds

For each of these actions, you can configure buildbot to never allow the
action, always allow the action, allow the action to any authenticated user, or
check with a function of your creation to determine whether the action is OK.

This is all configured with the :class:`Authz` class::

    from buildbot.status.html import WebStatus
    from buildbot.status.web.authz import Authz
    authz = Authz(
        forceBuild=True,
        stopBuild=True)
    c['status'].append(WebStatus(http_port=8080, authz=authz))

Each of the actions listed above is an option to :class:`Authz`.  You can specify
``False`` (the default) to prohibit that action or ``True`` to enable it.

.. _Authentication:

Authentication
##############

If you do not wish to allow strangers to perform actions, but do want
developers to have such access, you will need to add some authentication
support.  Pass an instance of :class:`status.web.auth.IAuth` as a ``auth``
keyword argument to :class:`Authz`, and specify the action as ``"auth"``. ::

    from buildbot.status.html import WebStatus
    from buildbot.status.web.authz import Authz
    from buildbot.status.web.auth import BasicAuth
    users = [('bob', 'secret-pass'), ('jill', 'super-pass')]
    authz = Authz(auth=BasicAuth(users),
        forceBuild='auth', # only authenticated users
        pingBuilder=True, # but anyone can do this
    )
    c['status'].append(WebStatus(http_port=8080, authz=authz))
    # or
    from buildbot.status.web.auth import HTPasswdAuth
    auth = (HTPasswdAuth('/path/to/htpasswd'))

The class :class:`BasicAuth` implements a basic authentication mechanism using a
list of user/password tuples provided from the configuration file.  The class
`HTPasswdAuth` implements an authentication against an :file:`.htpasswd`
file.

If you need still-more flexibility, pass a function for the authentication
action.  That function will be called with an authenticated username and some
action-specific arguments, and should return true if the action is authorized. ::

    def canForceBuild(username, builder_status):
        if builder_status.getName() == 'smoketest':
            return True # any authenticated user can run smoketest
        elif username == 'releng':
            return True # releng can force whatever they want
        else:
            return False # otherwise, no way.
    
    authz = Authz(auth=BasicAuth(users),
        forceBuild=canForceBuild)

The ``forceBuild`` and ``pingBuilder`` actions both supply a
:class:`BuilderStatus` object.  The ``stopBuild`` action supplies a :class:`BuildStatus`
object.  The ``cancelPendingBuild`` action supplies a :class:`BuildRequest`.  The
remainder do not supply any extra arguments.

.. _Logging-configuration:

Logging configuration
#####################

The `WebStatus` uses a separate log file (:file:`http.log`) to avoid clutter
buildbot's default log (:file:`twistd.log`) with request/response messages.
This log is also, by default, rotated in the same way as the twistd.log
file, but you can also customize the rotation logic with the following
parameters if you need a different behaviour.

``rotateLength``
    An integer defining the file size at which log files are rotated. 

``maxRotatedFiles``
    The maximum number of old log files to keep. 

.. _URL-decorating-options:
    
URL-decorating options
######################

These arguments adds an URL link to various places in the WebStatus,
such as revisions, repositories, projects and, optionally, ticket/bug references
in change comments.

revlink
'''''''

The ``revlink`` is used to create links from revision IDs in the web
status to a web-view of your source control system. The parameter's
value must be a format string, a dict mapping a string (repository
name) to format strings, or a callable.

The format string should use ``%s`` to insert the revision id in the url.  For
example, for Buildbot on GitHub::

    revlink='http://github.com/buildbot/buildbot/tree/%s'

The revision ID will be URL encoded before inserted in the replacement string

The callable takes the revision id and repository argument, and should return
an URL to the revision.  Note that the revision id may not always be in the
form you expect, so code defensively.  In particular, a revision of "??" may be
supplied when no other information is available.

Note that :class:`SourceStamp`\s that are not created from version-control changes (e.g.,
those created by a Nightly or Periodic scheduler) will have an empty repository
string, as the respository is not known.

changecommentlink
'''''''''''''''''

The ``changecommentlink`` argument can be used to create links to
ticket-ids from change comments (i.e. #123).

The argument can either be a tuple of three strings, a dictionary
mapping strings (project names) to tuples or a callable taking a
changetext (a :class:`jinja2.Markup` instance) and a project name,
returning a the same change text with additional links/html tags added
to it.

If the tuple is used, it should contain three strings where the first
element is a regex that searches for strings (with match groups), the
second is a replace-string that, when substituted with ``\1`` etc,
yields the URL and the third is the title attribute of the link. (The
``<a href="" title=""></a>`` is added by the system.) So, for Trac
tickets (#42, etc): ``changecommentlink(r"#(\d+)",
r"http://buildbot.net/trac/ticket/\1", r"Ticket \g<0>")`` . 

projects
''''''''

A dictionary from strings to strings, mapping project names to URLs,
or a callable taking a project name and returning an URL.

repositories
''''''''''''

Same as the projects arg above, a dict or callable mapping project names
to URLs.

.. _Display-Specific-Options:

Display-Specific Options
########################

The ``order_console_by_time`` option affects the rendering of the console;
see the description of the console above.

The ``numbuilds`` option determines the number of builds that most status
displays will show.  It can usually be overriden in the URL, e.g.,
``?numbuilds=13``.

The ``num_events`` option gives the default number of events that the
waterfall will display.  The ``num_events_max`` gives the maximum number of
events displayed, even if the web browser requests more.

.. _Change-Hooks:

Change Hooks
++++++++++++

The ``/change_hook`` url is a magic URL which will accept HTTP requests and translate
them into changes for buildbot. Implementations (such as a trivial json-based endpoint
and a GitHub implementation) can be found in :file:`master/buildbot/status/web/hooks`.
The format of the url is :samp:`/change_hook/{DIALECT}` where DIALECT is a package within the 
hooks directory. Change_hook is disabled by default and each DIALECT has to be enabled
separately, for security reasons

An example WebStatus configuration line which enables change_hook and two DIALECTS::

    c['status'].append(html.WebStatus(http_port=8010,allowForce=True,
        change_hook_dialects={
                              'base': True,
                              'somehook': {'option1':True,
                                           'option2':False}}))

Within the WebStatus arguments, the ``change_hook`` key enables/disables the module
and ``change_hook_dialects`` whitelists DIALECTs where the keys are the module names
and the values are optional arguments which will be passed to the hooks.

The :file:`post_build_request.py` script in :file:`master/contrib` allows for the
submission of an arbitrary change request. Run :command:`post_build_request.py
--help` for more information.  The ``base`` dialect must be enabled for this to
work.

github hook
###########

The GitHub hook is simple and takes no options. ::

    c['status'].append(html.WebStatus(..
                       change_hook_dialects={ 'github' : True }))

With this set up, add a Post-Receive URL for the project in the GitHub
administrative interface, pointing to ``/change_hook/github`` relative to
the root of the web status.  For example, if the grid URL is
``http://builds.mycompany.com/bbot/grid``, then point GitHub to
``http://builds.mycompany.com/bbot/change_hook/github``. To specify a project
associated to the repository, append ``?project=name`` to the URL.

Note that there is a standalone HTTP server available for receiving GitHub
notifications, as well: :file:`contrib/github_buildbot.py`.  This script may be
useful in cases where you cannot expose the WebStatus for public consumption.

.. index:: email, mail

.. _MailNotifier:

MailNotifier
~~~~~~~~~~~~

.. py:class:: buildbot.status.mail.MailNotifier

The buildbot can also send email when builds finish. The most common
use of this is to tell developers when their change has caused the
build to fail. It is also quite common to send a message to a mailing
list (usually named `builds` or similar) about every build.

The :class:`MailNotifier` status target is used to accomplish this. You
configure it by specifying who mail should be sent to, under what
circumstances mail should be sent, and how to deliver the mail. It can
be configured to only send out mail for certain builders, and only
send messages when the build fails, or when the builder transitions
from success to failure. It can also be configured to include various
build logs in each message.


By default, the message will be sent to the Interested Users list
(:ref:`Doing-Things-With-Users`), which includes all developers who made changes in the
build. You can add additional recipients with the extraRecipients argument.
You can also add interested users by setting the  ``owners`` build property
to a list of users in the scheduler constructor (:ref:`Configuring-Schedulers`).

Each :class:`MailNotifier` sends mail to a single set of recipients. To send
different kinds of mail to different recipients, use multiple
:class:`MailNotifier`\s.

The following simple example will send an email upon the completion of
each build, to just those developers whose :class:`Change`\s were included in
the build. The email contains a description of the :class:`Build`, its results,
and URLs where more information can be obtained. ::

    from buildbot.status.mail import MailNotifier
    mn = MailNotifier(fromaddr="buildbot@example.org", lookup="example.org")
    c['status'].append(mn)

To get a simple one-message-per-build (say, for a mailing list), use
the following form instead. This form does not send mail to individual
developers (and thus does not need the ``lookup=`` argument,
explained below), instead it only ever sends mail to the `extra
recipients` named in the arguments::

    mn = MailNotifier(fromaddr="buildbot@example.org",
                      sendToInterestedUsers=False,
                      extraRecipients=['listaddr@example.org'])

If your SMTP host requires authentication before it allows you to send emails,
this can also be done by specifying ``smtpUser`` and ``smptPassword``::

    mn = MailNotifier(fromaddr="myuser@gmail.com",
                      sendToInterestedUsers=False,
                      extraRecipients=["listaddr@example.org"],
                      relayhost="smtp.gmail.com", smtpPort=587,
                      smtpUser="myuser@gmail.com", smtpPassword="mypassword")

If you want to require Transport Layer Security (TLS), then you can also
set ``useTls``::

    mn = MailNotifier(fromaddr="myuser@gmail.com",
                      sendToInterestedUsers=False,
                      extraRecipients=["listaddr@example.org"],
                      useTls=True, relayhost="smtp.gmail.com", smtpPort=587,
                      smtpUser="myuser@gmail.com", smtpPassword="mypassword")

.. note:: If you see ``twisted.mail.smtp.TLSRequiredError`` exceptions in
   the log while using TLS, this can be due *either* to the server not
   supporting TLS or to a missing pyopenssl package on the buildmaster system.

In some cases it is desirable to have different information then what is
provided in a standard MailNotifier message. For this purpose MailNotifier
provides the argument ``messageFormatter`` (a function) which allows for the
creation of messages with unique content.

For example, if only short emails are desired (e.g., for delivery to phones) ::

    from buildbot.status.builder import Results
    def messageFormatter(mode, name, build, results, master_status):
        result = Results[results]
    
        text = list()
        text.append("STATUS: %s" % result.title())
        return {
            'body' : "\n".join(text),
            'type' : 'plain'
        }
    
    mn = MailNotifier(fromaddr="buildbot@example.org",
                      sendToInterestedUsers=False,
                      mode='problem',
                      extraRecipients=['listaddr@example.org'],
                      messageFormatter=messageFormatter)

Another example of a function delivering a customized html email
containing the last 80 log lines of logs of the last build step is
given below::

    from buildbot.status.builder import Results
    
    def html_message_formatter(mode, name, build, results, master_status):
        """Provide a customized message to Buildbot's MailNotifier.
        
        The last 80 lines of the log are provided as well as the changes
        relevant to the build.  Message content is formatted as html.
        """
        result = Results[results]
        
        limit_lines = 80
        text = list()
        text.append(u'<h4>Build status: %s</h4>' % result.upper())
        text.append(u'<table cellspacing="10"><tr>')
        text.append(u"<td>Buildslave for this Build:</td><td><b>%s</b></td></tr>" % build.getSlavename())
        if master_status.getURLForThing(build):
            text.append(u'<tr><td>Complete logs for all build steps:</td><td><a href="%s">%s</a></td></tr>'
                        % (master_status.getURLForThing(build),
                           master_status.getURLForThing(build))
                        )
            text.append(u'<tr><td>Build Reason:</td><td>%s</td></tr>' % build.getReason())
            source = u""
            ss = build.getSourceStamp()
            if ss.branch:
                source += u"[branch %s] " % ss.branch
            if ss.revision:
                source +=  ss.revision
            else:
                source += u"HEAD"
            if ss.patch:
                source += u" (plus patch)"
            if ss.patch_info: # add patch comment
                source += u" (%s)" % ss.patch_info[1]
                source +=
            text.append(u"<tr><td>Build Source Stamp:</td><td><b>%s</b></td></tr>" % source)
            text.append(u"<tr><td>Blamelist:</td><td>%s</td></tr>" % ",".join(build.getResponsibleUsers()))
            text.append(u'</table>')
            if ss.changes:
                text.append(u'<h4>Recent Changes:</h4>')
                for c in ss.changes:
                    cd = c.asDict()
                    when = datetime.datetime.fromtimestamp(cd['when'] ).ctime()
                    text.append(u'<table cellspacing="10">')
                    text.append(u'<tr><td>Repository:</td><td>%s</td></tr>' % cd['repository'] )
                    text.append(u'<tr><td>Project:</td><td>%s</td></tr>' % cd['project'] )
                    text.append(u'<tr><td>Time:</td><td>%s</td></tr>' % when)
                    text.append(u'<tr><td>Changed by:</td><td>%s</td></tr>' % cd['who'] )
                    text.append(u'<tr><td>Comments:</td><td>%s</td></tr>' % cd['comments'] )
                    text.append(u'</table>')
                    files = cd['files']
                    if files:
                        text.append(u'<table cellspacing="10"><tr><th align="left">Files</th><th>URL</th></tr>')
                        for file in files:
                            text.append(u'<tr><td>%s:</td><td>%s</td></tr>' % (file['name'], file['url']))
                        text.append(u'</table>')
            text.append(u'<br>')
            # get log for last step 
            logs = build.getLogs()
            # logs within a step are in reverse order. Search back until we find stdio
            for log in reversed(logs):
                if log.getName() == 'stdio':
                    break
            name = "%s.%s" % (log.getStep().getName(), log.getName())
            status, dummy = log.getStep().getResults()
            content = log.getText().splitlines() # Note: can be VERY LARGE
            url = u'%s/steps/%s/logs/%s' % (master_status.getURLForThing(build),
                                           log.getStep().getName(),
                                           log.getName())
            
            text.append(u'<i>Detailed log of last build step:</i> <a href="%s">%s</a>'
                        % (url, url))
            text.append(u'<br>')
            text.append(u'<h4>Last %d lines of "%s"</h4>' % (limit_lines, name))
            unilist = list()
            for line in content[len(content)-limit_lines:]:
                unilist.append(cgi.escape(unicode(line,'utf-8')))
            text.append(u'<pre>'.join([uniline for uniline in unilist]))
            text.append(u'</pre>')
            text.append(u'<br><br>')
            text.append(u'<b>-The Buildbot</b>')
            return {
                'body': u"\n".join(text),
                'type': 'html'
                }
    
    mn = MailNotifier(fromaddr="buildbot@example.org",
                      sendToInterestedUsers=False,
                      mode='failing',
                      extraRecipients=['listaddr@example.org'],
                      messageFormatter=message_formatter)

.. _MailNotifier-arguments:

MailNotifier arguments
++++++++++++++++++++++

``fromaddr``
    The email address to be used in the 'From' header.

``sendToInterestedUsers``
    (boolean). If ``True`` (the default), send mail to all of the Interested
    Users. If ``False``, only send mail to the ``extraRecipients`` list.

``extraRecipients``
    (list of strings). A list of email addresses to which messages should
    be sent (in addition to the InterestedUsers list, which includes any
    developers who made :class:`Change`\s that went into this build). It is a good
    idea to create a small mailing list and deliver to that, then let
    subscribers come and go as they please.

``subject``
    (string). A string to be used as the subject line of the message.
    ``%(builder)s`` will be replaced with the name of the builder which
    provoked the message.

``mode``
    (string). Default to ``all``. One of:

    ``all``
        Send mail about all builds, both passing and failing
        
    ``change``
        Only send mail about builds which change status
    
    ``failing``
        Only send mail about builds which fail

    ``warning``
        Only send mail about builds which fail or generate warnings

    ``passing``
        Only send mail about builds which succeed
        
    ``problem``
        Only send mail about a build which failed when the previous build has passed.
        If your builds usually pass, then this will only send mail when a problem
        occurs.

``builders``
    (list of strings). A list of builder names for which mail should be
    sent. Defaults to ``None`` (send mail for all builds). Use either builders
    or categories, but not both.

``categories``
    (list of strings). A list of category names to serve status
    information for. Defaults to ``None`` (all categories). Use either
    builders or categories, but not both.

``addLogs``
    (boolean). If ``True``, include all build logs as attachments to the
    messages. These can be quite large. This can also be set to a list of
    log names, to send a subset of the logs. Defaults to ``False``.

``addPatch``
    (boolean). If ``True``, include the patch content if a patch was present.
    Patches are usually used on a :class:`Try` server.
    Defaults to ``True``.

``buildSetSummary``
    (boolean). If ``True``, send a single summary email consisting of the
    concatenation of all build completion messages rather than a
    completion message for each build.  Defaults to ``False``.

``relayhost``
    (string). The host to which the outbound SMTP connection should be
    made. Defaults to 'localhost'

``smtpPort``
    (int). The port that will be used on outbound SMTP
    connections. Defaults to 25.

``useTls``
    (boolean). When this argument is ``True`` (default is ``False``)
    ``MailNotifier`` sends emails using TLS and authenticates with the
    ``relayhost``. When using TLS the arguments ``smtpUser`` and
    ``smtpPassword`` must also be specified.

``smtpUser``
    (string). The user name to use when authenticating with the
    ``relayhost``. 

``smtpPassword``
    (string). The password that will be used when authenticating with the
    ``relayhost``.

``lookup``
    (implementor of :class:`IEmailLookup`). Object which provides
    :class:`IEmailLookup`, which is responsible for mapping User names (which come
    from the VC system) into valid email addresses. If not provided, the
    notifier will only be able to send mail to the addresses in the
    extraRecipients list. Most of the time you can use a simple Domain
    instance. As a shortcut, you can pass as string: this will be treated
    as if you had provided ``Domain(str)``. For example,
    ``lookup='twistedmatrix.com'`` will allow mail to be sent to all
    developers whose SVN usernames match their twistedmatrix.com account
    names. See :file:`buildbot/status/mail.py` for more details.

``messageFormatter``
    This is a optional function that can be used to generate a custom mail message.
    A :func:`messageFormatter` function takes the mail mode (``mode``), builder
    name (``name``), the build status (``build``), the result code
    (``results``), and the BuildMaster status (``master_status``).  It
    returns a dictionary. The ``body`` key gives a string that is the complete
    text of the message. The ``type`` key is the message type ('plain' or
    'html'). The 'html' type should be used when generating an HTML message.  The
    ``subject`` key is optional, but gives the subject for the email.

``extraHeaders``
    (dictionary) A dictionary containing key/value pairs of extra headers to add
    to sent e-mails. Both the keys and the values may be a `WithProperties` instance.

As a help to those writing :func:`messageFormatter` functions, the following
table describes how to get some useful pieces of information from the various
status objects:

Name of the builder that generated this event
    ``name``

Name of the project
    :meth:`master_status.getProjectName()`

MailNotifier mode
    ``mode`` (one of ``all``, ``failing``, ``problem``, ``change``, ``passing``)

Builder result as a string ::
    
    from buildbot.status.builder import Results
    result_str = Results[results]
    # one of 'success', 'warnings', 'failure', 'skipped', or 'exception'

URL to build page
    ``master_status.getURLForThing(build)``

URL to buildbot main page.
    ``master_status.getBuildbotURL()``

Build text
    ``build.getText()``

Mapping of property names to values
    ``build.getProperties()`` (a :class:`Properties` instance)

Slave name
    ``build.getSlavename()``

Build reason (from a forced build)
    ``build.getReason()``

List of responsible users
    ``build.getResponsibleUsers()``

Source information (only valid if ss is not ``None``)

    ::
        
        ss = build.getSourceStamp()
        if ss:
            branch = ss.branch
            revision = ss.revision
            patch = ss.patch
            changes = ss.changes # list

    A change object has the following useful information:

    ``who``
        (str) who made this change
        
    ``revision``
        (str) what VC revision is this change
        
    ``branch``
        (str) on what branch did this change occur
        
    ``when``
        (str) when did this change occur
        
    ``files``
        (list of str) what files were affected in this change
        
    ``comments``
        (str) comments reguarding the change.

    The ``Change`` methods :meth:`asText` and :meth:`asDict` can be used to format the
    information above.  :meth:`asText` returns a list of strings and :meth:`asDict` returns
    a dictonary suitable for html/mail rendering.
    
Log information ::
    
    logs = list()
    for log in build.getLogs():
        log_name = "%s.%s" % (log.getStep().getName(), log.getName())
        log_status, dummy = log.getStep().getResults()
        log_body = log.getText().splitlines() # Note: can be VERY LARGE
        log_url = '%s/steps/%s/logs/%s' % (master_status.getURLForThing(build),
                                           log.getStep().getName(),
                                           log.getName())
        logs.append((log_name, log_url, log_body, log_status))

.. index:: IRC

.. _IRC-Bot:

IRC Bot
~~~~~~~

.. py:class:: buildbot.status.words.IRC


The :class:`buildbot.status.words.IRC` status target creates an IRC bot
which will attach to certain channels and be available for status
queries. It can also be asked to announce builds as they occur, or be
told to shut up. ::

    from buildbot.status import words
    irc = words.IRC("irc.example.org", "botnickname",
                    channels=[{"channel": "#example1"},
                              {"channel": "#example2",
                               "password": "somesecretpassword"}],
                    password="mysecretnickservpassword",
                    notify_events={
                      'exception': 1,
                      'successToFailure': 1,
                      'failureToSuccess': 1,
                    })
    c['status'].append(irc)

Take a look at the docstring for :class:`words.IRC` for more details on
configuring this service. Note that the ``useSSL`` option requires
`PyOpenSSL`_.  The ``password`` argument, if provided, will be sent to
Nickserv to claim the nickname: some IRC servers will not allow clients to send
private messages until they have logged in with a password. We can also specify
a different ``port`` number. Defalut values is 6667.

To use the service, you address messages at the buildbot, either
normally (``botnickname: status``) or with private messages
(``/msg botnickname status``). The buildbot will respond in kind.

If you issue a command that is currently not available, the buildbot
will respond with an error message. If the ``noticeOnChannel=True``
option was used, error messages will be sent as channel notices instead
of messaging. The default value is ``noticeOnChannel=False``.

Some of the commands currently available:

``list builders``
    Emit a list of all configured builders
    
:samp:`status {BUILDER}`
    Announce the status of a specific Builder: what it is doing right now.
    
``status all``
    Announce the status of all Builders
    
:samp:`watch {BUILDER}`
    If the given :class:`Builder` is currently running, wait until the :class:`Build` is
    finished and then announce the results.
    
:samp:`last {BUILDER}`
    Return the results of the last build to run on the given :class:`Builder`.
    
:samp:`join {CHANNEL}`
    Join the given IRC channel
    
:samp:`leave {CHANNEL}`
    Leave the given IRC channel
    
:samp:`notify on|off|list {EVENT}`
    Report events relating to builds.  If the command is issued as a
    private message, then the report will be sent back as a private
    message to the user who issued the command.  Otherwise, the report
    will be sent to the channel.  Available events to be notified are:

    ``started``
        A build has started
        
    ``finished``
        A build has finished
        
    ``success``
        A build finished successfully
        
    ``failure``
        A build failed
        
    ``exception``
        A build generated and exception
        
    ``xToY``
        The previous build was x, but this one is Y, where x and Y are each
        one of success, warnings, failure, exception (except Y is
        capitalized).  For example: ``successToFailure`` will notify if the
        previous build was successful, but this one failed

:samp:`help {COMMAND}`
    Describe a command. Use :command:`help commands` to get a list of known
    commands.
    
``source``
    Announce the URL of the Buildbot's home page.
    
``version``
    Announce the version of this Buildbot.

Additionally, the config file may specify default notification options
as shown in the example earlier.

If the ``allowForce=True`` option was used, some addtional commands
will be available:

:samp:`force build [--branch={BRANCH}] [--revision={REVISION}] {BUILDER} {REASON}`
    Tell the given :class:`Builder` to start a build of the latest code. The user
    requesting the build and *REASON* are recorded in the :class:`Build` status. The
    buildbot will announce the build's status when it finishes.The
    user can specify a branch and/or revision with the optional
    parameters :samp:`--branch={BRANCH}` and :samp:`--revision={REVISION}`.


:samp:`stop build {BUILDER} {REASON}`
    Terminate any running build in the given :class:`Builder`. *REASON* will be added
    to the build status to explain why it was stopped. You might use this
    if you committed a bug, corrected it right away, and don't want to
    wait for the first build (which is destined to fail) to complete
    before starting the second (hopefully fixed) build.

If the `categories` is set to a category of builders (see categories
option in :ref:`Builder-Configuration`) changes related to only that 
category of builders will be sent to the channel.

.. _PBListener:
    
PBListener
~~~~~~~~~~

.. @cindex PBListener
.. py:class:: buildbot.status.client.PBListener

::

    import buildbot.status.client
    pbl = buildbot.status.client.PBListener(port=int, user=str,
                                            passwd=str)
    c['status'].append(pbl)

This sets up a PB listener on the given TCP port, to which a PB-based
status client can connect and retrieve status information.
:command:`buildbot statusgui` (:ref:`statusgui`) is an example of such a
status client. The ``port`` argument can also be a strports
specification string.

.. _StatusPush:

StatusPush
~~~~~~~~~~

.. @cindex StatusPush
.. py:class:: buildbot.status.status_push.StatusPush

::

    def Process(self):
      print str(self.queue.popChunk())
      self.queueNextServerPush()
    
    import buildbot.status.status_push
    sp = buildbot.status.status_push.StatusPush(serverPushCb=Process,
                                                bufferDelay=0.5,
                                                retryDelay=5)
    c['status'].append(sp)

:class:`StatusPush` batches events normally processed and sends it to the
:func:`serverPushCb` callback every ``bufferDelay`` seconds. The callback
should pop items from the queue and then queue the next callback.
If no items were poped from ``self.queue``, ``retryDelay`` seconds will be
waited instead.

.. _HttpStatusPush:

HttpStatusPush
~~~~~~~~~~~~~~

.. @cindex HttpStatusPush
.. @stindex buildbot.status.status_push.HttpStatusPush

::

    import buildbot.status.status_push
    sp = buildbot.status.status_push.HttpStatusPush(
            serverUrl="http://example.com/submit")
    c['status'].append(sp)

:class:`HttpStatusPush` builds on :class:`StatusPush` and sends HTTP requests to
``serverUrl``, with all the items json-encoded. It is useful to create a
status front end outside of buildbot for better scalability.

.. _GerritStatusPush:

GerritStatusPush
~~~~~~~~~~~~~~~~

.. py:class:: buildbot.status.status_gerrit.GerritStatusPush

::

    from buildbot.status.status_gerrit import GerritStatusPush

    def gerritReviewCB(builderName, build, result, arg):
        message =  "Buildbot finished compiling your patchset\n"
        message += "on configuration: %s\n" % builderName
        message += "The result is: %s\n" % Results[result].upper()

        if arg:
            message += "\nFor more details visit:\n"
            message += "%sbuilders/%s/builds/%d\n" % (arg, builderName, build.getNumber())

        # message, verified, reviewed
        return message, (result == 0 or -1), 0

    c['buildbotURL'] = 'http://buildbot.example.com/'
    c['status'].append(GerritStatusPush('127.0.0.1', 'buildbot',
                                        reviewCB=gerritReviewCB,
                                        reviewArg=c['buildbotURL']))

GerritStatusPush sends review of the :class:`Change` back to the Gerrit server.

.. _Writing-New-Status-Plugins:

Writing New Status Plugins
~~~~~~~~~~~~~~~~~~~~~~~~~~

.. TODO: this needs a lot more examples

Each status plugin is an object which provides the
:class:`twisted.application.service.IService` interface, which creates a
tree of Services with the buildmaster at the top [not strictly true].
The status plugins are all children of an object which implements
:class:`buildbot.interfaces.IStatus`, the main status object. From this
object, the plugin can retrieve anything it wants about current and
past builds. It can also subscribe to hear about new and upcoming
builds.

Status plugins which only react to human queries (like the Waterfall
display) never need to subscribe to anything: they are idle until
someone asks a question, then wake up and extract the information they
need to answer it, then they go back to sleep. Plugins which need to
act spontaneously when builds complete (like the :class:`MailNotifier` plugin)
need to subscribe to hear about new builds.

If the status plugin needs to run network services (like the HTTP
server used by the Waterfall plugin), they can be attached as Service
children of the plugin itself, using the :class:`IServiceCollection`
interface.

.. [#] Apparently this is the same way http://buildd.debian.org displays build status

.. [#] It may even be possible to provide SSL access by using a
    specification like ``"ssl:12345:privateKey=mykey.pen:certKey=cert.pem"``,
    but this is completely untested
    
.. _PyOpenSSL: http://pyopenssl.sourceforge.net/
