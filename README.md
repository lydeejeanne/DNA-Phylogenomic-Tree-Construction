# DNA Phylogenomic Tree Construction
## General Workflow to generate a Phylogenomic tree by transfering orthofinder results (protien) to DNA before generating a maximum likelyhood tree
#### Orthofinder generates it's own trees, but they are based on protein sequences generated using annotation pipelines (prokka, RAST, etc). But for bacteria and other organisms, these may be inaccurate to what you're actually seeing in the process of evolution. This is a step-by-step tutorial on how to generate a Phylogenomic tree from orthologous proteins in their DNA form

### Generating Orthologs 
Annotate using your program of choice, this is for [prokka](https://github.com/tseemann/prokka)

`for f in *.fa; do b={$f%.fa}; prokka --outdir ${b} --prefix ${b} ${f}; done`

Use [orthofinder](https://github.com/davidemms/OrthoFinder) identify similar sequences

`orthofinder -f fasta_files -t 8`

### Getting Orthofinder results into DNA
Orthofinder will pop out the file, Single_Copy_Orthologue_Sequences, which contains the fasta files for everything you're working with. From this, you want to create an index file.

`for f in *.fa; do grep "^>" $f | sed 's/>//' > ${f}_index.txt; done`

You want to combine all of the annotated protein sequences in DNA form so you can transform it all back to DNA

`for f in *.ffn; do cat * > all_dna.seq`

#### Then you can use that to pull the sequences of interest out using [seqtk](https://github.com/lh3/seqtk)  

`for f in *_index.txt; do b=${f%_index.txt}; seqtk subseq all_dna.seq $f > ${b}.dna; done`

### Aligning all of the individual DNA sequences 
Then you want to align all of the sequences, this is using [mafft](https://mafft.cbrc.jp/alignment/software/)

`for f in *.dna; do b=${f%_index.txt}; mafft --auto --adjustdirection $f > ${b}.aln; done`

Once they are aligned, you need to take out the _R_ that will be added when mafft adjusts your sequences, then be sure to put your file name in there

`for f in *.aln; do b=${f%_index.txt}; sed "s/_R_//"; awk '/>/{sub(">","&"FILENAME"_");sub(/\.fasta/,x)}1' $f > names/$f; done`

### Generating concattonated genes from each sample
Now individual genes need to be separated back into their sample names I use the programs listed [here](https://bioinformatics.stackexchange.com/questions/2649/how-to-convert-fasta-file-to-tab-delimited-file/2658) and included in this repo

`./name.sh`

Then sort each newly generated fasta file

`for f in *.fasta; do b=${f%_index.txt}; awk 'BEGIN{RS=">"} NR>1 {gsub("\n", "\t"); print ">"$0}' $f | sort -t $'\t' -k1d,1| awk '{sub("\t", "\n"); gsub("\t", ""); print $0}' > ${b}_sorted.fa; done`

Need to take out all of the fasta names so you have one long sequences

`for f in *.fa; do b=${f%.fa} ; sed 's/^>.*$//' $f > $b.fa; done`

### Now to generate the tree
Put all of the newly aligned orthologous DNA sequences into a new file

`for f in *.fa; cat $f > all_aligned_sorted.txt`

Then make a maximum likelihood tree with bootstraps (bb); using this setting [iqtree](https://github.com/Cibiv/IQ-TREE) will assign it's own model

`iqtree -s all_aligned_sorted.txt -st DNA -m TEST -bb 1000 -alrt 1000 -nt AUTO`

Suggested viewing platform is FigTree [website here](http://tree.bio.ed.ac.uk/software/figtree/)

