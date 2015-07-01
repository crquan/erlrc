#summary Install howto for erlrc
#labels Phase-Deploy,Featured

# Introduction #

Do NOT attempt to build from a checkout from the source code repository: that is for wizards only. Instead, grab one of the released source tarballs from the [downloads](http://code.google.com/p/erlrc/downloads/list) tab.

This page will assist you in installing erlrc.[[2](#2.md)]

Prerequisites:
  1. gnu make
  1. working erlang installation.
    * debian: aptitude install erlang-dev
    * os/x (with [fink](http://www.finkproject.org/)): apt-get install erlang-otp
    * freebsd: pkg\_add -r erlang
    * others: ???
  1. [eunit](http://support.process-one.net/doc/display/CONTRIBS/EUnit) from process one; optional but 'make check' will not do anything without it.

Go to the [downloads section](http://code.google.com/p/erlrc/downloads/list?can=2&q=*.tar.gz&colspec=Filename+Summary+Uploaded+Size+DownloadCount) and grab the latest `.tar.gz` file.


```
# tar -zxf erlrc-0.1.14.tar.gz
# cd erlrc-0.1.14
# ./configure --prefix /usr/local && make && make -s check
...
```
The default prefix is `/usr` so if you are ok with that you can omit the prefix argument.  Hopefully what you see[[1](#1.md)] is something like
```
PASS: ok-boot-test-0
PASS: ok-boot-test-1
PASS: ok-boot-test-circular-1
PASS: ok-boot-test-circular-2
PASS: ok-boot-test-dependency-included
PASS: ok-boot-test-duplicate-included
PASS: ok-boot-test-nested-include
96% of 84 lines covered in ./erlrc.COVER.out
PASS: module-erlrc
94% of 1497 lines covered in ./erlrcdynamic.COVER.out
PASS: module-erlrcdynamic
PASS: test-start
PASS: test-stop
PASS: test-downgrade
PASS: test-upgrade
===================
All 13 tests passed
===================
```
If not, the test output for `test-FOO` is in `tests/test-FOO.out`; perhaps it is informative.

Now you can `make install`.

# Next Steps #

Check out ErlrcHowto

# Footnotes #

## 1 ##
If you see something like:
```
fw requires GNU make to build, you are using bsd make
*** Error code 1

Stop.
```
Then you are using bsd make which will not work.  Try again with gnu make, e.g.,
```
# ./configure --prefix=/usr/local && gmake -s check
```

## 2 ##

Downloadable source tarballs use automake to build. A source code checkout of the repository uses [fwtemplates](http://code.google.com/p/framewerk) to build, and if you don't know what that is, you probably want to stick with the tarballs.