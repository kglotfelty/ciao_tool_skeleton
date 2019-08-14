# CIAO tool skeleton 

Historically it has been difficult for people to write C/C++ programs
that use CIAO libraries.  One of the issues has been trying to 
understand the dependencies of the libraries and how to create 
`Makefile`s.  Until now, the easiest way to build new applications 
was to download and perform a source build. Users could then hook into
`Makefile.master`, `Makefile.scidev`, and|or `Makefile.cxc` to use the
variables for the various libraries.

This repository provides a skeleton set of `autoconf` / `automake` files that
allow the user to build new tools from the binary download alone.
The libraries and header files are that are really needed -- what's missing
is how to actually link the executables.  Thanks to the `pkgconfig` files
we can simply resolve this (somewhat).

Note: this skeleton applies to command line tools only and does not 
apply to GTK applications.

## Requirements

You will need to have the following packages installed on your system

- `aclocal`
- `automake`
- `autoconf` 


## Quick Start 

1. Download skeleton
2. Copy skeleton files into tool directory.
3. Move tool source code into the `src` directory.
4. Edit `./configure.ac`
5. Edit `src/Makefile.am`
6. Change `mytool` in `wrapper/Makefile.am`
7. Change `mytool` in `test/Makefile.am`
8. Run `./autogen.sh`
9. `./configure`, `make`, `make check`, `make install` 

## Step by step

In the example commands shown below I used the existing `dither_region` CIAO
tool.

### 1. Download skeleton

```bash
git clone https://github.com/kglotfelty/ciao_tool_skeleton 
```

### 2. Copy skeleton files into tool directory

Here I assume that the source code for your tool is in a separate directory.


```bash
cd ciao_tool_skeleton
/bin/ls .autom4te.cfg Makefile.am autogen.sh configure.ac pkgconfig/* src/* test/* wrapper/* | \
  cpio -pdv /pool1/kjg/dither_region
```

### 3. Move source code to `src` directory

```bash
cd /pool1/$USER/dither_region

mv *.[ch] src/
mv *.{par,xml} src/
```

If you have a CIAO-style regression test script, `mytool.t`, then you 
should copy it into the `test` directory.


```bash
mv *.t test/
```


### 4. Edit `configure.ac`

This is **NOT** an expertly crafted configure script.  It has several
hacks to work around issues with `pkgconfig` files in older versions
of CIAO and some probably less-than-portable constructs.  

**YOUR RESULTS MAY VARY**

#### 4.1. Update `AC_INIT`

```m4
AC_INIT([dither_region], [4.11.0], [cxchelp@cfa.harvard.edu])
```

#### 4.2. Update `PKG_CHECK_MODULES`

This is where the real magic happens.  With the `pkgconfig` files 
setup we only need to list the libraries that this tool 
actually uses.  We do not need to know all the dependencies or 
try to figure out the correct link order.  All that happens 
automatically.

Here is a short lookup table that maps the C/C++ include file vs. 
the pkgconfig name

| include file | pkgconfig name|  description
|--------------|---------------|------------------------------------|
| caldb4.h     | caldb4        | Query CALDB for calibration files  |
| parameter.h  | cxcparam      | Prompt for parameter values        |
| dslib.h      | ds            | Misc. helper routines              |
| liberr.h     | err           | Error reporting                    |
| hdrlib2.h    | hdr           | Header propagation and manipulation|
| histlib.h    | hist          | History generation                 |
| pixlib.h     | pix           | Chandra coordinates                |
| psflib.h     | psf           | Chandra point spread function      |
| stack.h      | stk           | Stacks of files (or values)        |
| tcd.h        | tcd           | Transform, Convolution, Deconv.    |
| ardlib.h     | ardlib        | Aux. Ref. Data library (ARF,etc)   |
| ascdm.h      | ascdm         | CXC Datamodel I/O library          |
| cxcregion.h  | region        | Geometric shapes                   |


Checking our tool we come up with the following list

```m4
PKG_CHECK_MODULES( [CIAO], [
  region
  ascdm
  cxcparam
  stk
  err
  ds
  pix
  ardlib
])
```


#### 4.3 `AC_CONFIG_FILES` CIAO wrapper script

I use `configure` to create the CIAO wrapper script.  We need
to replace `mytool` with the correct tool name:

```m4
AC_CONFIG_FILES([wrapper/dither_region:wrapper/wrapper.sh],[chmod +x wrapper/dither_region])
```

#### 4.4 Local extras

Some of the existing CIAO tools make use of _local_ _libraries_.
These are small libraries that are only used by a few tools and are 
not included as part of the CIAO binary packages. They are not installed;
they are used _locally_ in the CIAO source tree during the build process.

This includes things like `libtcdio.a`, `libdmimgio.a`, `libemap.a`,
`libbary.a`, and `libVoronoi.a`.

Similarly, some tools directly share source code files with other tools such 
as `reproject_image`+`reproject_image_grid`+`dmregrid2`,
`dmimgfilt`+`dmtabfilt`+`dmimgproject`, and `mkwarf`+`eff2evt`. 

