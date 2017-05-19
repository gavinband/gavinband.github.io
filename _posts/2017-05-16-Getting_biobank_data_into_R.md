---
layout: post
title:  "Getting biobank data into R"
date:   2017-05-16
categories: biobank BGEN
---
The UK Biobank is releasing its imputed genotypes in [BGEN format](http://www.well.ox.ac.uk/~gav/bgen_format). This format is the [apoidean's](http://www.onezoom.org/life.html/@Apoidea=419526?vis=spiral#x649,y386,w1.1187) [leg joint](https://youtu.be/uBUq1hmbtPE) but it does have one drawback, namely, it needs special code to read the files.  This means that stuff like getting data into R is currently nontrivial.

On the other hand, I've made a [C++ implementation](http://www.bitbucket.org/gavinband/bgen) available so it should be possible to hook this up.  How hard could that be?  This blog post is about building an rpackage (rbgen) that interfaces to this code.

## Getting started ##

The [bgen code](http://www.bitbucket.org/gavinband/bgen/src) is in C++, so we'll be best off using [Rcpp](http://www.rcpp.org).  Unfortunately I'm much too lazy to want to learn how R or Rcpp packages really work, but luckily [Rstudio](http://www.rstudio.com) seems to know just what to do.  I go to *File* -> *New Project*, choose a new directory, choose 'R Package' and then 'Package w/Rcpp' for the type, fill in the name (rbgen) etc. and click 'Create Project':
![Rstudio screenshot](/images/2017-05-16_rstudio_rcpp_package.png)

[Bug's your uncle](https://en.wikipedia.org/wiki/Ug_(book)), this creates a package with a folder structure like this:

```
Read-and-delete-me
DESCRIPTION
NAMESPACE
R/
  RcppExports.R
src/
  rcpp_hello_world.cpp
  RcppExports.cpp
man/
```
Also I've noticed, more or less by accident, that pressing Apple-Shift-B rebuilds the package (aka 'Build and Reload' from the build menu).

The `Read-and-delete-me` file says we should put any C++ code into src/.  Ok dokey.  It also witters on about other stuff, like editing NAMESPACE, that mean very little to me, so I'll ignore it and get on to building stuff:

## Designing the interface ##

This one is easy.  I want to load data from a bgen file:
```
data = rbgen::load(
	"myfile.bgen",
	range = list( chromosome = '1', start = 1, end = 1000000 )
) ;
```
'nuff said.

(Well maybe we'll want other features in future, like listing the variants in the file, or loading only a subset of samples, or selecting variants by ID.  Those will be addable later so for now we'll stick with the above.)

## Implementing the interface ##

Let's start by making a dummy implementation that returns an empty list.  I want something like this:

```
// in src/load.cpp
#include <Rcpp.h>
Rcpp::List load(
	std::string const& filename,
	Rcpp::DataFrame ranges
) {
	Rcpp::List result ;
	return result ;
}
```

One of the magical things about Rcpp is that this just works.  In fact all that's needed is to tell Rcpp to export this function - we do this by adding the comment line
```
// [[Rcpp::export]]
```
just above the function declaration.  I put this in a file src/load.cpp.  I hit Apple-Shift-B.  Magic things happen.  The package is built.  The package is loaded.  Cool.  Look, it even works:
```R
> rbgen::load( "hello", data.frame() )
list()
```

## Making it link ##
Now comes the hard part.  I've got some existing C++ code in the bgen package.  It is built into object files and libraries with the bgen package (that uses the [waf build system]().)  I need to link it into this package.  How?

Several lost hours later I've reached the following solution.  First, I've chosen to distribute rbgen as a subdirectory of the bgen repo.  This means I can link directly to the build results from that package, rather than building them all over again.  Second, it turns out the build is controlled by a file called `Makevars` that also lives in the `src/` directory.  `Makevars` is a [makefile]().  But it is called `Makevars`.  Go figure.  Anyway `Makevars` does not seem to exist to start off with, so I made one.  Here's what it looks like:
```
# in src/Makevars
PKG_CPPFLAGS = \
-I ../../../../genfile/include \
-I ../../../../db/include \
-I ../../../../3rd_party/boost_1_55_0 \
-I ../../../../3rd_party/zstd-1.1.0 \
-I ../../../../3rd_party/zstd-1.1.0/lib \
-I ../../../../3rd_party/zstd-1.1.0/lib/compress \
-I ../../../../3rd_party/zstd-1.1.0/lib/decompress \
-I ../../../../3rd_party/sqlite3 \
-fPIC -O3 -Wall
OBJECTS = RcppExports.o giveMeMyData.o $(wildcard ../../../src/*.o) ../../../db/libdb.a ../../../3rd_party/zstd-1.1.0/libzstd.a  ../../../3rd_party/sqlite3/libsqlite3.a ../../../3rd_party/boost_1_55_0/libboost.a

all: $(SHLIB)
$(SHLIB): $(OBJECTS) Makevars
```
In short:
* `PKG_CPPFLAGS` contains flags to pass to the C++ compiler.  In my case I give it a bunch of include paths (`-I`) pointing at files in the bgen repo, the `-fPIC` flag which turns on 'position independent code' (no idea what that is, think that means I'll be able to load data both at Oxford and at Sanger), an optimisation flag (`-O3`) and a flag to turn warnings into errors (`-Wall`).
* (`PKG_LIBS` also contains flags the compiler should pass to the linker, but it turns out I don't need these).
* `OBJECTS` lists object files and static libraries that need to be compiled and/or linked.
* `SHLIB` is the name of the target shared library - it turns out to be called `src/rbgen.so`.

The finished library depends on the object files (make magically builds the missing ones) and also on `Makevars`, so it gets rebuilt if any of these change.

Phew.

## Implementing the interface ##

Well... that'll be the subject of the [next post]({{ site.baseurl }}{% post_url 2017-05-17-Getting_biobank_data_into_R_part_2 %}).


