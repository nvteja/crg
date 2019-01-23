# Steps:
1. Align reads vs GRCh37 reference with decoy
  * Create a project(=case=family) dir: `mkdir -p project/input`
  * Copy/symlink input file(s) to project/input:  project_sample.bam, or project_sample_1.fq.gz and project_sample_2.fq.gz
  * Create bcbio project: `crg.prepare_bcbio_run.sh project align_decoy`
  * Run bcbio project: `qsub ~/cre/bcbio.pbs -v project=project`\
	For multiple projects create list of projects in projects.txt and run `qsub -t 1-N ~/cre/bcbio.array.pbs`\ where N = number of projects.

2. Remove decoy reads: `qsub ~/cre/cre.bam.remove_decoy_reads.sh -v bam=$bam`. Keep original bam with decoy reads to store data.

3. Call small and structural variants
 	* Create a project dir: `mkdir -p project/input`
 	* Move bam file(s) from step1 to project/input: project_sample.bam 
 	* Create bcbio project: `crg.prepare_bcbio_run.sh project no_align`
	* Run bcbio:  `qsub ~/cre/bcbio.pbs -v project=project`

4. Clean up bcbio run: `qsub ~/cre/cre.sh -v family=<project>,cleanup=1,make_report=0,type=wgs`

5. Excel reports for small variants.
	* coding report: `qsub ~/cre/cre.sh -v family=project`
	* noncoding variants for gene panels: 
		- create a bed file for a set of genes with [genes.R](~/bioscripts/genes.R)
		- subset variants: `bedtools intersect --header -a project-ensemble.vcf.gz -b panel.bed > project.panel.vcf.gz`
		- strip annotations: `~/cre/cre.annotation.strip.sh`
		- annotate variants in panels and create gemini.db: `qsub ~/cre/cre.vcf2cre.sh -v original_vcf=project.panel.vcf.gz,project=project `
		- build report: `qsub ~/cre/cre.sh -f family=project,type=wgs`
	* noncoding variants for gene panels with flank
		- modify bed file, add 100k bp to each gene start and end: `cat panel.bed | awk -F "\t" '{print $1"\t"$2-100000"\t"$3+100000'
		- proceed as for noncoding small variant report
	* de-novo variants for trios

6. Excel reports for structural variants  
	* [Report columns](https://docs.google.com/document/d/1o870tr0rcshoae_VkG1ZOoWNSAmorCZlhHDpZuZogYE/edit?usp=sharing)



## AnnotSV
[AnnotSV](http://lbgi.fr/AnnotSV/) must be set up as apart of the local environment to generate family level reports. Users should set FeaturesOverlap and SVtoAnnOverlap to 50 in the configFile. Because these scripts group SV's which have a 50% recipricol overlap, annotation should follow a similar rule.

# Report columns:
- CHR
- POS
- GT
- SVTYPE
- SVLEN
- END
- SOURCES: which programs called the event
- NUM_SVTOOLS: how many programs supported the event
- GENES: genes overlapping the event (mostly one gene)
- ANN: raw annotation from VEP
- SVscores: [SVscore-github](https://github.com/lganel/SVScore), [SVscore-article](https://academic.oup.com/bioinformatics/article/33/7/1083/2748212)
  - SVSCOREMAX
  - SVSCORESUM
  - SVSCORETOP5
  - SVSCORETOP10
  - SVSCOREMEAN
- DGV: frequency in DGV, 8000 WGS

# Family report
crg.intersect_sv_reports.py is a script which groups then annotates structural variants across multiple samples. 

Grouping is useful when analyzing families as most structural variants should be similar and conserved across samples. The criteria for grouping is defined as a minimum of a 50% recipricol overlap with an arbitrary "reference" interval. This criteria gaurentees that grouped structural variants are of similar size and position.

DGV, DDD and OMIM columns are annotated by [AnnotSV](http://lbgi.fr/AnnotSV/) made by [Véronique Geoffroy](https://www.researchgate.net/profile/Veronique_Geoffroy2).

The script produces a CSV file which can be analyzed using spreadsheet software.

## Family report columns:
Includes all of the columns above, except SOURCES, NUM_SVTOOLS, SVTYPE and ANN, in addition to:
- N_SAMPLES: number of samples a reference interval overlaps with
- LONGEST_SVTYPE: SVTYPE taken from the longest overlapping SV
- EXONS_SPANNED: number of exons a SV affects
- DECIPHER_LINK: hyperlink to DECIPHER website for the reference interval
- DGV_GAIN_IDs
- DGV_GAIN_n_samples_with_SV
- DGV_GAIN_n_samples_tested
- DGV_GAIN_Frequency
- DGV_LOSS_IDs
- DGV_LOSS_n_samples_with_SV
- DGV_LOSS_n_samples_tested
- DGV_LOSS_Frequency
- DDD_SV
- DDD_DUP_n_samples_with_SV
- DDD_DUP_Frequency
- DDD_DEL_n_samples_with_SV
- DDD_DEL_Frequency
- OMIM: Gene annotation, format: {GENE MIM# INHERITANCE DESCRIPTION};
- synZ
- misZ
- pLI
- HGMD_GROSS_INSERTION: gross (>20bp) insertion events in this gene that have been observed in HGMD, format {GENE|DISEASE|TAG|DESCRIPTION|COMMENTS|JOURNAL|AUTHOR|YEAR|PMID}
- HGMD_GROSS_DUPLICATION: gross duplication events in this gene that have been observed in HGMD
- HGMD_GROSS_DELETION: gross deletion events in this gene that have been observed in HGMD
- HGMD_COMPLEX_VARIATION: complex variantions (combination of indels, translocations, SNP, fusions, inversions) in this gene that have been observed in HGMD
- SAMPLENAME: does this sample have an overlapping SV in it? (0,1)
- SAMPLENAME_details: what are the SV's in this sample which overlap with the reference?

## To generate a family level report:
```python crg.intersect_sv_reports.py -exon_bed=/path/to/protein_coding_genes.exons.fixed.sorted.bed -o=output_family_report_name.csv -i sample1.sv.csv sample2.sv.csv sample3.sv.csv```

## Use case: compared SV calls from TCAG (ERDS) to MetaSV
```
bcftools view -i 'ALT="<DEL>"' 159_CH0315.pass.vcf.gz | bcftools query -f '%CHROM\t%POS\t%INFO/END\n' -o 159.metasv.del.bed
cat 159_CH0315.erds+_db20171204_20180815_3213_annotated.tsv |  awk '$5 ~/DEL/{print $2"\t"$3"\t"$4}'  > 159.tcag.del.bed
bedtools intersect -a 159.tcag.dup.bed -b 159.metasv.dup.bed -f 0.5 -wo -r | wc -l
```