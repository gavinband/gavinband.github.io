---
layout: post
title:  "Getting biobank data into R, part 2"
date:   2017-05-17
categories: biobank BGEN
---

Thanks to [RStudio]() and [Rcpp]() I've [succeeded]({{ site.baseurl }}{% post_url 2017-05-16-Getting_biobank_data_into_R %}
) in getting an Rcpp package up and running.  Right-ho.  Now it's down to writing the C++ code that does the work.

## Implementing the interface ##

We want to implement this function:
```
Rcpp::List load(
	std::string const& filename,
	Rcpp::DataFrame ranges
) {
```
to load data from a bgen file.  Here's how we'll do it.  

### Opening the files ###
First, we'll open the bgen file and its index.  For this I will use the `genfile::bgen::View` and `genfile::bgen::IndexQuery` classes that are part of the implementation of [bgenix]() in the bgen repo.  (The relevant `#include`s need to go at the top of the file.)

**Warning**: The API for the `View` and `IndexQuery` classes is somewhat experimental, meaning that it might change in future.  (In particular in the current design, the `View` class really represents both the bgen file, and a view of that bgen file; likewise `IndexQuery` represents both the index and a query against that index.  That's a bit weird.)  But they work, so let's use them without further ado:

```
  using namespace genfile::bgen ;
  using namespace Rcpp ;

  View::UniquePtr view = View::create( filename ) ;
  IndexQuery::UniquePtr query = IndexQuery::create( filename + ".bgi" ) ;
```
Good!  Files successfully opened (or not, in which case an exception will be thrown, we'll see how that works in a minute.)

(The `UniquePtr`s here are typedefs for `std::auto_ptr`, which represents unique ownership of the pointed-to objects.  These days we are supposed to use `std::unique_ptr` instead, but I'm keeping to `std::auto_ptr` for the moment to keep the code working on older compilers.)

### Setting up the query ###

Next we set up the query.  I'm going to assume that `ranges` is a dataframe specifying a set of genomic ranges, each with a specified `chromosome`, `start` coordinate, and `end` coordinate.  (By convention intervals in bgen are taken to be 1-based, closed intervals).  Let's add these to the query:
```
  for( std::size_t i = 0; i < ranges.nrows(); ++i ) {
    query->include_range( genfile::bgen::GenomicRange( ranges['chromosome'][i], ranges['start'][i], ranges['end'][i] ) ;
  }
```
Now I told you the API isn't perfect.  The query currently needs to be initialised before it can be added to the view:
```
  query->initialise() ;
  view->set_query( query ) ;
```
At this point, `view` represents a view of the specified ranges in the bgen file.

### Reading the variant data ###

The `load()` function has to return quite a lot of information:
* The list of variants in the query
* The list of samples in the data
* The ploidy of each sample at each variant
* The genotype probabilities for each sample at each variant

In addition, these need nice things like useful row and column names.  Let's set up storage for the data we'll need:
```
  std::size_t const number_of_variants = view->number_of_variants() ;

  // For this example we assume diploid samples and two alleles
  DataFrame variants = DataFrame::create(
    Named("chromosome") = StringVector( number_of_variants ),
    Named("position") = IntegerVector( number_of_variants ),
    Named("rsid") = StringVector( number_of_variants ),
    Named("allele0") = StringVector( number_of_variants ),
    Named("allele1") = StringVector( number_of_variants )
  ) ;
  StringVector sampleNames ;
```
(`Rcpp::StringVector` [turns out to be](http://dirk.eddelbuettel.com/code/rcpp/html/instantiation_8h_source.html) the same thing as `Rcpp:CharacterVector`.  I dunno why there are two names for the same thing.)

We'll return the genotype data as a 3 dimensional array indexed by the variant, the sample, and the genotype.  (For now we'll assume at most three genotypes exist).  Likewise the ploidy will be stored as a 2d array:
```
  Dimension data_dimension = Dimension( number_of_variants, number_of_samples, 3ul ) ;
  Dimension ploidy_dimension = Dimension( number_of_variants, number_of_samples ) ;

  NumericVector data = NumericVector( data_dimension ) ;
  IntegerVector ploidy = IntegerVector( ploidy_dimension ) ;
```

Now we're ready to get data.  First let's get the list of sample names from the bgen file (some bgen files have no sample names, in this case the View class provides a dummy set of names):
```
  view->get_sample_ids(
  	[&sampleNames]( std::string const& name ) { sampleNames.push_back( name ) }
  ) ;
````
The middle line here is a C++11 lambda function that adds each name to the list of sample names.

Now let's iterate the variants.  In the `bgen::View` class, that's done by alternately calling the `read_variant()` and the `ignore_genotype_data_block()` or `read_genotype_data_block()` methods.  (For the moment we'll ignore the genotypes themselves, I'll get back to this below).  Like this:
```
  for( std::size_t variant = 0; variant < number_of_variants; ++variant ) {
    view->read_variant( &SNPID, &rsid, &chromosome, &position, &alleles ) ;
    variants['chromosome'][i] = chromosome ;
    variants['position'][i] = position ;
    variants['rsid'][i] = rsid ;
    variants['allele0'][i] = alleles[0] ;
    variants['allele1'][i] = alleles[1] ;

    view->ignore_genotype_data_block() ; // will be fixed later
  }
```

For ease of use we'll give `variants` row names of the variant IDs:
````
  variants.attr( "row.names" ) = rsids ;
```

Finally, we need to put everything together in an `Rcpp::List` as the return value:
```
  List result ;
  result[ "variants" ] = variants ;
  result[ "samples" ] = sampleNames ;
  result[ "ploidy" ] = ploidy ;
  result[ "data" ] = data ;
```

And we're done!
```
  return( result ) ;
}
```

## Some things I learned about Rcpp

On trying to compile this I learned some things about Rcpp:

*Thing 1*: `Rcpp.h` includes R header files that define some macros, including one called `ERROR` (defined in [RS.h](https://svn.r-project.org/R/trunk/src/include/R_ext/RS.h)).  Bad, bad R!  This means you'd better `#include <Rcpp.h>` last, not before your other files, otherwise you'll get weirdo errors if you've used `ERROR` anywhere.  In my case, this broke the sqlite headers that try to define an enum value `SQLITE_ERROR`.

*Thing 2*: You can't access dataframe elements as I did above, like `mydataframe["column"][0] = 5`.  This is because Rcpp doesn't know what the type of each column in a DataFrame is.  Instead you make a reference to the column first, as in 
```
IntegerVector const& column = mydataframe["column"];
column[0] = 5 ;
```
or you use the `as<>` function to specify the type, as in
```
as< IntegerVector >( mydataframe["column"] ) = 5 ;
```

*Thing 3*: C++11 support was not turned on for me, so it worked better to replace that C++11 lambda function with a class built for the purpose, like this one:
```
struct set_sample_names {
  set_sample_names( Rcpp::StringVector* result ):
    m_result( result )
  {
    assert( result != 0 ) ;
  }
  
  void operator()( std::string const& value ) {
    m_result->push_back( value ) ;
  }
private:
  Rcpp::StringVector* m_result ;
} ;

```
which can be used like
```
  view->get_sample_ids( set_sample_names( &sampleNames ) ) ;
```
instead of the lambda function.

## Does it work?

Let's try.  Apple-shift-b again (or `R CMD INSTALL` or whatever) and let's try:
```
> library( rbgen )
> D = load( "example/example.16bits.bgen", data.frame( chromosome = '01', start = 0, end = 100000 ))
> str( D )
List of 4
 $ variants:'data.frame':	198 obs. of  5 variables:
  ..$ chromosome: Factor w/ 1 level "01": 1 1 1 1 1 1 1 1 1 1 ...
  ..$ position  : int [1:198] 1001 2000 2001 3000 3001 4000 4001 5000 5001 6000 ...
  ..$ rsid      : Factor w/ 198 levels "RSID_10","RSID_100",..: 3 111 4 122 5 133 6 144 7 155 ...
  ..$ allele0   : Factor w/ 1 level "A": 1 1 1 1 1 1 1 1 1 1 ...
  ..$ allele1   : Factor w/ 1 level "G": 1 1 1 1 1 1 1 1 1 1 ...
 $ samples : chr [1:500] "sample_001" "sample_002" "sample_003" "sample_004" ...
 $ ploidy  : int [1:198, 1:500] 0 0 0 0 0 0 0 0 0 0 ...
 $ data    : num [1:198, 1:500, 1:3] 0 0 0 0 0 0 0 0 0 0 ...
> head( D$variants )
         chromosome position     rsid allele0 allele1
RSID_101         01     1001 RSID_101       A       G
RSID_2           01     2000   RSID_2       A       G
RSID_102         01     2001 RSID_102       A       G
RSID_3           01     3000   RSID_3       A       G
RSID_103         01     3001 RSID_103       A       G
RSID_4           01     4000   RSID_4       A       G
```
It works!

Of course - we haven't actually got any genotype data yet.  For that, see the [next post]({{ site.baseurl }}{% post_url 2017-05-18-Getting_biobank_data_into_R_part_3 %}).
