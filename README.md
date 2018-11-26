# ACAD 2018 Advanced Bioinformatics Workshop Day 2

### Learning objectives

In this exercise you learn how to

- find toy examples to work with,
- construct a graph using `vg construct`,
- visualize that graph using `vg view`,
- simulate NGS reads from a graph using `vg sim`,
- create an index structure needed for read mapping using `vg index`,
- map reads to the graph using `vg map`.

### Getting started

Make sure you have vg installed. It is already available on the course workstations. If you want to bulid it on your laptop, follow the instructions at the [vg homepage](https://github.com/vgteam/vg) (note that building vg and all submodules from source can take ~1h). In this exercise, you will use small toy examples from the `test` directory. So make sure you have checked out vg:

    git clone https://github.com/vgteam/vg.git

Now create a directory to work on for this tutorial:

    mkdir exercise1
    cd exercise1
    cp vg/test exercise1

In the ACAD18 course, we already have the test directory prepared for you in the system image.

    cd /home/ubuntu/workshop/2_Tue_Garrison/toy

If you run into trouble, you can clone and copy out the test directory fom the vg repo.

### Constructing and viewing your first graphs

Like many other toolkits, vg is comes with many different subcommands. First we will use `vg construct` to build our first graph. Run it without parameters to get information on its usage:

    vg construct

Let's construct a graph from just one sequence in file `tiny/tiny.fa`, which looks like this:

    >x
    CAAATAAGGCTTGGAAATTTTCTGGAGTTCTATTATATTCCAACTCTCTG

To construct a graph, run

    vg construct -r tiny/tiny.fa -m 32 >tiny.ref.vg

This will create a (very boring) graph that just consists of a linear chain of nodes, each with 32 characters.

The switch `-m` tells vg to put at most 32 characters into each graph node. (You might want to run it with different values and observe the different results.) To visualize a graph, you can use `vg view`. Per default, `vg view` will output a graph in [GFA](https://github.com/GFA-spec/GFA-spec/blob/master/GFA1.md) format. By adding `-j` or `-d`, you can generate [JSON](https://www.json.org/) or [DOT](https://www.graphviz.org/doc/info/lang.html) output.

    vg view tiny.ref.vg
    vg view -j tiny.ref.vg
    vg view -d tiny.ref.vg

To work with the JSON output the tool [jq](https://stedolan.github.io/jq/) comes in handy. To get all sequences in the graph, for instance, try

    vg view -j tiny.ref.vg | jq '.node[].sequence'

Next, we use graphviz to layout the graph representation in DOT format.

    vg view -d tiny.ref.vg | dot -Tpdf -o tiny.ref.pdf

View the PDF and compare it to the input sequence. Now vary the parameter passed to `-m` of `vg construct` and visualize the result.

Ok, let's build a new graph that has some variants built into it. First, take a look at at `tiny/tiny.vcf.gz`, which contains variants in (gzipped) [VCF](https://samtools.github.io/hts-specs/VCFv4.2.pdf) format.

    vg construct -r tiny/tiny.fa -v tiny/tiny.vcf.gz -m 32 >tiny.vg

Visualize the outcome.  

Ok, that's nice, but you might wonder which sequence of nodes actually corresponds to the sequence (`tiny.fa`) you started from? To keep track of that, vg adds a **path** to the graph. Let's add this path to the visualization.

    vg view -dp tiny.vg | dot -Tpdf -o tiny.pdf

You find the output too crowded? Option `-S` removes the sequence labels and only plots node IDs.

    vg view -dpS tiny.vg | dot -Tpdf -o tiny.pdf

Another tool that comes with the graphviz package is *Neato*. It creates force-directed layouts of a graph.

    vg view -dpS tiny.vg | neato -Tpdf -o tiny.pdf

For these small graphs, the difference it not that big, but for more involved cases, these layouts can be much easier to read.

A better solution for larger graphs is [Bandage](http://rrwick.github.io/Bandage/), which is designed for visualizing assembly graphs. It can't display path information, but it can scale to multi-megabase graphs with ease. Adjust the `--nodewidth` parameter if your display is blank.

    vg view tiny.vg >x.gfa
    Bandage image x.gfa x.gfa.png --nodewidth 100

You can also apply `vg viz` to obtain a special kind of linear layout that scales well without losing path information.
It requires the `xg` index, which we'll build first.

    vg index -x tiny.xg tiny.vg
    vg viz -x tiny.xg -o tiny.svg

If the graph is very big, you'll need to view tiny.svg in chrome or chromium web browser.

### Mapping reads to a graph
Ok, let's step up to a slightly bigger example.

    ln -s ../vg/test/1mb1kgp

This directory contains 1Mbp of 1000 Genomes data for chr20:1000000-2000000. As for the tiny example, let's' build one linear graph that only contains the reference sequence and one graph that additionally encodes the known sequence variation. The reference sequence is contained in `1mb1kgp/z.fa`, and the variation is contained in `1mb1kgp/z.vcf.gz`. Make a reference-only graph named `ref.vg`, and a graph with variation named `z.vg`. Look at the previous examples to figure out the command.

You might be tempted to visualize these graphs (and of course you are welcome to try), but they are sufficiently big already that neato can run out of memory and crash.

In a nutshell, mapping reads to a graph is done in two stages: first, seed hits are identified and then a sequence-to-graph alignment is performed for each individual read. Seed finding hence allows vg to spot candidate regions in the graph to which a given read can map potentially map to. To this end, we need an index. In fact, vg needs two different representations of a graph for read mapping XG (a succinct representation of the graph) and GCSA (a k-mer based index). To create these representations, we use `vg index` as follows.

    vg index -x z.xg z.vg
    vg index -g z.gcsa -k 16 z.vg

Passing option `-k 16` tells vg to use a k-mer size of *16k*. The best choice of *k* will depend on your graph and will lead to different trade-offs of sensitivity and runtime during read mapping.

As mentioned above, the whole graph is unwieldy to visualize. But thanks to the XG representation, we can now quickly **find** individual pieces of the graph. Let's extract the vicinity of the node with ID 2401 and create a PDF.

    vg find -n 2401 -x z.xg -c 10 | vg view -dp - | dot -Tpdf -o 2401c10.pdf

The option `-c 10` tells `vg find` to include a context of 10 nodes in either direction around node 2401. You are welcome to experiment with different parameter to `vg find` to pull out pieces of the graph you are interested in.  

Next, we want to play with mapping reads to the graph. Luckily, vg comes with subcommand to simulate reads off off the graph. That's done like this:

    vg sim -x z.xg -l 100 -n 1000 -e 0.01 -i 0.005 -a >z.sim

This generates 1000 (`-n`) reads of length (`-l`) with a substitution error rate of 1% (`-e`) and an indel error rate of 0.5% (`-i`). Adding `-a` instructs `vg sim` to output the true alignment paths in GAM format rather than just the plain sequences. Map can work on raw sequences (`-s` for a single sequence or `-r` for a text file with each sequence on a new line), FASTQ (`-f`), or FASTA (`-f` for two-line format and `-F` for a reference sequence where each sequence is over multiple lines).

We are now ready to map the simulated read to the graph.

    vg map -x z.xg -g z.gcsa -G z.sim >z.gam

We can visualize alignments using an option to `vg view`. The format is not pretty but it provides us enough information to understand the whole alignment.
More advanced visualization methods (like [IVG](https://vgteam.github.io/sequenceTubeMap/)) are in development, but do not work on the command line.

These commands would show us the first alignment in the set:

    vg view -a z.gam | head -1 | vg view -JaG - >first_aln.gam
    vg find -x z.xg -G first_aln.gam | vg view -dA first_aln.gam - | dot -Tpdf -o first_aln.pdf

We see the `Mappings` of the `Alignment` written in blue for exact matches and yellow for mismatches above the nodes that they refer to. Many alignments can be visualized at the same time. A simpler mode of visualization `vg view -dSA` gives us the alignment's mappings to nodes, colored in the range from green to red depending on the quality of the match to the particular node.

For evaluation purposes, vg has the capability to compare the newly created read alignments to true paths of each reads used during simulation.

    vg map -x z.xg -g z.gcsa -G z.sim --compare -j

This outputs the comparison between mapped and and true locations in JSON format. We can use this quickly check if our alignment process is doing what we expect on the variation graph we're working on. For instance, we could set alignment parameters that cause problems for our alignment and then observe this using the `--compare` feature of the mapper. For example, we can map two ways and see a difference in how correct our alignment is:

    vg map -x z.xg -g z.gcsa -G z.sim --compare -j | jq .correct | sed s/null/0/ | awk '{i+=$1; n+=1} END {print i/n}'

In contrast, if we were to set a very high minimum match length we would throw away a lot of the information we need to make good mappings, resulting in a low correctness metric:

    vg map -k 51 -x z.xg -g z.gcsa -G z.sim --compare -j | jq .correct | sed s/null/0/ | awk '{i+=$1; n+=1} END {print i/n}'

It is essential to understand that our alignment process works against the graph which we have constructed. This pattern allows us to quickly understand if the particular graph and configuration of the mapper produce sensible results at least given a simulated alignment set. Note that the alignment comparison will break down if we simulate from different graphs, as it depends on the coordinate system of the given graph.

### Exploring the benefits of graphs for read mapping

To get a first impression of how a graph reference helps us do a better job while mapping reads. We will construct a series of graphs from a linear reference to a graph with a lot variation and look at mapping rates, i.e. at the fraction of reads that can successfully be mapped to the graph. For examples, we might include variation above given allele frequency (AF) cutoffs and vary this cutoff. You can make a VCF with a minimum allele fequency with this command (replace `FREQ` with the frequency you want):

    vcffilter -f 'AF > FREQ' 1mb1kgp/z.vcf.gz > min_af_filtered.vcf

Alternatively, you can also use `bcftools` to subset the VCFs. The ``--exculde`` option in conjunction with custom [expressions](https://samtools.github.io/bcftools/bcftools-man.html#expressions) is particularly useful to this end. You may also want to think about other properties that would be useful to filter on.

With this input you should be able to run the whole pipeline:

- Construct the graphs with the filtered VCF
- Index the graphs for mapping
- Map the simulated reads to the graphs
- Check the identity of the read mappings

Try doing this on graphs with a range of minimum allele frequencies (e.g. 0.5, 0.1, 0.01, etc.). How do the properties of the mappings change with different minimum frequencies? Let's also look at the sizes of the graphs and indexes.

    ls -sh *.vg
    ls -sh *.gcsa*

How do these files seem to scale with the minimum cutoff?

### Mapping data from real data to examine the improvement

We can also download some real data mapping to this region to see if the different graphs provide varying levels of performance.

    # note: if you are taking this course, this data is available in /home/ubuntu/workshop/2_Tue_Garrison/human
    samtools view -b ftp://ftp-trace.ncbi.nlm.nih.gov/giab/ftp/data/NA12878/NIST_NA12878_HG001_HiSeq_300x/RMNISTHS_30xdownsample.bam 20:1000000-2000000 >NA12878.20_1M-2M.30x.bam
    # alternatively:
    wget http://hypervolu.me/~erik/tmp/HG002-NA24385-20_1M-2M-50x.bam

The first is a sample that was used in the preparation of the 1000 Genomes results, and so we expect to find it in the graph. The second wasn't used in the preparation of the variant set, but we do expect to find almost all of its variants in the 1000G set. Why is this true?

We can run a single-ended alignment test to compare with bwa mem:

    samtools fastq -1 HG002-NA24385-20_1M-2M-50x_1.fq.gz -2 HG002-NA24385-20_1M-2M-50x_2.fq.gz data/HG002-NA24385-20_1M-2M-50x.bam
    bwa mem z.fa HG002-NA24385-20_1M-2M-50x_1.fq.gz | sambamba view -S -f json /dev/stdin | jq -cr '[.qname, .tags.AS] | @tsv' >bwa_mem.scores.tsv
    vg map --drop-full-l-bonus -d z.AF0.01 -f HG002-NA24385-20_1M-2M-50x_1.fq.gz -j | pv -l | jq -cr '[.name, .score] | @tsv' >vg_map.AF0.01.scores.tsv

Then we can compare the results using sort and join:

    join <(sort bwa_mem.scores.tsv ) <(sort vg_map.AF0.01.scores.tsv ) | awk '{ print $0, $3-$2 }' | tr ' ' '\t' | sort -n -k 4 | pv -l | gzip >compared.tsv.gz

We can then see how many alignments have improved or worse scores:

    zcat compared.tsv.gz | awk '{ if ($4 < 0) print $1 }' | wc -l
    zcat compared.tsv.gz | awk '{ if ($4 == 0) print $1 }' | wc -l
    zcat compared.tsv.gz | awk '{ if ($4 > 0) print $1 }' | wc -l

In general, the scores improve. Try plotting a histogram of the differences to see the extent of the effect.

We can pick a subset of reads with high or low score differentiation to realign and compare:

    zcat HG002-NA24385-20_1M-2M-50x_1.fq.gz | awk '{ printf("%s",$0); n++; if(n%4==0) { printf("\n");} else { printf("\t\t");} }' | grep -Ff <(zcat compared.tsv.gz | awk '{ if ($4 < -10) print $1 }' ) | sed 's/\t\t/\n/g' | gzip >worse.fq.gz
    zcat HG002-NA24385-20_1M-2M-50x_1.fq.gz | awk '{ printf("%s",$0); n++; if(n%4==0) { printf("\n");} else { printf("\t\t");} }' | grep -Ff <(zcat compared.tsv.gz | awk '{ if ($4 > 10) print $1 }' ) | sed 's/\t\t/\n/g' | gzip >better.fq.gz

Let's dig into some of the more-highly differentiated reads to understand why vg is providing a better (or worse) alignment. How might you go about this? There are many ways you could do this, but you may find some of these commands useful:

- `vg view -aj ALN.gam` : convert a .gam alignment into a text-based JSON representation with one alignment per line
- `vg view -aJG ALN.json` : convert the JSON representation back into a .gam alignment file
- `vg mod -g ID -x N GRAPH.vg` : extract the subgraph that is within `N` nodes from node `ID`
- `vg mod -P -i ALN.gam GRAPH.vg` : add the paths from the alignment into the graph (similar to the reference path in the exercise)

### Bonus: building a variation graph from whole genomes

The directory `~/workshop/2_Tue_Garrison/yeast` contains _S. cerevisiae_ data from [Yue et. al 2017](https://doi.org/10.1038/ng.3847).
This paper produced whole genome assemblies for a number of yeast strains based on PacBio long read sequencing, and then used these assemblies to examine evolutionary adaptation in wild and domesticated yeasts.
Here, your goal is to use the high-performance long read and whole genome aligner [minimap2](https://github.com/lh3/minimap2), the alignment filtering system [fpa](https://github.com/natir/fpa), and the variation graph inducer [seqwish](https://github.com/ekg/seqwish) to build a graph from these whole genome assemblies.
You can then use `Bandage` to visualize the output.
By indexing the graph as before, we can then test the relative performance of alignment with the Illumina data from the S288C and SK1 strains.

To build the graph, we first apply [minimap2 for whole genome alignment as directed by its author](https://github.com/lh3/minimap2/issues/251):

    cd genomes
    pan-minimap2 S288c.genome.fa DBVPG6765.genome.fa UWOPS034614.genome.fa Y12.genome.fa YPS128.genome.fa SK1.genome.fa DBVPG6044.genome.fa >cerevisae.paf

We filter the alignments to remove short alignments that are typical of repeats, and which can cause collapse of the induced graph.

    cat cerevisiae.paf | pv -l -c  | fpa -l 10000 | pv -l -c >cerevisiae.l10k.paf

Now we can run seqwish to induce the variation graph. Note that `cerevisiae.fa` contains the above genomes concatenated together.

    seqwish -s cerevisiae.fa -a cerevisiae.l10k.paf -b cerevisiae.l10k.paf -t 4 >cerevisiae.l10k.gfa

Bandage will require a lot of memory, but can provide an interesting view of the result:

    cat cerevisiae.l10k.gfa | grep -v ^P >cichlid_pan.l10k.no_paths.gfa
    Bandage image cichlid_pan.l10k.no_paths.gfa cichlid_pan.l10k.gfa.png --nodewidth 1000 --width 4000 

Now we should convert the graph to .vg format and [index it as described in the vg wiki](https://github.com/vgteam/vg/wiki/Index-Construction).
It is very likely that we will require some form of pruning to reduce the graph complexity, so follow suggestions there to do so.

Finally, to evaluate the benefit of aligning against this graph, we should build a linear graph from the S288C.genome.fa file.
Align the SK1 read set to both graphs and measure the relative alignment identity.
Does the aggregate alignment quality improve when aligning against the pangenome?
Do the reads have higher identity when aligned to the pangenome versus the linear reference?
Is the effect the same for S288C?
What happens when we remove SK1 or other genomes from the input set and repeat the process?
