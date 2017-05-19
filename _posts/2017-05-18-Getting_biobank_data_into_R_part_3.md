---
layout: post
title:  "Getting biobank data into R, part 3"
date:   2017-05-18
categories: biobank BGEN
---

Well, I've succeeded in [making an Rcpp package]({{ site.baseurl }}{% post_url 2017-05-16-Getting_biobank_data_into_R %}) and [loading variants]({{ site.baseurl }}{% post_url 2017-05-17-Getting_biobank_data_into_R_part_2 %}
) into R.  For the final piece of the puzzle it's time to actually get at the data.

## The BGEN repo API

The API for reading data was designed with the ideas that:

1. It should not impose specific data structures on client code;
2. It should provide enough information to setup whatever storage the client wanted to use;
3. and it should be easy to use.

To achieve this the BGEN code reports data back to the user via an indirect API. The user provides a 'setter' object that is supposed to know what to do with the data; the bgen code calls methods of that setter object to set the data.  This indirection makes using the code a bit more complex, but on the other hand, it's an advantage in situations like this where we want to get data into a particular data structure that bgen doesn't know about (in this case an Rcpp array).

The API is described fully [here](https://bitbucket.org/gavinband/bgen/wiki/The_parse_probability_data_API) but a short version is that, at each variant, the View class will follow this pseudocode:
```
1. initialise and set storage sizes
2. for each sample:
3.   if client want data for this sample:
4.     set ploidy and storage size for this sample
5.     set each probability value for this sample
6. finalise
```
translating into these method calls on the setter object:
```
1. initialise()
   set_min_max_ploidy()
2. for each sample i:
3.   if set_sample(i):
4.     set_number_of_entries()
5.     for each probability value p:
6.       set_value(p)
7. finalise()
```

All we have to do is implement these methods.

## Implementing the setter

Here we go:

```
struct DataSetter {
```

We want to fill a 2d array of ploidy values (represented by an `Rcpp::IntegerVector`) and a 3d array of probability values (represented by an `Rcpp::NumericVector`), that have [already been allocated]({{ site.baseurl }}{% post_url 2017-05-17-Getting_biobank_data_into_R_part_2 %}).  Unfortunately, Rcpp doesn't seem to have a multidimensional array class, so we have to do the indexing ourselves.  So let's begin by passing in pointers to the result fields, their dimensions, and also the index of the variant we are working on:

```C++
  DataSetter(
    IntegerVector* ploidy,
    Dimension const& ploidy_dimension,
    NumericVector* data,
    Dimension const& data_dimension,
    std::size_t variant_i
  ):
    m_ploidy( ploidy ),
    m_ploidy_dimension( ploidy_dimension ),
    m_data( data ),
    m_data_dimension( data_dimension ),
    m_variant_i( variant_i )
  {}
```
All these `m_` variables are declared as private members of the class.  I tend to put these at the end of the class, but I'll skip that bit for this post.  Instead let's go ahead and implement the API.  Because storage is already allocated, there's not much to do in the first two methods, but let's add some sanity checks anyway.  For the sake of this example we are assuming variants are biallelic, that all samples are at most diploid, and we'll check the data sizes are large enough:
```  
  void initialise( std::size_t number_of_samples, std::size_t number_of_alleles ) {
    // Nothing to do but run sanity checks.
			// Put them here
    if(!(
      (number_of_alleles == 2)
      && ( m_data_dimension[0] > m_variant_i )
      && ( m_data_dimension[1] == number_of_samples )
      && ( m_data_dimension[2] == 3 )
      && ( m_ploidy_dimension[0] > m_variant_i )
      && ( m_ploidy_dimension[1] == number_of_samples )
    )) {
      throw genfile::bgen::BGenError() ;
    }
  }

  void set_min_max_ploidy( genfile::bgen::uint32_t min_ploidy, genfile::bgen::uint32_t max_ploidy, genfile::bgen::uint32_t min_entries, genfile::bgen::uint32_t max_entries ) {
    if( max_ploidy > 2 ) {
      throw genfile::bgen::BGenError() ;
    }
  }
```

For the moment we want data on all samples, so let's tell bgen that.  Also to we cache the sample index here for later use:
```
  bool set_sample( std::size_t i ) {
    m_sample_i = i ;
    return true ;
  }
```

The `set_number_of_entries()` call tells us the ploidy (as well as the number of probability values) for the current sample.  Let's store that at the appropriate index:
```
  void set_number_of_entries(
    std::size_t ploidy,
    std::size_t number_of_entries,
    genfile::OrderType order_type,
    genfile::ValueType value_type
  ) {
    // Sanity checks go here
    // Compute the index for this variant and sample in the 2d ploidy matrix:
    int flatIndex = m_variant_i + (m_sample_i * m_ploidy_dimension[0]) ;
    (*m_ploidy)[ flatIndex ] = ploidy ;
  }
```

To store the values themselves is similar.  There are two versions of the `set_value()` method, one for non-missing and one for missing data.  In the latter we'll set the appropriate R missing value:
```
  void set_value( uint32_t entry_i, double value ) {
    int flatIndex = m_variant_i + (m_sample_i * m_data_dimension[0]) + (entry_i * m_data_dimension[0] * m_data_dimension[1]) ;
    (*m_data)[ flatIndex ] = value ;
  }

  void set_value( uint32_t entry_i, genfile::MissingValue value ) {
    int flatIndex = m_variant_i + (m_sample_i * m_data_dimension[0]) + (entry_i * m_data_dimension[0] * m_data_dimension[1]) ;
    (*m_data)[ flatIndex ] = NA_REAL ;
  }
```
There's nothing to do in finalise() either:
```
  void finalise() {
  }
```
And we're done!

## Using the setter object

Remember this code?
```
  for( std::size_t variant = 0; variant < number_of_variants; ++variant ) {
    view->read_variant( &SNPID, &rsid, &chromosome, &position, &alleles ) ;

    // (stuff to do with variant id data here)

    view->ignore_genotype_data_block() ; // will be fixed later
  }
```
Let's change it to read the data instead of ignoring it:
```
  for( std::size_t variant = 0; variant < number_of_variants; ++variant ) {
    view->read_variant( &SNPID, &rsid, &chromosome, &position, &alleles ) ;

    // (stuff to do with variant id data here)

    // Construct the setter object for this variant
    DataSetter setter(
      &ploidy, ploidy_dimension, &data, data_dimension, variant
    ) ;
    view->read_genotype_data_block( setter ) ;
  }
```
Job done.

A final tweak is to give the resulting data suitable names, I'll skip that here.

## Does it work?

R CMD INSTALL the package and let's test it:
```R
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
 $ ploidy  : int [1:198, 1:500] 2 2 2 2 2 2 2 2 2 2 ...
  ..- attr(*, "dimnames")=List of 2
  .. ..$ : chr [1:198] "RSID_101" "RSID_2" "RSID_102" "RSID_3" ...
  .. ..$ : chr [1:500] "sample_001" "sample_002" "sample_003" "sample_004" ...
 $ data    : num [1:198, 1:500, 1:3] 0.00168 NA 0.91611 0.00507 0.99286 ...
  ..- attr(*, "dimnames")=List of 3
  .. ..$ : chr [1:198] "RSID_101" "RSID_2" "RSID_102" "RSID_3" ...
  .. ..$ : chr [1:500] "sample_001" "sample_002" "sample_003" "sample_004" ...
  .. ..$ : chr [1:3] "0" "1" "2"
> head( D$variants )
         chromosome position     rsid allele0 allele1
RSID_101         01     1001 RSID_101       A       G
RSID_2           01     2000   RSID_2       A       G
RSID_102         01     2001 RSID_102       A       G
RSID_3           01     3000   RSID_3       A       G
RSID_103         01     3001 RSID_103       A       G
RSID_4           01     4000   RSID_4       A       G
> D$data[1,1:10,1:3]
                      0            1            2
sample_001 0.0016784924 0.0023498894 0.9959716182
sample_002 0.0070191501 0.0003662165 0.9926146334
sample_003 0.9959411002 0.0036316472 0.0004272526
sample_004 0.0002441444 0.9976195926 0.0021362631
sample_005 0.0020141909 0.9965819791 0.0014038300
sample_006 0.0023498894 0.0019226368 0.9957274739
sample_007 0.0112001221 0.9808346685 0.0079652094
sample_008 0.0072632944 0.9922178988 0.0005188067
sample_009 0.0029907683 0.9944457160 0.0025635157
sample_010 0.0024719615 0.9781490806 0.0193789578
```
So it works!

## The finished product

The code from these blogs is [currently available](https://bitbucket.org/gavinband/bgen/src/tip/R/package/) as part of the 'default' branch of the [bgen repo](http://www.bitbucket.org/gavinband/bgen).  (Commit [1332153121bb](https://bitbucket.org/gavinband/bgen/src/1332153121bb/R/package/) as of this writing).  I've made a few tweaks there:
* Calling a function 'load' is not very friendly as it masks the base function.  So I've called the main function `bgen.load` instead.
* I've arranged that the package is assembled in the build dir during the normal build phase of the bgen repo.  So after compiling the bgen repo, you can install it with `R CMD INSTALL build/R/rbgen`.
* It is highly experimental (not to mention currently largely untested) so if you do use it, use it with a liberal does of sanity checking.

I've also noticed that there are interactions between the compiler used to compile the BGEN repo, and the compiler used to compile the version of R you're using.  (They'd better be the same).  On our compute cluster, for example, I do
```
CXX=/path/to/gcc/5.4.0/bin/g++ CC=/path/to/gcc/5.4.0/bin/gcc ./waf-1.5.18 configure
```
and then I can install the package into a version of R also built with gcc 5.4.0.  I think this type of issue is going to be inevitable.

## The future

I expect to develop this further and hopefully move it to the 'master' branch when it becomes stable enough for regular use.  In the meantime, if you do try it, let me know how you get on.

[Enjoy!](https://bitbucket.org/gavinband/bgen/src/tip/R/package/).

