

# Introduction #

One of Erlang's strengths is hot code loading, the ability to change the software on the system without interrupting service.  For internet applications this is especially powerful.

Hot code loading in Erlang consists of low-level language support, and a high-level strategy of [release handling](http://www.erlang.org/doc/design_principles/release_structure.html) embodied in the OTP libraries.

erlrc embodies another high level strategy for using the low-level capabilities of Erlang for hot code loading.  Unlike release handling it is designed to be incremental and integrated into the OS package management.  The goal is an experience like this:
```
pmineiro@ub32srvvmw-199% sudo apt-get install egerl
Reading package lists... Done
Building dependency tree       
Reading state information... Done
The following packages will be upgraded:
  egerl
1 upgraded, 0 newly installed, 0 to remove and 34 not upgraded.
Need to get 0B/113kB of archives.
After unpacking 0B of additional disk space will be used.
WARNING: The following packages cannot be authenticated!  egerl
Install these packages without verification [y/N]? y
(Reading database ... 48221 files and directories currently installed.)
Preparing to replace egerl 4.0.1 (using .../archives/egerl_4.1.0_all.deb) ...
Unpacking replacement egerl ...
erlrc-upgrade: Upgrading 'egerl': (cb8eec1a1b85ec017517d3e51c5aee7b) upgraded
Setting up egerl (4.1.0) ...
erlrc-start: Starting 'egerl': (cb8eec1a1b85ec017517d3e51c5aee7b) already_running
```
Basically, on the running Erlang node on my machine, the Erlang application egerl was hot-code upgraded from 4.0.1 to 4.1.0 when the package was installed.  If this were an initial install, it would have started the application instead.  The OS package manager takes care of dependencies and you end up managing Erlang the way you manage everything else.[[1](#1.md)]

Perhaps you would prefer another flavor?
```
% sudo yum -q -y install lyet
erlrc-start: Starting 'lyet': (erlang) started
% sudo yum -q -y remove lyet
erlrc-stop: Stopping 'lyet': (erlang) unloaded
```
erlrc uses shell scripts as its interface so integration with any package manager should be feasible.

# Quick Start #

If you want your packages to start leveraging erlrc, then:

  1. Install erlrc into your system.
  1. Ensure that $ERLRC\_ROOT/nodes contains a file with your node name and cookie in it.  $ERLRC\_ROOT defaults to /etc/erlrc.d .
  1. Start your erlang nodes with the command line flag `-s erlrc_boot boot`.

Now your Erlang runtime is ready.  For your packages either:
  1. Use [fw-template-erlang](http://code.google.com/p/fwtemplates/wiki/FwTemplateErlangWalkthrough)
    * works with rpm and debian
  1. For debian:
    1. Add an empty file named the same as your application installed into $ERLRC\_ROOT/applications .
    1. Add postrm, postinst, and prerm hooks to your debian packages.
      * pre-remove should invoke erlrc-stop if the package is being removed.
      * post-remove should invoke erlrc-upgrade if the package is being upgraded.
      * post-install should invoke erlrc-start if the package is being configured.
  1. For rpm:
    1. Add an empty file named the same as your application installed into $ERLRC\_ROOT/applications .
    1. Add %preun, and %posttrans hooks to your rpm packages.
      * %preun should invoke erlrc-stop if the installation count indicates removal.
      * %preun should invoke erlrc-upgrade if the installation count indicates upgrade.
      * %posttrans should invoke erlrc-start.
      * you'll need to figure out the currently installed version number to invoke the shell scripts.  i grab the currently installed version with an `rpm -q` in a %pretrans hook and write it to TMPDIR.
  1. For others
    1. You'll have to adapt the examples.  If you come up with something, let me know and I'll add it here.

# Design #

Erlrc consists of two pieces, an upgrade piece, and a boot piece.

## Boot ##

Erlang uses a [boot script](http://www.erlang.org/doc/design_principles/release_structure.html#boot) to decide what to do when starting up.  The situation is analogous to http servers such as Apache which, when first designed, used a single file for configuration.  This is a natural approach which unfortunately complicates package managed operating systems, since different packages have to manipulate the same file.  As with Apache, our solution involves controlling the boot process from a directory.

$ERLRC\_ROOT/applications contains a list of applications that should be run at boot time.  [erlrc\_boot](http://code.google.com/p/erlrc/source/browse/trunk/src/erlrc_boot.erl) sorts these applications in dependency order and then starts them.[[2](#2.md)]
```
% ls /etc/erlrc.d/applications
appinspect       egerl   fragmentron  genherd       nodefinder    zfile
combonodefinder  erlrc   fuserl       loggins       schemafinder
ec2nodefinder    erlsom  gencron      n54etsbugfix  virtuerl
```
By default $ERLRC\_ROOT is /etc/erlrc.d .

You can of course start stuff in your boot script as well if you like, erlrc won't start something that's already started.  In practice we use the vanilla boot scripts and pass `-s erlrc_boot boot` to the erl command line.

Oh, everything works with [included applications](http://www.erlang.org/doc/design_principles/included_applications.html) as well.  By "everything works", we mean that erlrc will look at the set of applications to start, will note any application `Y` that includes another application `X`, and will consider the requirement to start `X` as being satisfied by starting `Y`, and will only start `Y`.

In practice, then, any package which provides an OTP application should place a file in $ERLRC\_ROOT/applications/ to ensure that it gets started up at (Erlang) boot time.  The filename indicates the name of the application to start, and the contents of this file are not examined by erlrc.  If you use [framewerk](http://code.google.com/p/fwtemplates) this is done for you automatically by [fw-template-erlang](http://code.google.com/p/fwtemplates/wiki/FwTemplateErlangWalkthrough).

## Package Installation ##

For package installation time, shell scripts are provided which are designed to be run from the packaging system's hooks.  These shell scripts ultimately invoke methods from [erlrcdynamic](http://code.google.com/p/erlrc/source/browse/trunk/src/erlrcdynamic.erl), which contains methods for starting, stopping, upgrading, and downgrading individual OTP applications.

These shell scripts look at $ERLRC\_ROOT/nodes for a list of nodes to manage; the filenames are the node names (on the localhost) and the file contents are the node cookie.
```
% ls /etc/erlrc.d/nodes
cb8eec1a1b85ec017517d3e51c5aee7b
```
We use md5sums to generate node names at our company, hence the funny node name.[[3](#3.md)]

Here's a table of the shell scripts, their corresponding methods in erlrcdynamic, a brief description of what they do, and what phase in the [debian package installation procedure](http://www.debian.org/doc/debian-policy/ch-maintainerscripts.html) it should be invoked.[[4](#4.md)]

| shell script | erlrcdynamic method | what | when (debian) | when (rpm) |
|:-------------|:--------------------|:-----|:--------------|:-----------|
| erlrc-start  | start               | starts an application (idempotent) | _postinst_ configure ; _postinst_ abort-remove | %posttrans |
| erlrc-stop   | stop                | stops an application (idempotent) | _prerm_ remove | %preun (installation count == 0) |
| erlrc-upgrade | upgrade             | upgrades an application | _postinst_ upgrade |  %preun (installation count > 0) |
| erlrc-downgrade | downgrade           | downgrades an application | _postinst_ abort\_upgrade | n/a        |

If you use [framewerk](http://code.google.com/p/fwtemplates) package hooks are created for you automatically by [fw-template-erlang](http://code.google.com/p/fwtemplates/wiki/FwTemplateErlangWalkthrough), and these hooks are inert unless erlrc is installed (which is automatically listed as a recommended package).

Finally, a note on [included applications](http://www.erlang.org/doc/design_principles/included_applications.html).  erlrc tries to "do the right thing" with included applications.  This is interpreted as:

  1. if an application is listed in the $ERLRC\_ROOT/applications directory, then it should be running.
  1. when an application is running, all of its included applications are considered running.
  1. an application cannot both run itself and be included.

Therefore, for applications `X` and `Y` both listed in $ERLRC\_ROOT/applications:

  * if `X` is included by `Y`, then before starting `Y`, `X` is stopped.
  * if `X` is included by `Y`, then after stopping `Y`, `X` is started.
  * if `Y` is upgraded or downgraded such that `X` becomes included when it was not previously, before `Y` is upgraded or downgraded, `X` is stopped.
  * if `Y` is upgraded or downgraded such that `X` becomes no longer included when it was previously, after `Y` is upgraded or downgraded, `X` is started.

We've found these rules work for us in the few cases we've used included applications.

Here's a complete flow of the logic:

### Upgrade Logic ###

  1. If there is an appup file in the new application version, use it; otherwise, generate one automatically.
  1. Ensure all modules listed in the old application specification are loaded.
  1. Find any added and removed included applications by examining the old and new OTP app files.
  1. Stop any newly included applications.
  1. Execute release\_handler:eval\_appup\_script/4 .
  1. Start any formerly included applications, if they are listed in $ERLRC\_ROOT/applications.

### Downgrade Logic ###

Very similar to upgrade, except the treatment of included applications is reversed.

  1. If there is an appup file in the new application version, use it; otherwise, generate one automatically.
  1. Ensure all modules listed in the new application specification are loaded.
  1. Find any added and removed included applications by examining the old and new OTP app files.
  1. Stop any removed included applications (they will be included by the version being downgraded to).
  1. Execute release\_handler:eval\_appup\_script/4 .
  1. Start any added included applications (they are no longer included after the downgrade), if they are listed in $ERLRC\_ROOT/applications.

## Automatic .appup file generation ##

You can use erlrcdynamic with [appup files](http://www.erlang.org/doc/man/appup.html) you have generated yourself.  However we quickly noticed that in most cases appup files could be automatically generating by inspecting the source at upgrade (downgrade) time.  We therefore included automation of most of the common cases outlined in the [appup cookbook](http://www.erlang.org/doc/design_principles/appup_cookbook.html).

The automatic scheme does not cover all possible scenarios, but since developing this automatic appup file generation code, we have not had to write a single appup file manually.

Here's an outline of the logic followed for generating the upgrade portion of the appup file.  It should be noted that this logic is executed at the point in the package installation process where both the old and new code are installed on disk, so that both can be inspected.

  1. Compute added and removed modules by looking at the old and new OTP app files.
  1. Emit a load\_module directive for every added module.
  1. For each module which is in both versions of the application:
    1. If the module implements the supervisor behaviour
      1. Emit an "upgrade for supervisors" instruction.
      1. If the module exports a sup\_upgrade\_notify/2 function, emit an instruction to call it.
    1. Else if the module exports a code\_change/3 function, emit a update directive for that module.
    1. Otherwise, emit a load\_module directive for that module.
  1. If there is a start module for the application:
    1. Determine if the new beam for the start module exports a version\_change/2 function.  If so, emit a directive to call it.
  1. Emit a delete\_module directive for every removed module.

The logic for computing the downgrade portion of the appup file is similar but reversed.

### Offline Generation ###

By request, we have isolated the appup generation logic into an escript called [erlrc-makeappup](http://code.google.com/p/erlrc/source/browse/trunk/erlrc/src/erlrc-makeappup).  You can invoke this script in order to generate an appup file anywhere you have the old and new versions of your software simultaneously installed.  This can be useful if you want automatic appup generation but don't want to employ erlrc at launch-time.

## Speciality Hooks ##

We added some special hooks to make automatic appup file generation more comprehensive.

### sup\_upgrade\_notify/2 ###

If you export sup\_upgrade\_notify/2 in your supervisor modules, it will be called whenever the supervisor is upgraded, with the arguments OldVsn (old version) and NewVsn (new version).  You can use this to start or stop any added or removed children from the supervisor, since (as indicated in the appup cookbook) the supervisor upgrade instruction does not start or stop children.  Note your supervisor will have to be a registered process for this hook to be useful.

Here is a generic sup\_upgrade\_notify/2 that will probably work for you.
```
sup_upgrade_notify (_Old, _New) ->
  { ok, { _, Specs } } = init ([]),

  Old = sets:from_list (
          [ Name || { Name, _, _, _ } <- supervisor:which_children (?MODULE) ]),
  New = sets:from_list ([ Name || { Name, _, _, _, _, _ } <- Specs ]),
  Kill = sets:subtract (Old, New),

  sets:fold (fun (Id, ok) ->
               supervisor:terminate_child (?MODULE, Id),
               supervisor:delete_child (?MODULE, Id),
               ok
             end,
             ok,
             Kill),

  [ supervisor:start_child (?MODULE, Spec) || Spec <- Specs ],
  ok.
```

### version\_change/2 ###

If you export version\_change/2 in your start modules, it will be called whenever the application is upgraded, with the arguments From (old version) and StartArgs (verbatim from the new OTP application file).

# Footnotes #

### 1 ###

If you have circular dependencies then you will have transients in your system when things aren't working while the installs happen.  We think (but are not sure) that the release handler has problems too, since the low-level primitives can only atomically upgrade one module at a time.  However maybe we're wrong about that.  In practice, we haven't have circular dependencies.

### 2 ###

There are two concepts of dependency here.  The first is an OS package manager dependency, the second is the OTP application specification dependency.  erlrcdynamic uses the former (implicitly, since it gets invoked by the OS package manager), and erlrc\_boot uses the latter.  You have to arrange for both types of dependencies to be correct.

### 3 ###

I'm not going to show you the cookie, but it's basically `` cookie=`cat /etc/erlrc.d/nodes/cb8eec1a1b85ec017517d3e51c5aee7b` ``.

### 4 ###

Adjust as appropriate for your OS package manager if not using a dpkg or rpm based system ... and email me the result!

### 5 ###

Best practice is to have your build system generate these hooks automatically when the package is made, to avoid human error.