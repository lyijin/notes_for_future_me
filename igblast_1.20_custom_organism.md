# Adding a custom organism to IgBLAST 1.20 #

`IgBLAST` is a fairly old tool written by NCBI--but it's been continually updated! Kudos to the authors/maintainers. Most up-to-date docs can be found here: https://ncbi.github.io/igblast/.

I am involved in a project that analyses IGHV/IGHD/IGHJ data from alpacas. Unfortunately, alpaca isn't on the list of supported organisms out of the `IgBLAST` box. As recent `IgBLAST` versions (well, since 1.16, Apr 2020) claim that they added the ability to define custom organisms, I wanted to give that a go so that I could analyse the millions of NGS reads locally on my workstation.

The entire process was harder than I thought, hence documenting my trial-and-error experiences here in case it helps others/myself-from-the-future. Advice is current as of **Nov 2022**. Hopefully future versions are less frustrating to get to work.

## Required downloads ##

Well, one can't IgBLAST without the tool itself. I followed NCBI's instructions, and downloaded Linux x64 binaries from https://ftp.ncbi.nih.gov/blast/executables/igblast/release/LATEST. I downloaded it into `~/tools/` with `wget`, and `tar zxvf`-ed it.

V/D/J sequences from non-model organisms can be found on IMGT's website (and this is linked from NCBI's docs). Go to https://www.imgt.org/vquest/refseqh.html#VQUEST, and download IGHD/IGHJ **nucleotide** files corresponding to your organism of interest from **"F+ORF+in-frame P"**; for IGHV, download the **nucleotide** file from **"F+ORF+in-frame P with IMGT gaps"**--this is needed for a script that I wrote (more about it later).

Alpacas were, thankfully, not too far off the model organism beaten track, so there are IMGT files for it. If yours isn't on it, I'm afraid I have no advice for that...

## Some potholes worth mentioning now ##

IMGT is a *very* comprehensive website for anything immunogenetics related. Its... extensive history is evident from its site design. It takes a while to understand how to get to pages you deem interesting--and when you find useful stuff, commit them to bookmarks. Being able to find something consistently on the IMGT website was... something I had trouble with. Sometimes I found it easier to Google what I wanted with "IMGT" added to the query to get back to stuff I remember seeing in the past.

