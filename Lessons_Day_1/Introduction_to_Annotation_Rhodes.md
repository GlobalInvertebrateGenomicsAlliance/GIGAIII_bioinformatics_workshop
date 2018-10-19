## Instructions for Annotation Tutorial for GIGA III 2018

This tutorial is specific to the files placed on the server.  For practice, you can upload your own genome and genes of interest and use the same protocol by changing the filenames to your reference and your gene of interest data set.

***We will start with a simple question - is my gene of interest in this particular genome?***

Then, progress to efficient methods for naming all the genes in the genome based on highly conserved regions.

Finally, we will talk about customizing our search databases to find genes of interest and explore them using web-based tools.

If we have time, we will discuss the annotation of noncoding regions.

---

## Organism and Genes of Interest

1.  Bugula neritina, "moss animals" 

http://www.exoticsguide.org/bugula_neritina
https://en.wikipedia.org/wiki/Bryozoa

2.  Innate immune recogniation genes

https://www.ncbi.nlm.nih.gov/pmc/articles/PMC4109969/

MyD88
Toll Interleukin Receptor Pathways (TIR, TLR)
etc., many more

---

## Question One: Are these innate immune recognition genes in my genome?

Let's create a file to test for the presence of MyD88

1.) Navigate to http://uniprot.org

2.) Search for non-vertebrate, non-arthropod MyD88 proteins. 

    A.) gene:myd88 NOT taxonomy:"Vertebrata [7742]" NOT taxonomy:"Arthropoda [6656]"
    B.) Select all the sequences (should be 18 remaining) and choose "align".
    C.) Kick out outliers (I removed A0A2B4S3X5, A0A2B4SJ75, A0A2B4RT06, Q4W1E7_SUBDO)
    D.) Return to main screen and choose "download" and format Fasta (canonical)
    E.) Copy and paste the text directly into the terminal or make a file on your desktop using notepad.
    
### Note: don't use Microsoft word for text copy/paste for command line, it adds hidden characters which will mess up the code.

3.) Make a text file that holds the resulting fasta sequences for use in a few minutes.  For this demo, call it MyD88.fasta


What is the Uniprot database?

https://www.uniprot.org/downloads

What is MyD88?

https://www.ncbi.nlm.nih.gov/pmc/articles/PMC4109969/


---

## Terminal Practice

Log into the server and follow these steps.

### Putting data on your remote computer, method 1 - creating a file and copying and pasting information.

```
nano MyD88.fasta

```

1.) Copy and paste the sequences from the text file we just created.
    If you are using a traditional terminal (putty or iterm2) this is a right click.
    If you are using the old web shell in Jetstream, right click and choose paste.
    If you are using the new web shell in Jetstream, use control+alt+shift and follow the directions.

2.) Control-X to save file

If you get stuck, here is a way to download it directly:

```
cd ~

mkdir annotation

cd ~/annotation

wget https://de.cyverse.org/dl/d/E9A541A2-907C-459F-8133-4F559CE0F3F1/MyD88.fasta
```

### Putting data on your remote computer, method 2 - WinSCP, Cyberduck, Filezilla or other external program.

Using the key provided in the slack, you can add an authorization to your log-in to allow for a remote connection to view files.  We will demonstrate if time allows.

### Putting data on your remote computer, method 3 - Start Rstudio (which we pre-installed for the class) and transfer files.

Lisa can demonstrate if we have time and if we cannot get the data into the system quickly using one of the other methods.

### Putting data on your remote computer, method 3 - download a file from an ftp or http location.

A direct link between remote computers avoids the time and effort of downloading files to your personal computer first.  This is a good option for genomes.

```
cd ~/annotation

wget https://de.cyverse.org/dl/d/A47FAD90-1837-4868-8896-61231F14F779/genome_canu_filtered.fasta

```

***Protein to Protein searches are more efficient in Blast, so let's convert our genome into a set of proteins instead of nucleotides.***

We will use Transdecoder to do this.

***What are some reasons why converting our raw scaffold into proteins will help us find our proteins?

1.) Load the program Transdecoder into our instance

```
conda

conda install transdecoder
```

After installation, check to see if you get instructions for the program by running:

```
TransDecoder.LongOrfs
```

If not, retry the install.


2.) Run the program on our reference genome.


```

export REFERENCE="~/annotation/genome_canu_filtered.fasta"

TransDecoder.LongOrfs -t $REFERENCE

```

#### Note: you can replace the demonstration reference with your genome for practice later on.  Using a variable called "reference" allows us to write code once and reuse it by just re-setting what we mean by "reference".  Using variables in your script can help you be more efficient as you can try multiple searches using the same script just by changing the reference, or feeding a list of reference genomes into your bash script.***

