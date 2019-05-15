---
layout: post
title:  "Me vs. the EGA part 4: losing again"
date:   2019-05-15
categories: bioinformatics data
---

It's [not over]({{ site.baseurl }}{% post_url
2019-05-12-Me_versus_the_European_Genome_Phenome_Archive_part_three %}). 
As it turns out I'd omitted to include the README files.

I decided this was such a little detail that I'd ask the EGA to just deal with it.

However, they suggested instead that I instead add them as new analyses via the
[EGA webin site](https://www.ebi.ac.uk/ena/submit/webin/).

I retorted that based on inspection of the XML schema, the analysis XML has no appropriate
`ANALYSIS_TYPE`*.

They suggested I send it in with `ANALYSIS_TYPE` set to `SAMPLE_PHENOTYPE` and file type `readme_file`.

I spent half an hour building the relevant XML file (which are more annoying than you might think,
because they are supposed to include references to the samples).  It looks like this:

```
<ANALYSIS_SET>
<ANALYSIS alias="my_README_files" center_name="<center name>" broker_name="EGA" analysis_center="<center name>">
        <TITLE>My README</TITLE>
        <DESCRIPTION>Release note for my data</DESCRIPTION>
        <STUDY_REF refname="alias of my study" refcenter="<center name>"/>
        <SAMPLE_REF refname="sample 1 alias" refcenter="<center name>" label="<sample label>"/>
        ...
        (etc.)
        <ANALYSIS_TYPE>
            <SAMPLE_PHENOTYPE></SAMPLE_PHENOTYPE>
        </ANALYSIS_TYPE>
        <FILES>
            <FILE filename="<path to release note file" filetype="readme_file" checksum_method="MD5" checksum="470d0e6f8f8f6dc1794a0d28aa63bae5" unencrypted_checksum="53405d44a4593b7bbfa2d62f7853000c"/>
        </FILES>
    </ANALYSIS>
</ANALYSIS_SET>
...
```

I sent it in.  Quoth it:
```
    In analysis, alias:"my_README_files", accession:"".
    Invalid group of files: 1 "readme_file" file.
    Supported file grouping(s) are: [ at least 1 "phenotype_file" files, any number of "readme_file" file].
```

In other words, it doesn't work.

<small>* The docs on ANALYSIS_TYPE aren't very clear. There are [these
docs](https://www.ebi.ac.uk/ega/submission/sequence/programmatic_submissions/prepare_xmls), which
talk about BAMs (REFERENCE_ALIGNMENT), VCFs (SEQUENCE_VARIATION), and Phenotype files
(SAMPLE_PHENOTYPE). I think these are the correct docs to follow for EGA, i.e. maybe these three
are all that are permitted.
</small>

<small> Confusing though because those docs link to [these
docs](https://www.ebi.ac.uk/ena/submit/read-xml-format-1-5), which point to the [XML schema
files](ftp://ftp.sra.ebi.ac.uk/meta/xsd/sra_1_5/SRA.analysis.xsd) files which list the following
analysis types: REFERENCE_ALIGNMENT, SEQUENCE_VARIATION, SEQUENCE_ASSEMBLY, SEQUENCE_FLATFILE,
SEQUENCE_ANNOTATION, REFERENCE_SEQUENCE, SAMPLE_PHENOTYPE, PROCESSED_READS, GENOME_MAP,
AMR_ANTIBIOGRAM, PATHOGEN_ANALYSIS, and TRANSCRIPTOME_ASSEMBLY. And not to be left out, there are
also [these docs](https://ena-docs.readthedocs.io/en/latest/programmatic.html), which have examples
for SEQUENCE_VARIATION and REFERENCE_ALIGNMENT nad GENOME_MAP, and feature (dead) links to the
schema. </small>

<small>But in any case it turns
out that not all of these analysis types are allowed; for example, trying to submit an analysis of type
SEQUENCE_ANNOTATION gives you: "In analysis, alias:"my_README_files", accession:"". Invalid
analysis type SEQUENCE_ANNOTATION.".
</small>
