---
layout: post
title:  "Me vs. the European Genome-Phenome Archive"
date:   2019-05-01
categories: bioinformatics data
---

When [Hercules faced up against
Hydra](https://en.wikipedia.org/wiki/Labours_of_Hercules#Second_labour:_Lernaean_Hydra), he didn't
have to interface with each head using its own set of API endpoints. He just lopped them off.

Sadly, things are more complex these days. Any hero(in)es planning to submit data to the [European
Genome-Phenome Archive](https://ega-archive.org) (the EGA) will face a creature of obnoxious
proportions.   Here are some options for dealing with this:

1. Don't bother.  Data release is not that important.  (recommended*)
2. Employ someone else to do it for you.  (recommended)
3. You have to do it yourself?  No way! You should totally go back and consider options 1 and 2.
4. Ok, well, you're in for a rough ride but this post might help.

<small>* this isn't really recommended.</small>

## The perils ahead

More specifically I'm going to show you how I submitted our genome sequencing data to the EGA.
What I've got is 6 folders, each containing ~50 CRAM files, a VCF file of genotypes from
array typing, and a couple of annotation files.
On the EGA I want these to appear as one study containing 6 datasets.

To do this I had to write:
- one submission XML (reused at each step)
- one study XML describing the study
- one samples XML listing all the samples (actually I split this into two, one for sequenced samples and one for microarray samples).
- an 'analysis' XML, listing all the CRAM files and all the other files.
- a 'dataset' XMLs, specifying how all the analysis files fit into the six datasets.

i.e. 6 XML files in total*.

Actually, it turned out I also had to write

- a data access committee XML, and
- a policy XML

Writing all these XML files is not much fun.  It's not any fun at all†. 
But hey, you didn't see Hercules complaining, did you?

<small>* Actually, I also had to register a new data access committee and a
new data access policy, even though we already have a
[data access committee](https://www.ebi.ac.uk/ega/dacs/EGAC00000000002) and a
data access policy.  They were among the first ones created.  I think the explanation is that
these objects are old, and hence should be ignored, but it's not a view I would
generally subscribe to.
</small>

<small>†except perhaps for that moment when your XML <em>finally goes in successfully</em>.
That's fun.  Does that make up for the time spent on it?  No.</small>

### What to ignore

This may be the most important section of the post. Ignore the
[two](https://www.ebi.ac.uk/ega/home) EGA [websites](https://ega-archive.org), the [submitter
portal](https://ega-archive.org/submitter-portal/#/), the [other submitter
portal](https://www.ebi.ac.uk/ena/submit/sra/#home), and the [Excel spreadsheet-based submission
process](https://ega-archive.org/submission/array_based/metadata). Ignore the [JSON-based REST
API](https://ega-archive.org/submission/programmatic_submissions/submitting-metadata), no matter
how tempting. Read the docs if you want to but beware that they all seem to be subtly misleading.

I am here to show you the One True Way*.

<small>*that I got to work.</small>

### What not to ignore

Even though we are ignoring the [EGA Webin site](https://www.ebi.ac.uk/ena/submit/sra/#home), we are not
going to ignore the [other EGA Webin site](https://www.ebi.ac.uk/ena/submit/webin/), because this is
the one that actually works, and it consumes the XML we'll be writing.
We will use that for our actual submission.  But since that site doesn't seem to
have a version for testing, for most of this post we'll use `curl` to submit programmatically
to the test service instead.

### What you will need

Your weapons are: an EGA submission account (presumably called something like `ega-box-XXX` with an associated
password). And you need to know the 'center name' associated with that account.  For me, the center
name is "MalariaGEN". If you don't have these things, or don't know them, your adventure is over.
[Register an account](https://ega-archive.org/submission-form.php) or [contact the EGA helpdesk](mailto:helpdesk@ega-archive.org)
about your existing account* and then come back and see me.  (Or reconsider Option 1?)

<small>* If you've got an existing data access committee / policy you want to use, now's a good
time to also ask the EGA helpdesk what their accessions are. There's a page listing
[DACs](https://www.ebi.ac.uk/ega/submission/data_access_committee) but not one listing policies.
But if you're creating new ones, this doesn't matter.</small>

### A note on filenames

The EGA seems to careth not what you call your files, but you should care. In fact what you should do,
right now and before you do anything else, is to write down the exact file names and file structure
that your data release will occupy.  And then never change it.

Why?  There are two reasons.

1. It'll stop you faffing about with renaming files, which is what I do whenever I change my mind
about how something looks.

2. The EGA submission system *does not recognise changes to its FTP inboxes
straight away*. It appears to take overnight to do it - no doubt it's a CRON job or some other such thing
running in the wee hours.

So let's say you're trying to get stuff uploaded and you suddenly realise a file in one of your
directories is named slightly wrong (like I did), or that you'd rather the README files was named
_this_ way instead of _that_ way (like I did), or that actually you want these files to appear
together in the same directory instead of in seperate ones (like I did). Each of these changes will
cost you 24 hours of waiting around before you can make any progress. You'll probably find it
frustrating (like I did).

## Get on with it

Curiously enough, the only way to defeat this monster is to submit to it.

### Submission XML

To submit anything you need this piece of XML.

```
<SUBMISSION_SET>
    <SUBMISSION alias="" center_name="<your center name>" broker_name="EGA">
        <ACTIONS>
                <ACTION>
                        <ADD/>
                </ACTION>
                <ACTION>
                        <PROTECT/>
                </ACTION>
        </ACTIONS>
    </SUBMISSION>
</SUBMISSION_SET>
```

You don't need any of the other complexities in the documentation. The `alias` can be empty, and
you don't need to list any files under the `ADD` action. Fill in your 'center name' as described
above. Save this to a file called `submit.xml`.

Test it out using the `curl` command, like this:
```
$ curl \
-u <ega-box-XXX>:<password> \
-F "SUBMISSION=@submit.xml" \
https://www-test.ebi.ac.uk/ena/submit/drop-box/submit/
```

(Fill in your EGA username and password). This sends a request to the [EGA test
service](https://www-test.ebi.ac.uk/ena/submit/drop-box/swagger-ui.html), which replies:

```
<?xml version="1.0" encoding="UTF-8"?>
<?xml-stylesheet type="text/xsl" href="receipt.xsl"?>
<RECEIPT receiptDate="2019-05-01T08:50:59.853+01:00" submissionFile="submit.xml" success="false">
     <MESSAGES>
          <ERROR>The submission must contain at least one object in addition to the submission for ADD, MODIFY and VALIDATE actions.</ERROR>
          <INFO>Submission has been rolled back.</INFO>
          <INFO>This submission is a TEST submission and will be discarded within 24 hours</INFO>
     </MESSAGES>
     <ACTIONS>ADD</ACTIONS>
     <ACTIONS>PROTECT</ACTIONS>
</RECEIPT>
```

We've got the EGA's attention! Now we need to lob something substantial into its mandibles.

## Submitting a study

For the EGA, a "study" is a collection of datasets. To submit a study you
need a study name and an abstract.

Also, you *must generate a unique identifier for this study*. This is called the alias, and it is
used to refer to the study later. It is not shared with anyone and it is
supposed to be globally unique within the submission account. So it's probably best to make it
quite specific.

So something like this:
```
<STUDY_SET>
    <STUDY alias="my_study_v1" center_name="MalariaGEN">
        <DESCRIPTOR>
            <STUDY_TITLE>My whole genome sequencing study</STUDY_TITLE>
            <STUDY_TYPE existing_study_type="Whole Genome Sequencing"/>
            <STUDY_ABSTRACT>We sequenced some stuff, and it was good</STUDY_ABSTRACT>
        </DESCRIPTOR>
        <STUDY_ATTRIBUTES>
            <STUDY_ATTRIBUTE>
                <TAG>url</TAG>
                <VALUE>https://my.study.website.org/</VALUE>
            </STUDY_ATTRIBUTE>
        </STUDY_ATTRIBUTES>
    </STUDY>
</STUDY_SET>
```
To submit it:
```
$  curl \
-u <ega-box-XXX>:<password> \
-F "STUDY=@study.xml" \
-F "SUBMISSION=@submit.xml" \
https://www-test.ebi.ac.uk/ena/submit/drop-box/submit/
```

If this works, you'll get a reply containing `<INFO>Submission has been committed.</INFO>`.

**Note**: Make a note of the EGA ID that you received.  This might be important later.  (But we're only testing right now.)

You can of course add more attributes to the study - I think they are arbitrary tag/value pairs -
but it's not clear what they end up being used for. In my study, I added a couple of citations
since that felt like the right thing to do.

(I'll admit it - there is a perverse satisfaction in having my XML consumed like this.
It's tempting to submit this again, just to see what happens.  Turns out this particular hydra won't eat
the same thing twice, so we're going to have to feed it something else.)

## Submitting a sample

This monster's appetite is piqued but far from sated. We need to feed it some actual data,
in the hope it will later be excreted onto the EGA website.  To register a sample you
need this:

```
<SAMPLE_SET>
<SAMPLE alias="illumina_hiseq:<a_unique_sample_identifier>" center_name="MalariaGEN">
<TITLE>
My whole-genome sequenced sample
</TITLE>
<SAMPLE_NAME>
    <TAXON_ID>9606</TAXON_ID>
    <SCIENTIFIC_NAME>homo sapiens</SCIENTIFIC_NAME>
    <COMMON_NAME>human</COMMON_NAME>
</SAMPLE_NAME>
<DESCRIPTION>A whole-genome sequenced human sample</DESCRIPTION>
<SAMPLE_ATTRIBUTES>
    <SAMPLE_ATTRIBUTE>
        <TAG>subject_id</TAG>
        <VALUE></VALUE>
    </SAMPLE_ATTRIBUTE>
    <SAMPLE_ATTRIBUTE>
        <TAG>sex</TAG>
        <VALUE>female</VALUE>
    </SAMPLE_ATTRIBUTE>
    <SAMPLE_ATTRIBUTE>
        <TAG>phenotype</TAG>
        <VALUE>genome</VALUE>
    </SAMPLE_ATTRIBUTE>
</SAMPLE_ATTRIBUTES>
</SAMPLE>
</SAMPLE_SET>
```

Once again the sample needs a unique alias. For me, samples were given a unique-ish identifier by
the Sanger centre when they were processed. They were genotyped and sequenced but it's not likely
there'll ever be more data on these samples, so the formula 'illumina_hiseq:&lt;Sanger sample
identifier&gt;' should be enough. For you, you might need to generate something unique.

For each sample, it turns out you must also provide:

-  a taxon id (that's the `<TAXON_ID>9606</TAXON_ID>` bit).
- a `subject_id` or `donor_id` (I don't know what the difference is).
- a `sex` or `gender`
- a 'phenotype'

That's documented on [this page](https://www.ebi.ac.uk/ega/submission#sample). I *think* that the
allowable values for `gender` and `sex` are `male`, `female`, or `unknown`.

The "phenotype` is supposed to come from the [Experimental Factor
Ontology](http://bioportal.bioontology.org/ontologies/EFO). That's a whole other beast, and as
it lunges at me I remember that my samples don't have any measured phenotypes. Panic! But quick as
a flash, I parry by writing for the phenotype the only thing that was measured - the
[genome](http://bioportal.bioontology.org/ontologies/EFO/?p=classes&conceptid=http%3A%2F%2Fwww.ebi.a
c.uk%2Fefo%2FEFO_0004420)*.

Save the above file in `sample.xml` and submit it:
```
$  curl \
-u <ega-box-XXX>:<password> \
-F "SAMPLE=@sample.xml" \
-F "SUBMISSION=@submit.xml" \
https://www-test.ebi.ac.uk/ena/submit/drop-box/submit/
```

<small>* Is the genome a phenotype?  Yes, a 3.2 billion-dimensional one.</small>

## Submitting lots of samples

The monster is still hungry! Hungry hungry hungry! We need to feed it more samples.
Do this by including multiple `<SAMPLE>` blocks in the above XML. Then submit as before*.
```
$ curl \
-u <ega-box-XXX>:<password> \
-F "SAMPLE=@sample.xml" \
-F "SUBMISSION=@submit.xml" \
https://www-test.ebi.ac.uk/ena/submit/drop-box/submit/
```

<small>* Also, I reckon
you'll want to start keeping a log of the
result of these transactions - e.g. by piping the output into a file,
`curl ... > sample_result.xml`.  But for the real run I found it easier to use
the [Webin portal](https://www.ebi.ac.uk/ena/submit/webin/) to make getting the output easier.</small>


## The journey continues

Congratulations intrepid adventurer! You have survived the first encounter. (Admittedly, we're
still in the [monster simulator](https://www-test.ebi.ac.uk/ena/submit/drop-box/swagger-ui.html),
but hey.)

In [part two]({{ site.baseurl }}{% post_url
2019-05-02-Me_versus_the_European_Genome_Phenome_Archive_part_two %}) we will conduct a flanking
manoevre via the EGA ftp site, confronting it with some actual data.
