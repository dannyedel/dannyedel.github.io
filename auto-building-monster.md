---
---

## Motivation
I wrote a small software named `dspdfviewer`.
While it is not ready for inclusion in the official debian archives,
I want to make it installable in as many debian variants as
possible.

That means I'll need binary packages in proper debian repositories,
since those will handle automatic updates and dependency management
for my users.

Since different debian versions (or derivates) can have different
dependencies, the same binary package normally does not work on
different platforms. Generally that means rebuilding the source
package (with normally zero changes) with a tilde version suffix.

Ubuntu actually maintains such a service called launchpad, which can
follow a source repository and automatically build source and binary
packages for all currently supported ubuntu versions.

The target of this project is to create a program that will,
when run in the source directory of a package (for example my
dspdfviewer) start building source/arch-indep for sid,
and then using sbuild chroots or similar to build
all of the following:

* sid amd64 binary
* sid i386 binary
* ~deb8jessie1 source/arch-indep
* ~deb8jessie1 amd64 binary
* ~deb8jessie1 i386 binary
* ~deb7wheezy1 source/arch-indep

... and so on, for any combination of codename and architectures
you feed it. Obviously the program *itself* should be in a debian
package so it will be easily installed and updated (and it can
use itself to generate versions for all compatible debian versions)

## Why not use something that's already there?

Short answer: Because I didn't find it.

Long answer: I'm using this service for ubuntu, and on debian - as
long as your package is in the sid release - the debian infrastructure
will do that for you. But I didn't find any debian service that does
that for packages without adding them to the main debian pool.

All websites I find tend to have loads of scripts that need to be
upated and maintained, providing very much flexibility and control
over the build process, while all I need is a simple program that
takes a configuration file of "which distributions/which architectures"
and starts compiling for each and every possible combination,
throwing the result into a reprepro-managed repository.

## What commands need to be automated?

Basically, this is what I do manually right now, and it should get
automated by the software.
Asumme that $GITREV points to a tag/sha/whatever git can parse.

### Define a version number that reflects git

This step should account for the differences between "git describe"
and debian:

* Leading v:
  * In git, I call my tags `v1.11` (note the `v` at the start) but
  debian wants the version to just be 1.11
* Minus vs. Plus sign:
  * In git describe, "one revision after v1.11" will be something like
  `v1.11-1-gc0df65b`. In debian, the minus sign is reserved for debian
  revisions, so "increase" should be a plus sign, resulting in
  `v1.11+1+gc0df65b`.
* Release candidate:
  * In git convention, a suffix like `v1.11-rc1` would sort *before*
  `v1.11`. In debian, that needs to be written with a tilde:
  `v1.11-rc1` becomes `v1.11~rc1`.
  * Same counts for `-alpha` and `-beta` suffixes.
* Current bash-ism:
  * In a small bash wrapper around git describe, I emulate these.
  See my [gitdescribedebian] script.

[gitdescribedebian]: https://github.com/dannyedel/config-files/blob/master/home_nodot/bin/gitdescribedebian

### Make a clean source/indep build on sid

* use git archive to copy only the actual source files to
a temporary directory. Notice that debian prefers this
directory to have a name in the `PACKAGENAME-VERSION(-REVISION)`
format (again, `VERSION` should not contain minus signs).

Assume `$VERSION` is set to the "debian compatible version" created earlier,
and `$GITREV` to the revision we're building from.

Note the prefix parameter to `git archive` ends with a slash,
that will create a directory.

* Create a directory hosting our build
  * `mkdir -p /tmp/building/`
* Copy only versioned and commited files there:
  * `git archive --format tar --prefix="$PACKAGENAME-$VERSION/" $GITREV | tar -C /tmp/building/ -xv`
  * `cd "/tmp/building/$PACKAGENAME-$VERSION/"`
* Only relevant for continuous integration builds: Check
whether debian's changelog contains the correct revision
  * `dpkg-parsechangelog --show-field version`
  * If not, add an auto-generated changelog message:
  In the final program, it should fail here if a command-line
  switch "do daily build" was not configured.
    * `dch --newversion="$VERSION" --distribution unstable 'Auto-build from git'`
* Build source package:
  * `debuild -uc -us -S`
* Try to build architecture indepent package:
Right now, if this fails, it will be completely ignored, assuming
that the package did not have architecture independent parts.
FIXME: Somehow(tm) detect if a package has arch-indep parts.
  * `debuild -uc -us -A`
* Include source and indep package in reprepro (export=never prevents
signing, which might block while waiting for my passphrase)
  * `reprepro include --export=never sid ../${PACKAGENAME}_${VERSION}_source.changes`
  * `reprepro include --export=never sid ../${PACKAGENAME}_${VERSION}_all.changes`


### Make builds for the other architectures



#### (once) Preparation of the sbuilder

Create the sbuilder. It should(tm) later be named
`DISTRIBUTION-ARCH-sbuild`, for example `jessie-i386-sbuild`


`sudo sbuild-createchroot --make-sbuild-tarball=/var/lib/sbuild/jessie-i386.tar.gz --arch=i386 jessie $(mktemp -d) http://httpredir.debian.org/debian`



#### (each) Build binary packages for each distribution

Environment variables:

* $MAINTAINER is "Firstname Lastname <emailaddress>"
* $ARCHITECTURE is "i386" or "amd64" and so forth.
* $APPEND is something like "~deb8jessie" or "~deb7wheezy"
* $DISTRIBUTION is "wheezy" or "jessie" or "unstable"

* Quick check if it exists:
  * `schroot --chroot "${DISTRIBUTION}-${ARCHITECTURE}-sbuild" --info`
  * Return code will be 0 if it exists, nonzero otherwise.

* Build it:
  * `sbuild --arch $ARCHITECTURE --dist $DISTRIBUTION --append-to-version="$APPEND" --uploader="$MAINTAINER" --maintainer="$MAINTAINER" PACKAGENAME_${VERSION}.dsc`
