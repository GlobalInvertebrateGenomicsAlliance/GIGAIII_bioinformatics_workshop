
# Gene prediction, the shortcut approach
  
## Assumptions

1. You have a access to a Linux environment (possible in others as well but might need some troubleshooting)
2. You have a genome assembled in a fasta format with an adequate N50 or at least few long enough scaffolds
3. You are somewhat comfortable running things in the terminal using BASH/ZSH/FISH etc.
4. Refer elsewhere on how to get to 1-3

## Environment setup

- Login into server environment (Jetstream , cluster, or your own *nix setup)
- Setup Conda (mentioned [here](https://gigaiii-bioinformatics-workshop.readthedocs.io/en/latest/)) then bioconda <https://bioconda.github.io/>
- Install Busco this should install many packages & will take care of all the dependencies needed to predict genes such as augustus, hmmer, and blast.
- Note that currently (Circa. Sept 2018) the default settings from bioconda will likely fail to install a working version of AUGUSTUS on  Mac OS , see [here]( https://github.com/nextgenusfs/funannotate/issues/3) and [here](https://github.com/nextgenusfs/funannotate/issues/194). If you have no option at all to access a Linux environment you might want to look into running a Virtual machine using [Virtualbox](https://www.virtualbox.org/) or using [DOCKER](https://hub.docker.com/r/robsyme/busco/), [VAGRANT](https://www.vagrantup.com/). On windows you can use the [WSL](https://docs.microsoft.com/en-us/windows/wsl/install-win10) environment to run BUSCO.  It is not impossible to get BUSCO & AUGUSTUS to work natively on Windows or Mac Os if you spend many hours to downgrade/install specific compilers and compile augustus yourself.

To install BUSCO which includes augustus run the following:

```bash
 conda install busco
 ```

This should churn for a while, depending on how many other dependencies you already have on your system.

## Files needed

Download a small genome for the sake of this tutorial such as C.elegans

```bash
wget ftp://ftp.ncbi.nlm.nih.gov/genomes/all/GCF/000/002/985/GCF_000002985.6_WBcel235/GCF_000002985.6_WBcel235_genomic.fna.gz
```

You can substitute your own genome as well. This FASTA file should be either full chromosomes, scaffolds or contigs, depending on the current stage of your assembly. The quality of your predictions will depend on the quality of your assembly.

gunzip your file if your file is gzipped

## Create files and folders

Create a directory called Celegans_Euk_ODB9 or your favorite name

```bash
mkdir Celegans_Euk_ODB9
```
## Run BUSCO
cd into the directory you created
run the following command
Type

```bash
run_busco -h
```

Examine the output and see what are the parameters needed to run busco

Download the needed dataset to predict in your particular genome, you should check the busco website in case the link below gets outdated as new ODB releases are made <https://busco.ezlab.org/>

I recommend starting with the eukaryota dataset first then moving up the tree.

```bash
# Eukaryota
wget https://busco.ezlab.org/datasets/eukaryota_odb9.tar.gz
# Metazoa
wget http://busco.ezlab.org/v2/datasets/metazoa_odb9.tar.gz

```

Extract using tar/gzip the corresponding file, it will create a directory with the name of the file minus the ".tar.gz"

```bash
tar -xvf eukaryota_odb9.tar.gz
```

Before running Busco OR augustus you need to configure the AUGUSTUS_CONFIG_PATH other wise you will see the following error

```
ERROR   The environment variable AUGUSTUS_CONFIG_PATH is not set
```

The AUGUSTUS_CONFIG_PATH will depend on how installed BUSCO/conda. In most cases it will be located in ~/miniconda3/config - if you used miniconda as your setup. In the Jetstream image we use, miniconda is installed in /opt/miniconda , thus the config folder will be /opt/miniconda3/config. However if you have other installations of augustus outside of conda you might have to specify that location. To know that you have found the correct Augustus config folder it will contain  these folders/files when you run ls inside it.

```bash
ls

cgp  config.ini  extrinsic  model  profile  species
```

To set the path simply use the export command as follows - adjust based on where you have miniconda/ augustus installed.

```bash
export AUGUSTUS_CONFIG_PATH=/opt/miniconda3/config
```

Now busco can run without complaining, note that in the future Bioconda might set this path automatically during installation but so far it has to be set manually. In certain cluster environments where augustus is already installed, it might be in a non- writeable location - consult with your sys admin in this situation. Or download a local version of AUGUSTUS and use the config folder of that.

```bash
# Run Busco
run_busco -i genome.fasta -c 6 -l eukaryota_odb9 -o genome_euk -m genome
```

What are these options?

- -i ,  your input fasta
- -c , number of cores, adjust accordingly
- -l , where you extracted your busco ortholog set e.g. eukaryota_odb9
- -o , where to store the output of the run
- -m , mode to run in our case genome -

The error below occurs sometimes, the solution is to run with less or no threads

```
ERROR   tblastn has ended prematurely (the result file lacks the expected final line), which will produce incomplete results in the next steps ! This problem likely appeared in blast+ 2.4 and seems not fully fixed in 2.6. It happens only when using multiple cores. You can use a single core (-c 1) or downgrade to blast+ 2.2.x, a safe choice regarding this issue. See blast+ documentation for more information.
```

## Training Augustus

Once Busco is done, you will be able to see the completeness of your genome / dataset. This is useful to compare your different assemblies and to other genomes 
Go to the output directory you specified and examine it.
You will notice that Busco added the "run" prefix to your output directory, within that locate the directory called augustus_output

Locate the file named training_set_uniquename where "uniquename" is the one your chose with the -o option

This file has the extension .txt , however it is really a Genbank format file, which is what we need to train AUGUSTUS

The first step is to create a new species, which will create default parameters for gene prediction.
The goal is to optimize these parameters with training.

There are many ways you can train augustus, and you should refer to the augustus [manual](augustus.gobics.de) for more comprehensive overview of the training options.

To create a new species use the following commands, I usually add a postfix/prefix to the species name to be able to tell apart how different models have been trained and then compare their performance in the end. The different kinds of training will make a model biased more towards false negatives or false positives, in certain circumstances you might be inclined to favor one over the other. Let's say you are desperate to find if a species contains a particular gene, thus a high false positive rate might not be your biggest concern as a high false negative rate.

```bash
# Create new Species
new_species.pl --species=C_elegansBUSCO

```

This will create a new directory within the config directory pointed by the environment variable AUGUSTUS_CONFIG_PATH , we set earlier

Create training and test set.

If you ran multiple stages of busco, i.e. using the Eukaryotic dataset, Metazoan dataset, other orthogroups. You can concatenate your training sets, in some cases specially with chromosome level assemblies you might end up with duplicated LOCUS ids which causes problems for the random_split.pl script, use the perl onliner below to rename & make unique Locus Ids in the genbank file as a workaround.

To create a training and testing dataset from one genbank file use the `randomSplit.pl` script that comes with Augustus, if installed with conda, it should be in the path already

First count how many genes are in the training dataset by using - which counts how many "//" is in the Genbank file which is the record delimiter for genbank files

training_set.txt will differ based on how you named your busco run
```bash
grep -c "//" training_set.txt

```

You want to use a reasonable portion for training,
you might want around 50% for training and 50% for testing, however if you have a really huge dataset, you probably don't want to use more than 1000 features for training to avoid overfitting the model.
So if the number you get is around 800, then use 400 for training.

The command to split the file into a training and testing set is:

```bash
# change 400 to 50% of your trainingset
randomSplit.pl trainingset.gb 400
```

If you get this error

```

ERROR in randomSplit.pl line 47: LOCUS names in genbank file are not unique!

```

Use the following trick to make the Genbank file unique

```bash
 cat  TrainingSet.gb  | perl -pe 's/LOCUS\s*\S*/"LOCUS      " . ($. + 1)/ge' > UniqueSet.gb
```

This randomSplit.pl will create 2 new files based on the file you provided. They will contain randomly assigned sequences. They will have the extension .train & .test

### Training augustus

Now we can start training  augustus with the training set
to do this run the following:

```bash
etraining --species=C_elegansBUSCO TrainingSet.gb.train
```

This happens very fast!
Now we can actually assess how well did the trained model perform?
to do this we will actually run a prediction on our test set, to do this we run

```bash
augustus --species=C_elegansBUSCO genes.gb.test > genes.gb.test.predictions
```

Now examine the output using less, what you are looking at is an extended GFF format, which essentially describes locations in the genome and where the genes are detected

At the end of the file you will see some important statistics about how well the predictor performed

Take a look at the sensitivity vs specificity.

According the augustus manual if your sensitivity is lower than 20% then your training did not go too well, you might want to try more training data, or a different training strategy

Before running the predictions, you need to fix one issue in one of the augustus scripts as a result of the conda installation. Locate the bin folder in your miniconda folder, then locate the autoAugPred.pl script using the editor nano , use ctrl + w to search for a keyword find the line that says

```perl
            my $aug="$AUGUSTUS_CONFIG_PATH/../src/augustus";

```

Change it to

```perl
            my $aug="$AUGUSTUS_CONFIG_PATH/../bin/augustus";
```

Run the `autoAugPred.pl` script, the script is mainly designed to split your genome into chunks and run predictions sequentially or in parallel and then merge. As input it will use your genome fasta file, and your trained species name. Also if you have made RNA-Seq based hints gff file it can also be used at this stage.
Substitute the species with the name you chose, and the genome as well with the correct genome filename

```bash
autoAugPred.pl --genome=GCF_000002985.6_WBcel235_genomic.fna --species=C_elegansBUSCO2
```

The program will create a directory with this structure

```
autoAugPred_abinitio
|   gbrowse
|   predictions
|   shells
|   |__aug1
|   |__aug2
|   |__....
|   |__augN
|   |__shellForAug
```

To actually run the predictions you will need to cd into shells folder.

Each of the aug* files is a command to run augustus on a chunk of the genome fasta. You can examine them using less
However, since the script `shellForAug` produces commands suitable for a PBS based cluster environment, you might want to utilize the cores on your machine in a more efficient way if you are in a shared memory system with many cores rather than a batch system.
To work around this, run the following command, you must have the GNU parallel installed, it can also be installed using conda.

```bash

(find -name "aug*"  -type f | parallel -j1 cat {}';'echo ) > AllCommands

```

When you run this command inside the shells folder will produce the AllCommands file simply after that execute all the jobs in parallel using GNU parallel

```bash
parallel -j < AllCommands
```

You can restrict the number of cores using the -j parameter but parallel should be able to optimize the execution based on free cores.

Once all the jobs are done ~ 40-45 minutes depending on the size of your genome and how many cores you have!

Re-run the autoAugPred.pl script with the --continue switch from the same location with the same parameters you ran the first time, the script now will finish the prediction step by merging all the produced GFF files and generating cleaned up single GFF, GTF, amino acid fasta file for your gene predictions.

Now you should have your predictions ready in the predictions folder. you should see `augustus.gff` , `augustus.gtf`, `augustus.aa`

## What to do next

- Run busco in protein mode on your predicted proteins, is it worse or better than running busco on the genome?

- Retrain using other datasets and compare

- Try with predictions with RNA-Seq data and without it
- Load your gff/gtf file in IGV together with your raw RNA-Seq mapped data? how well do the gene models match with your RNA-Seq data?
- Take the `augustus.aa` file and use blastp and hmmer to annotate it and find function of your predicted proteins
- Are you doing some popgen things? use the gff/gtf file to filter exon and intron regions and intersect with your mapped short reads.
- Use the predicted proteins to train other gene predictors for other species!
- Combine with other predictions using MAKER and other tools

## Optimizing augustus

This step is time intensive and might run for days, weeks, months. So only run when are ready to do it. Also in many cases the optimized model does not perform much better than the non-optimized ( you won't know unless you try)

to optimize the augustus parameters use

```bash
optimize_augustus.pl --species=C_elegansBUSCO TrainingSet.gb.train

```
You can run optimize_augustus.pl using more cpus, however make sure the perl dependancy `Parallel::ForkManager` is installed before you utilize the multiple CPUs

```bash
optimize_augustus.pl --cpus=6 --species=C_elegansBUSCO TrainingSet.gb.train
```

After the `optimize_augustus.pl` is finished re-run the `etraining` command again to utilize the new augustus metaparameters

## Additional Resources

- <https://www.gensas.org/> Lets you run many genome annotation piplines online, without installing anything, assembly must be in a good shape before trying it.
- <http://www.yandell-lab.org/software/maker.html> Maker will allow you to combine the results of many gene predictions
- <http://arthropods.eugenes.org/EvidentialGene/> Allows you to combine RNA-Seq data with gene predictions to get better gene models
- <https://funannotate.readthedocs.io/en/latest/> encapsulates many gene prediction tools and easy to use, originally for fungi but scales well

## Reading material

Yandell, Mark, and Daniel Ence. "A beginner's guide to eukaryotic genome annotation." Nature Reviews Genetics 13.5 (2012): 329.

Katharina J. Hoff, Simone Lange, Alexandre Lomsadze, Mark Borodovsky, Mario Stanke; BRAKER1: Unsupervised RNA-Seq-Based Genome Annotation with GeneMark-ET and AUGUSTUS, Bioinformatics, Volume 32, Issue 5, 1 March 2016, Pages 767–769, https://doi.org/10.1093/bioinformatics/btv661

Hosmani, Prashant S., et al. "A quick guide for student-driven community genome annotation." arXiv preprint arXiv:1805.03602 (2018).

Stanke, Mario, and Stephan Waack. "Gene prediction with a hidden Markov model and a new intron submodel." Bioinformatics 19.suppl_2 (2003): ii215-ii225.

Pevzner, Pavel. Computational molecular biology: an algorithmic approach. MIT press, 2000.

http://www.geneprediction.org/
http://www.geneprediction.org/books.html

Eddy, Sean R. "What is a hidden Markov model?." Nature biotechnology 22.10 (2004): 1315.

https://www.coursesource.org/courses/a-hands-on-introduction-to-hidden-markov-models

