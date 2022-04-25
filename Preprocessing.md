## Preprocessing amplicon data

Activate QIIME2 [V2019.1](https://docs.qiime2.org/2019.1/) and run q2-import plugin to add raw data into one.qza file

## Data import

```
qiime tools import 
  --type 'SampleData[PairedEndSequencesWithQuality]' 
  --input-path <path to raw data> 
  --input-format CasavaOneEightSingleLanePerSampleDirFmt #Illumina fastq files setting
  --output-path demux-paired-end.qza
```

The `demux.paired-end.qza` output file contains all raw data located in the folder provided in `--input-path` flag. `.gza` is a comprassion format
file that can be transformed to `.qzv` format for visualization. To convert `.qza` into `.qzv` format, run the following command:

```
qiime demux summarize 
  --i-data demux-paired-end.qza 
  --o-visualization demux_from_raw.qzv
```

`demux_from_raw.qzv` file can be vizualized in [view.qiime2.org](https://view.qiime2.org) page. The file will show a boxplot with the quality score
from the forward and reverse reads for all sequence bases. Mean quality score above 20 can be considered good to setup a trunc-len value for denoising

## Data denoise

```
qiime dada2 denoise-paired 
  --i-demultiplexed-seqs demux-paired-end.qza 
  --p-trunc-len-f 300 #based on boxplot
  --p-trunc-len-r 284 #based on boxplot
  --p-n-threads 4 
  --o-table feature-table.qza
  --o-representative-sequences rep-seqs.qza
  --o-denoising-stats denoising-stats.qza
```

`dada 2 denoise-paired` plugin denoises paired-end sequences, dereplicating them and filtering chimeras using DADA2 [see Callahan et al. (2016)](https://pubmed.ncbi.nlm.nih.gov/27214047/)
output files from denoise include `feature-table.qza`, which contains sequencing depth data. `rep-seqs.qza`, containing amplicon sequence variants (ASVs)
"featureID", lenght and hyperlink to NCBI. `denoising-stat.qza` containing denoising stats.

## Data classifier

```
qiime feature-classifier classify-sklearn 
  --i-classifier <path to classifier> 
  --i-reads rep-seqs.qza 
  --o-classification taxonomy.qza
```

`feature-classifier` is a q2 plugin that supports variety of taxonomic classifications methods for amplicon sequences [see Bokulich et al. (2018)](https://microbiomejournal.biomedcentral.com/articles/10.1186/s40168-018-0470-z).
Here we used `classify-sklearn`, which uses a machine learning trained classifier using UNITE or SILVA databases (see q2 classifier training [tutorial](https://docs.qiime2.org/2019.1/tutorials/feature-classifier/)). 
The output file will be a table containing featureIDs and taxonomic annotation for each ASV.

## Data phylogeny

```
qiime phylogeny align-to-tree-mafft-fasttree 
  --i-sequences rep-seqs.qza 
  --o-alignment aligned-rep-seqs.qza 
  --o-masked-alignment masked-aligned-rep-seqs.qza 
  --o-tree unrooted-tree.qza 
  --o-rooted-tree rooted-tree.qza
```
Using `rep-seqs.qza` file, q2-phylogeny generates phylegeny tree, using the `align-to-tree-mafft-fasttree` pipeline, which create sequence aligment using MAFFT (see `align-to-tree-mafft-fasttree` [description](https://docs.qiime2.org/2019.1/plugins/available/phylogeny/align-to-tree-mafft-fasttree/)).
As output, q2-phylogeny will provide aligned sequence tables and both rooted and unrooted trees.

## Data preparation for R

After all steps are finished, `feature-table.qza`, `rooted-tree.qza` and `taxonomy.qza` files will be used to run the metagenomic analysis with R. Decompress all
three files and move `feature-table.biom`, `tree.nwk` and `taxonomy.tsv` to working directory

### Transfor feature-table.biom

Having q2 active, run:

```
biom convert -i feature-table.biom -o feature-table.txt --to-tsv
```

Then open output file and remove first line and '#' from the first colname.