This can take up to an hour, so we need to stop the program prematurely.  It won't hurt anything, so just hit Control-C after a few minutes.

Now we need to make the peptide list a searchable blast database.

First, install the blast tools set from NCBI:

```

sudo apt install ncbi-blast+

```
#### Note: using sudo is usually reserved for machine owners.  In this course, we can install programs because we make our own machines.  This will not work if you are on a privately managed server, such as a university HPC.  Please consult with your system administrators on best practices for installing programs.

Check the install.

```

makeblastdb

```


1.) Copy the protein version of the genome into the annotation folder.

```
cp ~/annotation/$REFERENCE.transdecoder_dir.__checkpoints_longorfs/longest_orfs.pep ~/annotation

```

#### Note: your directory name may be slightly different, use ls to see whusing at the directory name is before the copy

2.) Make a blast database of the genome for searching.


```
makeblastdb -in longest_orfs.pep -dbtype prot -title Bugula.pep -out Bugula.pep

```

3.) Let's blast our sequences to see if we get a result.

```

blastp -query ~/annotation/MyD88.fasta -db Bugula.pep -outfmt 6 -evalue 1e-5 -out blastp.MyD88.Bugula.pep.outfmt6

```
#### Note: outfmt6 is a very versatile tool that can be used to output your gene hits as a table, read more about it here:

https://www.ncbi.nlm.nih.gov/books/NBK279682/

This should be a rather short blast operation, because we only have a few sequences.

***Did you get any results at all?

### Let's try a probabilistic approach with HMMER (HMMER: biosequence analysis using profile hidden Markov models)

http://eddylab.org/software/hmmer/Userguide.pdf

HMMER is a program that uses hidden markov profiles to predict the probability that two proteins are related.

The first step is to give it a set of data for calculating probabilities.  

There are two common sources for reference data sets for HMMER:

1.) precomputed hmm profiles in public databases (e.g. Pfam from the Uniprot reference proteomes)
2.) multiple sequence alignments

We are going to do approach #2 first.

We have a set of data already that we can create our own hmm profile from, the MyD88 set of data.

##Install hmmer and muscle

```

conda install hmmer

```

Check the install of hmmer by typing the following and seeing if the instructions are provided.

```
hmmscan
```

## Install muscle

```
conda install muscle
```

Check the install by typing the following and checking for the instructions.

```
muscle
```

## Align sequences

First we have to align the sequences (as we did when we were selecting the perfect set for our analysis).  It is important to eyeball them at some point to make sure that they are not composed of non-overlapping, too long or too short sequences.  We did our original scan on uniprot.  This can also be done on NCBI and through the online version of muscle.  It is better to be more conservative in the number of sequences rather than trying to include **everything**.  Some databases contain mislabeled sequences that can confound your search for proteins.

```

muscle -in MyD88.fasta -out MyD88.msa

```

Take a look inside the file to see if the alignments make sense.  Kick out any sequences that are making the sequences less cohesive.

```
less -S MyD88.msa

```

Then, we are going to use hmmbuild to analyze the occurrence of each peptide in relation to the position in the aligned proteins.

```

hmmbuild MyD88.hmm MyD88.msa

```

The final step to create a mathematical matrix that shows the probability of each transition based on the data provided.

```

hmmpress MyD88.hmm

ls

```

The MyD88.htm is now a searchable hmm profile that we can use with our genome to find proteins which are probably related.

```

hmmscan --domtblout Bugula.MyD88.domtblout MyD88.hmm Bugula.pep

```

Look at the results in the Bugula.MyD88.domtblout

Are there any potential matches to our hmm model?

---

##Predict multiple genes simultaneously

It can take a lot of time to do multiple sequence alignments for individual genes, so we are going to switch to using a reference hmm profile to annotate all the orf's in our genome.

We are going to stick with proteins, so please be aware that many items in our genome will not be annotated.

```
hmmscan --domtblout Bugula.All.domtblout /home/data/rseas/Pfam-A.hmm Bugula.pep

```

This can be interrupted and the file can be read without hurting anything.  So let the program run for a little while then hit Control-C.

We can do the same using blast.

```

blastp -query Bugula.pep -db /home/data/rseas/uniprot_sprot.fasta  -max_target_seqs 1  -outfmt 6 -evalue 1e-5  > Bugula.swissprot.blastp.outfmt6
 
```

Also let this run for a few minutes and terminate early with Control-C

Let's take a look at our data and discuss a few things.

    1.) How much does the reference database influence our ability to annotate the Bugula genome?
    2.) Which approach was more sensitive - local alignments with BLAST or hmm profile search with HMMER?
    3.) What strategies might you use with these two tools to improve your annotations?

Thank you!