(Both these lists may be incomplete.)

Those tools require additional considerations when trying to build
using this skeleton.

For example, I have isolated the `dmimgio` _local_ _library_, created a 
stand alone github repro for it, and install it separately into the CIAO 
distribution.  The skeleton `configure.ac` checks for it

```m4
# -------- dmimgio.h ------------
foo=$CPPFLAGS
CPPFLAGS=$CIAO_CFLAGS
AC_CHECK_HEADER( dmimgio.h, [], 
  AC_MSG_ERROR([Cannot locate dmimgio.h header file], [1])
)
CPPFLAGS=$foo
```

Since `dither_region` does not use `dmimgio`, we can comment out that part.

### 5. Edit `src/Makefile.am`

#### 5.1 Change `mytool`

The way `automake` works, the variable names need to match the executable
so I need to change all instances of `mytool` to `dither_region`

```Makefile
tool = dither_region
...

dither_region_SOURCES = dither_region.c
dither_region_CPPFLAGS = $(CIAO_CFLAGS)
dither_region_LDADD = $(CIAO_LIBS) 
dither_region_LINK = $(CXX) -o $@ -Wl,-rpath,$(prefix)/lib -Wl,-rpath,$(prefix)/ots/lib 
```

Note: For arcane historical reasons, all CIAO tools must use the
C++ linker.






#### 5.2 Update `SOURCES`

The `SOURCES` variable must include all the source code file
names, including any header files.  (This is required so that `make dist`
will include them when it builds the tar file.)

```Makefile
dither_region_SOURCES = ard_pix.c asp.c convex_hull.c dither_region.c \
  dither_region.h dtf.c gti.c imap.c output.c psf.c region.c \
  t_dither_region.c
```


#### No ahelp?

If the tool does not have a CIAO style, XML-format ahelp file
then you should comment it out as shown here:

```Makefile
dist_param_DATA = $(tool).par
#dist_ahelp_DATA = $(tool).xml

install-data-hook:
	chmod a-w $(paramdir)/$(dist_param_DATA)
#	chmod a-w $(ahelpdir)/$(dist_ahelp_DATA)
```

### 6. Change `mytool` in `wrapper/Makefile.am`

The CIAO wrapper is created by the configure script; it is
installed via  wrapper/Makefile. 

```Makefile
dist_bin_SCRIPTS = dither_region
```

### 7. Change `mytool` in `test/Makefile.am`

Finally, replace the test script name in `test/Makefile.am`.

```Makefile
TESTS = dither_region.t
```

The way the skeleton is setup, it looks for the input 
data in `test/indata/dither_region` and the baseline
data for comparison in `test/save_data/dither_region`.

If you don't have CIAO tool style test script then you 
can omit the `test` directory from the top level `Makefile.am`
and remove `test/Makefile` from `configure.ac`

The skeleton is setup so that you can `make check` before you
`make install`.  This is on a tool-by-tool basis. The CIAO 
libraries must be `install`ed already to build the tool.


### 8. Run `./autogen.sh`

At this point the tool has now been grafted onto the skeleton.
We can then use the standard suite of `autoconf|aclocale|automake`
tools to generate the configure script.  The `autogen.sh` script
in the skeleton runs these commands.

```bash
$ ./autogen.sh 
configure.ac:5: installing './ar-lib'
configure.ac:4: installing './compile'
configure.ac:2: installing './install-sh'
configure.ac:2: installing './missing'
src/Makefile.am: installing './depcomp'
parallel-tests: installing './test-driver'
```


### 9. `./configure`, `make`, `make install`

`autogen` creates a `configure` script with the usual 
options, to ahead and configure pointing to your CIAO installation


```bash
$ ./configure --prefix=/soft/ciao
...
checking for CIAO... yes
...
```
The `checking for CIAO` line is where it use `pkg-config` to examine
the CIAO pkgconfig files to determine the link line.


You can then `make` your tool

```bash
$ make
...
g++ -o dither_region -Wl,-rpath,/soft/ciao/lib -Wl,-rpath,/soft/ciao/ots/lib  
  dither_region-ard_pix.o
  dither_region-asp.o dither_region-convex_hull.o
  dither_region-dither_region.o dither_region-dtf.o
  dither_region-gti.o dither_region-imap.o dither_region-output.o 
  dither_region-psf.o dither_region-region.o dither_region-t_dither_region.o 
  -L/soft/ciao/lib -L/soft/ciao/ots/lib -lds -lerr -lNewHdr2 
  -lcaldb4 -lstk -lardlib -lpix -lm -lascdm -lcxcparam -lcfitsio 
  -lregion -lreadline -lhistory   -L/soft/ciao/ots/lib -lstdc++ 
  -lreadline -lhistory -lncurses -ltinfo
```

While still a long link line, it's a lot easier then trying to figure out 
the order yourself.

And as per-usual you can then


```bash
$ make check     # aka make test
$ make install
$ make clean

$ make dist      # creates tar file for distribution
$ make distclean # cleans up generated files
``` 





