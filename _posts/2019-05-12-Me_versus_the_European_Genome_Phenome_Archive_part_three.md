---
layout: post
title:  "Me vs. the European Genome-Phenome Archive part three: success"
date:   2019-05-12
categories: bioinformatics data
---

## The story so far

If you've been following these posts so far, you'll have XML files that [register a study and sample
samples]({{ site.baseurl }}{% post_url 2019-05-01-Me_versus_the_European_Genome_Phenome_Archive %}),
and an XML file that [registers an analysis]({{ site.baseurl }}{% post_url 2019-05-02-Me_versus_the_European_Genome_Phenome_Archive_part_two %})
(i.e. that lists all the actual data files).  And you've uploaded these to your FTP inbox and you've run them
through the test service, and it responded without any errors.

Right?

Right.  So now you need to join all this into datasets.

## Anti-aliasing

Since we've generated a unique 'alias' for each object (studies, samples, analysis files) we should
be able use that to link our datasets together, right? Wrong. For the dataset XML we need the
accessions.

To get accessions for the analysis files, we need to actually submit all the stuff we had already. I
imagine you _could_ do that by submitting to the real (non test) service, like this:

```
$ curl \
-u <ega-box-XXX>:<password> \
-F "SUBMISSION=@submit.xml" \
-F "[THING]=@[XML filename]" \
https://www.ebi.ac.uk/ena/submit/drop-box/submit/
```
You would then need to capture the response, which is another XML file, and parse it to get the accessions.

But there's an easier way: go to [this EGA Webin site](https://www.ebi.ac.uk/ena/submit/webin/) and log in.
You'll see a 'Submit' tag and a 'Submit XML files' button.  They give us a way to submit the XML files
and get useful results back.

For example: start by choosing your submission xml (`submit.xml`) and your study xml (`study.xml`)
from [this post]({{ site.baseurl }}{% post_url
2019-05-01-Me_versus_the_European_Genome_Phenome_Archive %}).  Then submit them.
You get back a tab-separated file giving you the accession for your study, and also a receipt file.
Save them.

Now do the same for each of the 'samples' and 'analysis' XMLs in turn. (To do this you have to unselect the
XML you've already selected.  I couldn't see a way to do this, but reloading the page seems to work.)  Save the results.

So now you've got a set of EGA accessions - one for the study, one for each of the samples, and importantly,
one for each of the analysis objects.

## Dataset

This is now pretty straightforward.  You basically want this:

```
<DATASETS>
    <DATASET alias="Submission of my cool datasets submission for " center_name="<center name>" broker_name="EGA">
        <TITLE>My cool dataset</TITLE>
        <DATASET_TYPE>Whole genome sequencing</DATASET_TYPE>
        <ANALYSIS_REF accession="&lt; accession of first analysis &gt;" />
        <ANALYSIS_REF accession="&lt; accession of second analysis &gt;" />
        ...
        <POLICY_REF accession="<accession of data access policy>" refcenter="&lt;center name&gt;"/>
        <DATASET_LINKS>
            <DATASET_LINK>
                <URL_LINK>
                    <LABEL>My website</LABEL>
                    <URL>&lt;URL of my website&gt;</URL>
                </URL_LINK>
            </DATASET_LINK>
        </DATASET_LINKS>
    </DATASET>
    ...
</DATASETS>
```

Before writing this you need to know:

- the accession of all your analyses (from the submission step above)
- and the accession of your data access policy (to be found from the EGA)

that's about it. But here I ran into a couple of issues.

First, although there's an EGA page [listing data access commitees](https://www.ebi.ac.uk/ega/submission/data_access_committee), there's no such page listing
policies. So if you've got an existing policy but don't know its accession, you have to ask the
[EGA helpdesk](mailto:ega-helpdesk@ebi.ac.uk).

Second, even if you do have a policy, the submission system might not know about it. Advice from
the EGA helpdesk was that this happens because these objects are too old, and were entered
manually, and the procedure has now changed completely. However this isn't much help when uploading
data.

I've chosen to workaround this by creating a new data access committee and a new data access
policy. Then I'm going to email the helpdesk and ask them to link it to the correct policy and DAC
when the data is processed. Luckily the two XMLs used for this are pretty simple:

```
<DAC_SET>
    <DAC alias="Placeholder for EGAC&lt;<my policy number>" center_name="&lt;center_name&gt;" broker_name="EGA">
        <TITLE>My data access committee</TITLE>
        <CONTACTS>
            <CONTACT name="My IDAC" email="&lt;my dac email address&gt;" organisation="&lt;my organisation&gt;" telephone_number=""/>
       </CONTACTS>
    </DAC>
</DAC_SET>
```
and
```
<POLICY_SET>
    <POLICY alias="Placeholder for EGAP&lt;<my policy number>" center_name="&lt;center_name&gt;" broker_name="EGA">
        <TITLE>My data access policy</TITLE>
        <DAC_REF accession="EGAC00001001201" refcenter="&lt;center_name&gt;"/>
        <POLICY_TEXT>See policy EGAP&lt;<my policy number></POLICY_TEXT>
        <POLICY_LINKS>
            <POLICY_LINK>
                <URL_LINK>
                    <LABEL>Data Access Agreement</LABEL>
                    <URL>&lt;my policy URL&gt;</URL>
                </URL_LINK>
            </POLICY_LINK>  
        </POLICY_LINKS>
    </POLICY>
</POLICY_SET>
```

(Note that although the dataset only needs the policy, not the DAC, you need to do the DAC because
otherwise you can't creat the policy).

These two XMLs are pretty easy to submit, and now you've got a policy accession to include in your
dataset.

## Return of the work/life balance

Check it out: ![success]({{ site.baseurl }}/images/2019-05-12 EGA success.png){:class="img-responsive"}

Sit back.  Take a deep breath.  Now go and play football.

** END **