IgBLAST does not give a lot of guidance when it comes to adding custom organisms (yes I know they have a section for that in the docs, but it's... sparse). Also, there are weird crashes with weird error messages that... lead me nowhere. Hence my writing this.

## Back to business ##

This is the contents of the decompressed tarball--ignore `myseq.fa` though. I copied a single sequence into the folder so that I could use it to test the binaries. Also ignore `database/`, that contains a file that was downloaded later. Official docs (https://ncbi.github.io/igblast/cook/examples.html) mention the existence of `database/`, but this is NOT PROVIDED WITH THE 1.20 BINARIES.

```shell
$ ~/tools/ncbi-igblast-1.20.0$ ls -l
total 68K
-rw-r--r-- 1 lie128 lie128 5.1K Oct 14 07:06 ChangeLog
-rw-r--r-- 1 lie128 lie128  28K Oct 14 07:06 LICENSE
-rw-r--r-- 1 lie128 lie128   86 Oct 14 07:06 README
drwxr-xr-x 2 lie128 lie128 4.0K Oct 14 07:06 bin/
drwxr-xr-x 2 lie128 lie128 4.0K Nov  8 16:53 database/
drwxr-xr-x 8 lie128 lie128 4.0K Nov  8 16:53 internal_data/
-rw-r--r-- 1 lie128 lie128 1.3K Nov  9 17:29 myseq.fa
-rw-r--r-- 1 lie128 lie128 4.5K Oct 14 07:06 ncbi_package_info
drwxr-xr-x 2 lie128 lie128 4.0K Nov  9 17:23 optional_file/
```

Just in case you're curious what's in the FASTA file.

```shell
$ ~/tools/ncbi-igblast-1.20.0$ cat myseq.fa
>M05960:439:000000000-KRCD9:1:1101:11067:6403
CTAGTGCGGCCGCTGGAGACGGTGACCTGGGTCCCTTTGCCCCAGTAGTCCATGCCGTAGTTGGGCGCGATACACATACCATGAGAAGTAACCATGAAATGGCGTGCGAGTGCTGCACAGTAATACATGGCTATGTCCTCAGGTTTCAGACTGTTCATTTGCAGATACACCGTGTTCTTGGCGCTGTCTCTGGAGATGGTGAATCGGCCCTTCGCGGAGTCTGTATAATATGTGCTACCATCATTACCACTAATACATGAGACCCCCTCACGCTCCTTCCCTGGGGCCTGGCGGAACCAGCCTATAATATAATCATCCAAAGTAATTCCAGAGGCTACACAGGAGAGTCTCAGAGACCCCCCAGGCTGCACCAAGCCTCCTCCAGACTCCTGCAGCTGCACATC
```

## Sanity check: pretend sequence is from model organism first ##

As defining a custom organism could result in a bunch of errors, I wanted to make sure that the commandline `IgBLAST` works on a model organism first.

I would have expected a minimal command to work--I mean, model organism and all, right?

```shell
~/tools/ncbi-igblast-1.20.0$ bin/igblastn -organism mouse -query myseq.fa
BLAST Database error: No alias or index file found for nucleotide database [mouse_gl_V] in search path [/home/lie128/tools/ncbi-igblast-1.20.0::/home/lie128/blast/db:]
```

Uh... well. I see that it's picking up the BLAST folders that I defined in `~/.ncbirc`. If I added the mouse folder to that, maybe it'd work?

```shell
~/tools/ncbi-igblast-1.20.0$ export BLASTDB=/home/lie128/tools/ncbi-igblast-1.20.0/internal_data/mouse/

~/tools/ncbi-igblast-1.20.0$ bin/igblastn -organism mouse -query myseq.fa
BLAST Database error: No alias or index file found for nucleotide database [mouse_gl_V] in search path [/home/lie128/tools/ncbi-igblast-1.20.0:/home/lie128/tools/ncbi-igblast-1.20.0/internal_data/mouse:/home/lie128/blast/db:]
```

Someone posted their experience on https://github.com/xinyu-dev/igblast/blob/master/Using%20IgBlast.ipynb, saying that there's a variable called IGDATA that has to be defined, pointing to the `bin/` folder of IgBLAST. Maybe that's the hack?

```shell
~/tools/ncbi-igblast-1.20.0$ export IGDATA=/home/lie128/tools/ncbi-igblast-1.20.0/bin

~/tools/ncbi-igblast-1.20.0$ bin/igblastn -organism mouse -query myseq.fa
BLAST Database error: No alias or index file found for nucleotide database [mouse_gl_V] in search path [/home/lie128/tools/ncbi-igblast-1.20.0::/home/lie128/blast/db:]
```

No dice again.

This was when I started adapting my commands to what I read on https://ncbi.github.io/igblast/cook/examples.html. This was the suggested command.

```shell
~/tools/ncbi-igblast-1.20.0$ bin/igblastn -germline_db_V database/mouse_gl_V -germline_db_J database/mouse_gl_J -germline_db_D database/mouse_gl_D -organism mouse -query myseq -auxiliary_data optional_file/mouse_gl.aux -show_translation -outfmt 3
```

And I was like... what the heck is "database/mouse_gl_V" and the other files? It wasn't in the binary. ARGH!

Eventually I discovered https://ftp.ncbi.nih.gov/blast/executables/igblast/release/database/. There is a file "mouse_gl_VDJ.tar". Aha. This is the annoying thing--if it wasn't for my random clicking, I wouldn't have discovered what these files are. It feels like the docs were written for an IgBLAST several versions ago, and files that stopped being distributed weren't accounted for in the docs.

I created a folder `database/`, `wget`-ed the file, and extracted the files from the tarball.

And with a bit of trial-and-error, this command FINALLY worked for me

```shell
~/tools/ncbi-igblast-1.20.0$ bin/igblastn -germline_db_V database/mouse_gl_V -germline_db_J database/mouse_gl_J -germline_db_D database/mouse_gl_D -organism mouse -query myseq.fa -auxiliary_data optional_file/mouse_gl.aux -show_translation -outfmt 3
```

and the results were similar to `IgBLAST`-ing online (https://www.ncbi.nlm.nih.gov/igblast/) and selecting "mouse".

I still haven't figured out how the flags `-germline_db_?` work. Running the same command from a different folder with absolute paths DID NOT WORK. ARGH!!

```shell
~/from/elsewhere$ /home/lie128/tools/ncbi-igblast-1.20.0/bin/igblastn -germline_db_V /home/lie128/tools/ncbi-igblast-1.20.0/database/mouse_gl_V -germline_db_J /home/lie128/tools/ncbi-igblast-1.20.0/database/mouse_gl_J -germline_db_D /home/lie128/tools/ncbi-igblast-1.20.0/database/mouse_gl_D -auxiliary_data /home/lie128/tools/ncbi-igblast-1.20.0/optional_file/mouse_gl.aux -organism mouse -query tmptmp.nt.fa -outfmt 3
BLAST query/options error: Germline annotation database mouse/mouse_V could not be found in [internal_data] directory
Please refer to the BLAST+ user manual.
```

Same files, just with absolute paths, weird errors. This is why I mentioned needing to sanity check BEFORE we dare think about custom organisms. If it's already crashing so often with "mouse", I would not have been able to figure out how to add "alpaca" in had I not understood what crashes the program.

TL;DR YOU HAVE TO RUN IGBLAST FROM ITS FOLDER. IT DOES NOT LIKE BEING RUN FROM ELSEWHERE. NO I DON'T KNOW WHY.

## Adding a custom organism ##

Now that I have a better handle of how things work/crash... onwards we go.

The gist is that we need to add another folder in `internal_data/` (not `database/`. Really. It works and at this point I've stopped questioning why things work).

```shell
~/tools/ncbi-igblast-1.20.0$ ls -l internal_data/
total 24K
drwxr-xr-x 2 lie128 lie128 4.0K Nov  9 15:21 alpaca/
drwxr-xr-x 2 lie128 lie128 4.0K Oct 14 06:08 human/
drwxr-xr-x 2 lie128 lie128 4.0K Oct 14 06:08 mouse/
drwxr-xr-x 2 lie128 lie128 4.0K Oct 14 06:08 rabbit/
drwxr-xr-x 2 lie128 lie128 4.0K Oct 14 06:08 rat/
drwxr-xr-x 2 lie128 lie128 4.0K Oct 14 06:08 rhesus_monkey/
```

Note the extra `alpaca/` folder. So what sits in it?

```shell
~/tools/ncbi-igblast-1.20.0/internal_data/alpaca$ lr
total 336K
-rw-r--r-- 1 lie128 lie128  35K Nov  8 09:25 alpaca_V.imgt.fa
-rw-r--r-- 1 lie128 lie128  873 Nov  8 09:25 alpaca_D.imgt.fa
-rw-r--r-- 1 lie128 lie128  961 Nov  8 09:26 alpaca_J.imgt.fa
-rw-r--r-- 1 lie128 lie128  26K Nov  8 09:30 alpaca_V
-rw-r--r-- 1 lie128 lie128  265 Nov  8 09:30 alpaca_D
-rw-r--r-- 1 lie128 lie128  427 Nov  8 09:30 alpaca_J
(snip)
```

The first three are the files downloaded from IMGT. V has gaps, D and J do not have gaps (well, they're not provided with gaps anyway). The file names for these files are up to you, does not matter.

Having gaps in V does not matter, as IgBLAST has a Perl script that you have to run before `makeblastdb`. That script strips all gaps anyway. Run these commands VERBATIM. The FASTA FILES DO NOT HAVE SUFFIXES. THESE FILES HAVE TO BE NAMED THAT EXACT WAY.

```shell
~/tools/ncbi-igblast-1.20.0/internal_data/alpaca$ ../../bin/edit_imgt_file.pl alpaca_V.imgt.fa > alpaca_V
~/tools/ncbi-igblast-1.20.0/internal_data/alpaca$ ../../bin/edit_imgt_file.pl alpaca_D.imgt.fa > alpaca_D
~/tools/ncbi-igblast-1.20.0/internal_data/alpaca$ ../../bin/edit_imgt_file.pl alpaca_J.imgt.fa > alpaca_J
```

Then you run `makeblastdb` on it. NO ROOM FOR CREATIVITY HERE TOO.

```shell
~/tools/ncbi-igblast-1.20.0/internal_data/alpaca$ ../../bin/makeblastdb -parse_seqids -dbtype nucl -in alpaca_V
~/tools/ncbi-igblast-1.20.0/internal_data/alpaca$ ../../bin/makeblastdb -parse_seqids -dbtype nucl -in alpaca_D
~/tools/ncbi-igblast-1.20.0/internal_data/alpaca$ ../../bin/makeblastdb -parse_seqids -dbtype nucl -in alpaca_J
```

## Generating `*.ndm.imgt` coords ##

The `alpaca/` folder is now missing a `*.ndm.imgt` file that defines the start/end of FWR1/CDR1/... (note, these abbreviations assume you understand the general structure of VH/VHH). Feel free to check out `internal_data/human/human.ndm.imgt`.

```shell
~/tools/ncbi-igblast-1.20.0/internal_data/alpaca$ head ../human/human.ndm.imgt
#gene/allele name, FWR1 start, FWR1 stop, CDR1 start, CDR1 stop, FWR2 start, FWR2 stop, CDR2 start, CDR2 stop, FWR3 start, FWR3 stop, chain type, coding frame start.
#FWR/CDR positions are 1-based while the coding frame start positions are 0-based
A1      1       78      79      111     112     162     163     171     172     279     VK      0
A10     1       78      79      96      97      147     148     156     157     264     VK      0
A11     1       78      79      99      100     150     151     159     160     267     VK      0
A14     1       78      79      96      97      147     148     156     157     264     VK      0
A17     1       78      79      111     112     162     163     171     172     279     VK      0
A18b    1       78      79      111     112     162     163     171     172     279     VK      0
A19     1       78      79      111     112     162     163     171     172     279     VK      0
A2      1       78      79      111     112     162     163     171     172     279     VK      0
```

The `bin/` folder seems to have a few Python scripts to generate this, but I think they work on... OGRDB files? Not sure what they are.

I reached out to IMGT to check whether these magic numbers are hosted somewhere on their website, but I had not heard back from them. Impatient, I forged ahead... and figured out a way to generate these numbers from the **GAPPED** IGHV sequences. It was totally possible after all!

Feed https://github.com/lyijin/common/blob/master/generate_igblast_ndm_imgt.py with the IGHV gapped FASTA, and it gave me magic numbers that I saved as `alpaca.ndm.imgt`.

## Generating `*_gl.aux` coords ##

This file is in a separate folder (surprise!). They're at `optional_file/`. This one is needed for IgBLAST to produce CDR3 sequences--the aux files store nucleotide coordinates where the CDR3 stops.

```shell
~/tools/ncbi-igblast-1.20.0/optional_file$ head human_gl.aux
#gene/allele name, first coding frame start position, chain type, CDR3 stop, extra bps beyond J coding end.
#All positions are 0-based

IGHJ1*01        0       JH      17      1
IGHJ1P*01       2       JH
IGHJ2*01        1       JH      18      1
IGHJ2P*01       0       JH
IGHJ3*01        1       JH      15      1
IGHJ3*02        1       JH      15      1
IGHJ3P*01       0       JH
```

This one was something I reverse-engineered by hand for alpaca. There were 7 IGHJ sequences for alpaca. I checked how the human IGHJ sequences were stored on IMGT, then tried to understand what "first coding frame start position" meant, and where the "CDR3 stop" is at.

If you stared at the IMGT FASTA annotations long enough, I found out that the eighth column, "8. codon start, or 'NR' (not relevant) for non coding labels" is basically the aux file's "first coding frame start position" + 1.

The fourth column, I figured out what the values are by aligning the IGHJ sequences and checking which base was the 17th for human IGHJ*01 etc. Applied same logic to alpaca sequences. Sorry no shortcuts here. Hand-crafted, artisanal bioinformatics. Mmm mmm.

## Final command ##

Once all files are prepared and ready, this was the command that worked for me. Note that the `.ndm.imgt` file isn't specified anywhere. I suspect it's tied to the `-organism` flag, and it expects it to have a certain name. I think.

```shell
~/tools/ncbi-igblast-1.20.0$ bin/igblastn -germline_db_V internal_data/alpaca/alpaca_V -germline_db_J internal_data/alpaca/alpaca_J -germline_db_D internal_data/alpaca/alpaca_D -auxiliary_data optional_file/alpaca_gl.aux -organism alpaca -query myseq.fa -outfmt 3
```

Thankfully, one can change the `-query` flag to point to files in other folders, and one can redirect the text output to other folders. This command however MUST be executed in the `ncbi-igblast-x.x.x/` folder.

Another improvement is to change the `-outfmt` flag to 19 to produce tabular output. This output produces CDR1/2/3 sequences and VDJ gene usages too.

A troubleshooting tip: the `*_gl.aux` files is needed for the `-auxiliary_data` flag. If you wanted to test the command before generating this file, or if you don't care about CDR3, you can run the command sans `-auxiliary_data` just with the `.ndm.imgt` file. If the command doesn't crash, you're on the right track!
