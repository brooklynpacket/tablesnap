Tablesnap
=========

Theory of Operation
-------------------

Tablesnap is a script that uses inotify to monitor a directory for `IN_MOVED_TO`
events and reacts to them by spawning a new thread to upload that file to
Amazon S3, along with a JSON-formatted list of what other files were in the
directory at the time of the copy.

When running a Cassandra cluster, this behavior can be quite useful as it
allows for automated point-in-time backups of SSTables. Theoretically,
tablesnap should work for any application where files are written to some
temporary location, then moved into their final location once the data is
written to disk. Tablesnap also makes the assumption that files are immutable
once written.

Installation
------------

This distribution provides a debian/ source directory, allowing it to be built
as a standard Debian/Ubuntu package and stored in a repository. The Debian
package includes an init script that can run and daemonize tablesnap for you.
Tablesnap does not daemonize itself. This is best left to tools like
init, supervisord, daemontools, etc.

There are pre-build binaries for Ubuntu Maverick amd64 and i386 in this PPA:
<https://launchpad.net/~synack/+archive/tablesnap>

	# cat /etc/apt/sources.list.d/tablesnap.list << EOF
	> deb http://ppa.launchpad.net/synack/tablesnap/ubuntu maverick main
	> deb-src http://ppa.launchpad.net/synack/tablesnap/ubuntu maverick main
	> EOF
	# aptitude update

If you are not a Debian/Ubuntu user or do not wish to install the tablesnap
package, you may copy the tablesnap script anywhere you'd like and run it from
there. Tablesnap depends on the pyinotify and boto Python packages. These are
available via "pip install pyinotify; pip install boto;", or as packages from
most common Linux distributions.

Configuration
-------------

All configuration for tablesnap happens on the command line. If you are using
the Debian package, you'll set these options in the `DAEMON_OPTS` variable in
`/etc/default/tablesnap`.

    Usage: tablesnap [options] <bucket:prefix:path:regex> [...]
    Options:
      -h, --help            show this help message and exit
      -k AWS_KEY, --aws-key=AWS_KEY
      -s AWS_SECRET, --aws-secret=AWS_SECRET
      -r, --recursive       Recursively watch the given path(s)s for new SSTables
      -a, --auto-add        Automatically start watching new subdirectories within path(s)
      -B, --backup          Backup existing SSTables to S3 if they're not already there

    bucket: s3 bucket
    prefix: subdirectory under s3 bucket
    path: local folder location to watch
    regex: file name to match under the local path

    prefix and regex can be optionally left blank, while bucket and path are required
    the regex needs to be a valid python regex and while be compared using the 're' module and the 'match' function

For example:

	$ tablesnap -k AAAAAAAAAAAAAAAA -s BBBBBBBBBBBBBBBB me.synack.sstables:location/in/s3:/var/lib/cassandra/data/GiantKeyspace:columnfamily-hc.*db

This would cause tablesnap to use the given Amazon Web Services credentials to
backup the SSTables for my `GiantKeyspace` to the S3 bucket named
`me.synack.sstables`.


Notes
-----

Tablesnap respawns by default when launched from the init.d script

A path can only be watched once, but the regex allows you to watch different files within that path

By following this strategy, we are able to monitor both staging and prod on the same cassandra cluster
and upload to separate staging and production buckets in s3.  We can also cherry pick which
column family's from a given keyspace we want to back up


Debian Packaging
----------------

https://wiki.ubuntu.com/PackagingGuide/Complete

To build your own .deb from this repo..

git clone git://github.com/brooklynpacket/tablesnap.git
cd tablesnap
sudo apt-get install cdbs debuild
debuild -us -uc

The debuild command disables gpg key signing.
The build must take place on a Linux box.  Tested with Ubuntu Lucid.

To install the tablesnap.deb you just built..
(optionally, a pre-built .deb is available: wget ...)

apt-get install python-pip python-pyinotify daemon
pip install pyinotify boto
lesspipe tablesnap.deb
sudo dpkg --install tablesnap.deb


Questions, Comments, and Help
-----------------------------
The fine folks in `#cassandra-ops` on `irc.freenode.net` are an excellent
resource for getting tablesnap up and running, and also for solving more
general Cassandra issues.
