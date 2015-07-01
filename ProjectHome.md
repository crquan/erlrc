erlrc is designed to make it easy to integrate your OS package manager with erlang's hot code loading mechanism.  It consists of an extensible boot procedure controlled by a directory, and hooks for upgrading and downgrading individual applications (with shell script interface).  Also includes an automatic appup file generation scheme which incorporates most of the cases from the [appup cookbook](http://erlang.mirror.libertine.org/documentation/doc-5.4.6/doc/design_principles/appup_cookbook.html).

Check out [ErlrcInstall](http://code.google.com/p/erlrc/wiki/ErlrcInstall) to get started.

**Update**: the download page has extra utilities:
  * erlstart: some helper scripts for starting and talking to an Erlang VM
  * apt-snapshot: snapshot and restore (debian) package state; useful for mass provisioning
  * ec2-do: rolling window execution of an arbitrary command on ec2