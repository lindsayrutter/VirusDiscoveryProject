# Virus Genes 

Annotation of genes (including viral genes) can be based on similarity to other known genes or based on gene features (_ab initio_ prediction).

## Ab initio gene predition

Several programs or search models exist to determine putative genes, such as `GeneMark` or `Prodigal`. As `GeneMark` is a licensed program, in this open-source project prodigal has been chosen. An example of prodigal command is as following:

```bash

bin/prodigal -i DRR128724.realign.local.fa -a prodigal/DRR128724.realign.local.prodigal.11.meta.faa -d prodigal/DRR128724.realign.local.prodigal.11.meta.fna -s prodigal/DRR128724.realign.local.prodigal.11.meta.txt -g 11 -o prodigal/DRR128724.realign.prodigal.11.meta.fa

```

In this command, `Prodigal` will search for putative ORFs and will extract the genes in nucleotides and aminoacids, using the 11^th genetic code. See results in GitHub folder `prodigal/` in this team-folder.

## Annotation by similarity

Annotation by similarity is an standard feature of most of the programs that are regularly used. These similarity searches can either be based on basic scoring matrices, such as `BLAST` or `DIAMOND` or using probabilistic models, such `hidden-Markov models (HMM)`. As the first type of search has already been conducted by other teams in early stages, this team will focus on HMM annotation.

The most extensivly used, and still gold standard program is `HMMer`. `HMMer` requires one or multiple models to scan one or multiple sequnces. Interestingly, it allows to annotate distant orthologs, ideal for the discovery of viral ORF in NGS dark matter.

Here 2 different databases or collections of HMM are used: pVOGs and RVDBs. Here there is an example of how they work:

```bash

hmmscan -o pVOGs/DRR128724.realign.local.prodigal.11.meta.out --tblout pVOGs/DRR128724.realign.local.prodigal.11.meta.tblout --cpu 32 /novel/databases/pVOGs/all_vogs.hmm prodigal/DRR128724.realign.local.prodigal.11.meta.faa &
hmmscan -o RVDB/DRR128724.realign.local.prodigal.11.meta.out --tblout RVDB/DRR128724.realign.local.prodigal.11.meta.tblout --cpu 32 /novel/databases/RVDB/U-RVDBv14.0-prot-new.hmm prodigal/DRR128724.realign.local.prodigal.11.meta.faa &

```
See results in GitHub folder `pVOGs/` or `RVDB/` in this team-folder for the corresponding results.


## Ab initio gene profiling and gene annotation with VIGA pipeline

For a real annotation of viral ORFs in a given NGS, it is important to use a combined strategy, or pipeline, of annotation, in order to obtain as much information as possible. In this open-source project, it was decided to use an already existing pipeline: De novo VIral Genome Annotator or [VIGA](https://www.biorxiv.org/content/early/2018/03/07/277509).

VIGA is an automated pipeline that performes ab initio ORF profiling with Prodigal and gene annotation with Diamond, Blast and HMMER. It allows to uses multiple databases (i.e. RefSeq, pVOG, RVDB, etc).

All programs can be used through a [Docker image](https://hub.docker.com/r/vimalkvn/viga/)

```bash

docker pull vimalkvn/viga # All programs pre-install

```
Databases need to be manually downloaded and formatted. Additionally, BLAST, Diamond, Infernal and Hmmer should be install for formatting the databases.

```bash
# Create all databases
mkdir databases
cd databases

## rfam
mkdir rfam
cd rfam
curl -O ftp://ftp.ebi.ac.uk/pub/databases/Rfam/CURRENT/Rfam.cm.gz &> /dev/null
gunzip Rfam.cm.gz
# Before formatting the database, delete all but the first 2 instances, as we don't needed it in our pipeline, but VIGA requires it to launch as mandatory.
cmpress Rfam.cm &> /dev/null
cd ..

## RefSeq Viral Proteins
curl -O ftp://ftp.ncbi.nlm.nih.gov/refseq/release/viral/viral.1.protein.faa.gz &> /dev/null
curl -O ftp://ftp.ncbi.nlm.nih.gov/refseq/release/viral/viral.2.protein.faa.gz &> /dev/null
curl -O ftp://ftp.ncbi.nlm.nih.gov/refseq/release/viral/viral.3.protein.faa.gz &> /dev/null
gunzip viral.1.protein.faa.gz
gunzip viral.2.protein.faa.gz
gunzip viral.3.protein.faa.gz
cat viral.1.protein.faa viral.2.protein.faa viral.3.protein.faa > refseq_viral_proteins.faa
rm viral.1.protein.faa viral.2.protein.faa viral.3.protein.faa

## RefSeq Viral Proteins for DIAMOND
mkdir RefSeq_Viral_DIAMOND
cd RefSeq_Viral_DIAMOND
cp ../refseq_viral_proteins.faa .
diamond makedb --in refseq_viral_proteins.faa -d refseq_viral_proteins &> /dev/null
rm refseq_viral_proteins.faa
cd ..

## RefSeq Viral Proteins for BLAST
mkdir RefSeq_Viral_BLAST
cd RefSeq_Viral_BLAST
cp ../refseq_viral_proteins.faa .
makeblastdb -in refseq_viral_proteins.faa -dbtype prot -out refseq_viral_proteins &> /dev/null
rm refseq_viral_proteins.faa
cd ..
rm refseq_viral_proteins.faa

## pVOG and RVDB formatting

mkdir pvogs
cd pvogs
curl -O http://dmk-brain.ecn.uiowa.edu/VOG/downloads/All/AllvogHMMprofiles.tar.gz &> /dev/null
tar zxvf AllvogHMMprofiles.tar.gz &> /dev/null
{ echo AllvogHMMprofiles/*.hmm | xargs cat; } > pvogs.hmm
rm -rf AllvogHMMprofiles

wget https://rvdb-prot.pasteur.fr/files/U-RVDBv14.0-prot.hmm.bz2 
bzip2 -dk U-RVDBv14.0-prot.hmm.bz2
hmmconvert U-RVDBv14.0-prot.hmm > U-RVDBv14.0-prot3.hmm

cat pvogs.hmm U-RVDBv14.0-prot3.hmm > pvogs.hmm
hmmpress -f pvogs.hmm
cd ../..

```

VIGA has some specificities to be met:

- `run-viga` wrapper, `modifiers.txt` target `.fasta` or any other input file should be both in the current working directory.

- VIGA output has to be generated in the same working directory.

- Blast and Diamond perform correctly when multithreaded (option `--cpu` in VIGA), but Hmmer does not. Hmmer perfomes better with a high-memory single node.
 
To scale its usage to multiple contig files, a threading-controller was coded, being called as follows in the working directoy where all fasta files and the viga-wrapper is located:

```bash

./director.sh

```

Additionally, `director.sh` will call `unmapvigaannotations.pl`, `genbankfeature.py` and `unmapvigaannotations2.pl`, in this order, expand pVOG info and extract the translated ORF annotated in the VIGA `.gbk` output.

Fig. 1: 

![alt figure](pictures/cpu_occupy_160.png)


All scripts should be located in the working directory. Scripts have `python2.X`, `biopython` and `pandas`, as dependencies.

Fig. 2: Example of dataset reduction though the pipeline

![alt figure](pictures/reduction.jpg)
