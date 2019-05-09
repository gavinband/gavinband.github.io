---
layout: post
title:  "Me vs. the European Genome-Phenome Archive part two: uploading data"
date:   2019-05-01
categories: bioinformatics data
---

In [part one]({{ site.baseurl }}{% post_url
2019-05-01-Me_versus_the_European_Genome_Phenome_Archive %}) of this series, we took on the EGA by
submitting a study and a set of samples to it. The EGA is now intrigued by the taste of data we've
sent it - not to mention confused by our persistence. It was pretty sure we would choose [options 1 or 2]({{ site.baseurl
}}{% post_url 2019-05-01-Me_versus_the_European_Genome_Phenome_Archive %}). In other words it's on
the back foot.

Nevertheless, there's still a long way to go.  This post is about actually getting data uploaded and submitted.
First, remember to ignore [all
the red herrings]({{ site.baseurl }}{% post_url
2019-05-01-Me_versus_the_European_Genome_Phenome_Archive %}#What-to-ignore). We're using the XML schema
to do this because it's the One True Way*.

<small>* that I got to work.</small>

## Where to put the data

The basic trick is to *place files in absolute paths on the FTP site that are the paths you want the data users to download*  *.

<small>* This may seem obvious.  But my initial instinct was to put data in well-organised subfolders of my
choosing, but that wouldn't be exposed to data users.  Turns out that won't work.</small>

For my study I want people to get a folder of data in this form:
```
    EGAS0000XXXXXX/
        dataset1_name/
            sample1.cram
            sample1.cram.crai
            sample2.cram
            sample2.cram.crai
            ...
        dataset2_name/
            sample1.cram
            sample1.cram.crai
            ...
        ...
```

Where `EGAS0000XXXXXX` is the EGA identifier for the study.  You did make a note of this right?*  So I've uploaded data into a top-level folder `EGAS0000XXXXXX` on the FTP site.

<small>* personally, I didn't know the EGA identifier when I uploaded the data. Instead I used the
placeholder folder name `EGAS0000XXXXXX` as shown above. Then I renamed it after getting the study
ID during the real submission.  Another way would be to submit the study XML from the previous post to
[the live service](https://www.ebi.ac.uk/ena/submit/webin/) first, as that will give you the EGA ID.
I'll cover that in the next post. Yet another way is to simply not name your files with the EGA identifier in.
I'm doing it because I'm stubborn that way.</small>

## How to prepare the data

You can't just upload your data to the EGA - you have to encrypt it and compute checksums first.
You upload the encrypted files (`*.gpg`), the checksums (`*.md5`), and the checksums of the
encrypted files (`*.gpg.md5`). Is all this encryption and checksumming overkill? I don't know, but that's what you have to do.

EGA provides a tool for this, called `EgaCryptor.jar`. Get it
[here](https://ega-archive.org/submission/tools/egacryptor). Run it on each file as

```
$ java -jar ../EgaCryptor.jar -file <filename>
```

I'm not going to go into how to do this across many files. Use `find -exec` to do it, or use a
compute cluster. This can be a mini-adventure, set in the broader narrative context of the larger
one, for you to complete on your own. There may be challenges on the way, but I reckon you can handle
them.

You now upload all these files to your ega box. For large directories of files, I found
[`ncftpput`](https://www.ncftp.com/ncftp/) to be the simplest tool for this. So if, locally, your
files are all in a local `EGAS0000XXXXXX` folder:

```
$ ncftpput -u <ega-box-XXX> -p <password> -R ftp.ega.ebi.ac.uk /EGAS0000XXXXXX/ ./EGAS0000XXXXXX/
```
Where `-R` specifies recursive mode, and it's `ncftpput [options] host remotedir localdir`.
So what you've got now on the ftp site is:
```
EGAS0000XXXXXX/
    dataset1_name/
        sample1.cram.gpg
        sample1.cram.gpg.md5
        sample1.cram.md5
        sample1.cram.crai.gpg
        sample1.cram.crai.gpg.md5
        sample1.cram.crai.md5
        sample2.cram.gpg
        sample2.cram.gpg.md5
        sample2.cram.md5
        sample2.cram.crai.gpg
        sample2.cram.crai.gpg.md5
        sample2.cram.crai.md5
        ...
    dataset2_name/
        sample1.cram.gpg
        sample1.cram.gpg.md5
        sample1.cram.md5
        sample1.cram.crai.gpg
        sample1.cram.crai.gpg.md5
        sample1.cram.crai.md5
        ...
    ...
```

Note there aren't any of the unencrypted files here. I acheived that by first making a parallel
directory structure with all the `.md5` and `.gpg` and `.gpg.md5` files in it (but not the original
files) before running `ncftpput`.

## How to submit a CRAM file

It's time to feed the monster again.  In this step we tell EGA about our files by submitting another XML.

This is going to get a bit complicated because the XML has to contain
- a reference to the study - as in the study XML we [submitted before]().  (This is where the 'alias' for the study is used.)
- the full name of the unencrypted file
- the md5sum of the unencrypted file
- and the md5sum of the encrypted file

For the CRAM files we're working with, it also has to contain
- a reference to the relevant sample - as in the sample XML we [submitted before]().  (This is where the 'alias' for the sample is used.)
- details on the reference used for alignment.  This includes the name and accession of the reference, and those of all the reference sequence names.

## The analysis XML

For a test with one sample I'm going to assume:

- the study alias is `my_study_v1`
- the sample alias is `illumina_hiseq:test_sample_1`
- the files are at `EGAS0000XXXXXX/dataset1/test_sample1.cram[.crai]`.

My CRAM files are aligned to GRCh37. In the analysise XML we're supposed to give an accession for this
reference - in my case it is the GenBank assembly accession
[`GCA_000001405.1`](https://www.ncbi.nlm.nih.gov/assembly/GCA_000001405.1). Moreover, you're
supposed to also give an accession for each reference contig. This makes [the full XML]({{ site.url
}}/download/cram.xml) pretty long. Here's a simplified version:

```
<ANALYSIS_SET>
<ANALYSIS alias="test_analysis_1" center_name="<center name>" broker_name="EGA" >
        <TITLE>
            Aligned reads file for illumina_hiseq:test_sample_1
        </TITLE>
        <DESCRIPTION>Aligned sequence reads for illumina_hiseq:test_sample_1, mapped to GRCh37</DESCRIPTION>
        <STUDY_REF refname="my_study_v1" refcenter="<center name>"/>
        <SAMPLE_REF refname="illumina_hiseq:test_sample_1" refcenter="<center name>" label="illumina_hiseq:test_sample_1"/>
        <ANALYSIS_TYPE>
            <REFERENCE_ALIGNMENT>
                <ASSEMBLY>
                    <STANDARD refname="GRCh37" accession="GCA_000001405.1"/>
                </ASSEMBLY>
                <SEQUENCE accession="CM00663.1" label="1"/>
                <SEQUENCE accession="CM00664.1" label="2"/>
                (etc.)
            </REFERENCE_ALIGNMENT>
        </ANALYSIS_TYPE>
        <FILES>
            <FILE filename="EGAS0000XXXXXX/dataset1/test_sample_1.cram" filetype="cram" checksum_method="MD5" checksum="fd847e0c4849ec50cdf310accd4293b0" unencrypted_checksum="d41d8cd98f00b204e9800998ecf8427e"/>
            <FILE filename="EGAS0000XXXXXX/dataset1/test_sample_1.cram.crai" filetype="crai" checksum_method="MD5" checksum="2b37te0c4994f850cdf310accd8b04c" unencrypted_checksum="e32d7cd98f00b204e5801958ecf772S"/>
        </FILES>
    </ANALYSIS>
</ANALYSIS_SET>
```

How to get the checksums in there?  Well to upload the files to the FTP site you had to encrpyt them.
So you can read the checksums from the `.md5` files that were created.

Submit with:
```
$  curl \
-u <ega-box-XXX>:<password> \
-F "ANALYSIS=@cram.xml" \
-F "SUBMISSION=@submit.xml" \
https://www-test.ebi.ac.uk/ena/submit/drop-box/submit/
```

If successful it'll return something like this:
```
<RECEIPT receiptDate="2019-05-01T13:49:59.366+01:00" submissionFile="submit.xml" success="true">
     <ANALYSIS accession="EGAZ00001399343" alias="test_analysis_1" status="PRIVATE"/>
     <SUBMISSION accession="EGA00001532515" alias="SUBMISSION-01-05-2019-13:49:59:289"/>
     <MESSAGES>
          <INFO>Submission has been committed.</INFO>
          <INFO>This submission is a TEST submission and will be discarded within 24 hours</INFO>
     </MESSAGES>
     <ACTIONS>ADD</ACTIONS>
     <ACTIONS>PROTECT</ACTIONS>
</RECEIPT>
```
If not there'll be errors.

### Some of it has issues

At this point I ran into a problem. For this XML to go in, the relevant encrypted (`.gpg`) files
have to be on the FTP site in the specified locations (though the md5sums don't seem to actually have
to be right - don't know why, but assume this is checked later).  However

*There's a delay in the system recognising what you've put on the FTP site.*

As far as I can tell, this is a delay of up to 24 hours, though I couldn't find this documented.

Depending on how you work, this might not affect you. It is a real pain in the neck for me, as I'm
a last-minute tinkerer trying to "get things right" for submission today.  But having just renamed some files,
I'm going to have to wait.  The moral is: pick your filenames carefully from the start, and stick with them.

(Another complication with the above is that my sequences were actually aligned to the `hs37d5`
reference ([available
here](ftp://ftp.1000genomes.ebi.ac.uk/vol1/ftp/technical/reference/phase2_reference_assembly_sequenc
e)) that includes [a concatenated set of decoy
sequences](https://lh3.github.io/2017/11/13/which-human-reference-genome-to-use). I couldn't find
an accession for that and don't know right now if this will come back to bite me later.)

### Submitting many CRAM files

This is now easy, right? Just repeat the `<ANALYSIS>` tags as many times as you want within the XML. Of
course, you'll have to write a computer program to generate all that XML.  You'll have to make this
code read all the `.md5` files, so that you can put the md5 values in the XML.

Should be straightforward. I'll leave it to you.

### Submitting many analyses

*Important!* If you ultimately want to submit several datasets - like I do - that implies you have
to also split up your analysis XMLs so that you submit one per dataset.

This is because the dataset XMLs we'll submit later refer, not to the samples, but to the analyses
we've submitted. In my data I want one study, 6 datasets, so I'm generating 6 files of analyses
instead of one.

### But what if I don't have CRAM files.

Oh. Well, as it turns out there are lots of different stuff that can be submitted. As the
[https://ega-archive.org/files/Analysis_BAM.xml](example XML file) says:

```
<!-- Many other filetypes can be used to define your analysis file and
    multiple file types can be submitted for a single analysis.
    For example, you may wish to submit a readme_file and
    phenotype_file to accompany your bam file: 
    <"cram"/>
    <"tabix"/>
    <"wig"/>
    <"bed"/>
    <"gff"/>
    <"fasta"/>
    <"contig_fasta"/>
    <"contig_flatfile"/>
    <"scaffold_fasta"/>
    <"scaffold_flatfile"/>
    <"scaffold_agp"/>
    <"chromosome_fasta"/>
    <"chromosome_flatfile"/>
    <"chromosome_agp"/>
    <"chromosome_list"/>
    <"unlocalised_contig_list"/>
    <"unlocalised_scaffold_list"/>
    <"sample_list"/>
    <"readme_file"/>
    <"phenotype_file"/>
    <"OxfordNanopore_native"/>
    <"other"/>
    -->
```

Unfortunately here the EGA is pernickety (I mean "controlled") and I found that I couldn't submit exactly what
I wanted.  Instead I got messages of this form:

```
 <ERROR>
 In analysis, alias:"illumina_hiseq:test_sample_1", accession:"".
 Invalid group of files: 1 "other" file, 1 "cram" file, 1 "crai" file.
 Supported file grouping(s) are:
 [1 "bam" file, 0..1 "bai" files, 0..1 "readme_file" files],
 [1 "cram" file, 0..1 "crai" files, 0..1"readme_file" files].
 </ERROR>
```
So this suggests you [can't just submit what you want](https://www.youtube.com/watch?v=oqMl5CRoFdk).
A CRAM file can only go with an index file and a README, not e.g. an annotation file or any other associated information.

This is a bit of a problem for my data, because in addition to the CRAM files I want to release

- a VCF file of array genotypes for these samples.
- annotation files for the sequenced and chip-typed samples
- and a README.

And I want to release them all inside the same dataset.

One way to do this seems to be to link them all to a genotyping analysis included within
the `ANALYSIS_SET`.  Like this:

```
<ANALYSIS alias="omni_typing_for_test_project" center_name="<center name>" broker_name="EGA" >
    <TITLE>Illumina Omni 2.5M genotyping</TITLE>
    <DESCRIPTION>Illumina Omni 2.5M genotyping</DESCRIPTION>
    <STUDY_REF refname="my_study_v1" refcenter="<center name>"/>
    <SAMPLE_REF refname="illumina_omni2.5M:test_sample_1" refcenter="MalariaGEN" label="3999807010_R01C01"/>
    <SAMPLE_REF refname="illumina_omni2.5M:test_sample_2" refcenter="MalariaGEN" label="3999807010_R01C01"/>
    ...
    <ANALYSIS_TYPE>
        <SEQUENCE_VARIATION>
        <ASSEMBLY>
            <STANDARD refname="GRCh37" accession="GCA_000001405.1"/>
        </ASSEMBLY>
        <SEQUENCE accession="CM00663.1" label="1"/>
        <SEQUENCE accession="CM00664.1" label="2"/>
        ...
        <EXPERIMENT_TYPE>Genotyping by array</EXPERIMENT_TYPE>
        </SEQUENCE_VARIATION>
    </ANALYSIS_TYPE>
    <FILES>
        <FILE filename="test_submission/test_samples.vcf.gz" filetype="vcf" checksum_method="MD5" checksum="47f0d94097b043fa4ec6a028da8c2de4" unencrypted_checksum="ac5dd03c3927f39806c4e593e78290d2"/>
        <FILE filename="test_submission/test_samples.vcf.gz.tbi" filetype="tabix" checksum_method="MD5" checksum="47f0d94097b043fa4ec6a028da8c2de4" unencrypted_checksum="ac5dd03c3927f39806c4e593e78290d2"/>
        <FILE filename="test_submission/test_samples.tsv" filetype="other" checksum_method="MD5" checksum="47f0d94097b043fa4ec6a028da8c2de4" unencrypted_checksum="ac5dd03c3927f39806c4e593e78290d2"/>
        <FILE filename="test_submission/README.md" filetype="readme" checksum_method="MD5" checksum="47f0d94097b043fa4ec6a028da8c2de4" unencrypted_checksum="ac5dd03c3927f39806c4e593e78290d2"/>
    </FILES>
    <ANALYSIS_ATTRIBUTES>
        <ANALYSIS_ATTRIBUTE>
        <TAG>platform</TAG>
        <VALUE>Illumina Omni 2.5M</VALUE>
        </ANALYSIS_ATTRIBUTE>
    </ANALYSIS_ATTRIBUTES>
</ANALYSIS>
```

For other types of data, you're [on your own](https://www.ebi.ac.uk/ega/submission/sequence/programmatic_submissions/prepare_xmls).
(or maybe I should point you at [this](https://ena-docs.readthedocs.io/en/latest/programmatic.html)
or [this](https://ega-archive.org/submission/sequence/programmatic_submissions/working_xml)
or [this](https://ega-archive.org/submission/sequence/programmatic_submissions/prepare_xml).
Yes, that's a lot of stuff to read but you didn't hear Hercules complaining, did you?)

## How goeth the quest?

The EGA, let's be honest, is fighting back.  Although we've navigated its network of complicated XML schema, it has brought
to bear stringent rules that confound our expectations, and is trying to annoy us by pretending our data isn't even there.
This is as good a time as any to give up - to choose life, say, or to choose to go and watch a colour TV, or to go back and choose [option 1 or 2]({{ site.baseurl }}{% post_url
2019-05-01-Me_versus_the_European_Genome_Phenome_Archive %}).  But I'm not going to.  I'm going to defeat the beast, and then I'm
going to have a good moan/gloat about it on my blog.




