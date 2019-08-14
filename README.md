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
git clone file:///data/lenin2/Projects/kCIAO/_Skeleton skel
```

### 2. Copy skeleton files into tool directory

Here I assume that the source code for your tool is in a separate directory.


```bash
cd skel
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

#### 4.3 Local extras

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

```m4
tool = dither_region
...

dither_region_SOURCES = dither_region.c
dither_region_CPPFLAGS = $(CIAO_CFLAGS)
dither_region_LDADD = $(CIAO_LIBS) 
dither_region_LINK = $(CXX) -o $@ -Wl,-rpath,$(prefix)/lib -Wl,-rpath,$(prefix)/ots/lib 
```


#### 5.2 Update `SOURCES`

The `SOURCES` variable must include all the source code file
names, including any header files.  (This is required so that `make dist`
will include them when it builds the tar file.)

```m4
dither_region_SOURCES = ard_pix.c asp.c convex_hull.c dither_region.c \
  dither_region.h dtf.c gti.c imap.c output.c psf.c region.c \
  t_dither_region.c
```

Note: if your tool is C++, then change `CPPFLAGS` with `CXXFLAGS`

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

####  7.2. List all source files, including header files, in `mytool_SOURCES`
####  7.x. comment out ahelp
### 6. Change `mytool` in `wrapper/Makefile.am`
### 7. Change `mytool` in `test/Makefile.am`
### 8. Run `./autogen.sh`
### 9. `./configure`, `make`, `make check`, `make install` 
