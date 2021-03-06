# Sequence to Medical Phenotypes (STMP)
A pipeline featuring variant annotation, prioritization, pharmacogenomics, and tools for analyzing genomic trios (mother, father, child).

**Release versions can be downloaded from [https://github.com/AshleyLab/stmp/releases](https://github.com/AshleyLab/stmp/releases), or you can clone this repository to download the latest version of the code.**

The toolkit currently uses an SQLite database for added portability.

Contents
=================

  * [Sequence to Medical Phenotypes (STMP)](#sequence-to-medical-phenotypes-stmp)
  * [Contents](#contents)
    * [Dependencies](#dependencies)
      * [External Dependencies (to be placed in the third\_party folder \-\- see instructions below)](#external-dependencies-to-be-placed-in-the-third_party-folder----see-instructions-below)
      * [Other dependencies (these must be in the user or system PATH before running STMP)](#other-dependencies-these-must-be-in-the-user-or-system-path-before-running-stmp)
      * [Python dependencies](#python-dependencies)
    * [Installation Instructions](#installation-instructions)
      * [Downloading software and dependencies](#downloading-software-and-dependencies)
      * [Setting up STMP](#setting-up-stmp)
      * [Testing STMP](#testing-stmp)
    * [Running STMP](#running-stmp)
      * [1) Annotation](#1-annotation)
      * [2) Tiering](#2-tiering)
      * [3) Pharmacogenomics (pgx)](#3-pharmacogenomics-pgx)
      * [4) Trio (separate script)](#4-trio-separate-script)
    * [Customization](#customization)
    * [Acknowledgement](#acknowledgement)
    * [Appendices](#appendices)
      * [Appendix 1: List of ANNOVAR datasets to download for functional annotation](#appendix-1-list-of-annovar-datasets-to-download-for-functional-annotation)
      * [Appendix 2: List of ANNOVAR datasets to download for trio tools](#appendix-2-list-of-annovar-datasets-to-download-for-trio-tools)

Created by [gh-md-toc](https://github.com/ekalinin/github-markdown-toc.go)


## Dependencies

To date, STMP has been tested on Python 2.7.6. Other versions of Python may also work, but are not officially supported.

### External Dependencies (to be placed in the `third_party` folder -- see instructions below)

- ANNOVAR version 2015-03-22 15:29:59 (Sun, 22 Mar 2015)
- snpEFF version 4.1e (build 2015-05-02)

Other versions of the above tools may also work but are not currently supported.

### Other dependencies (these must be in the user or system PATH before running STMP)
- bcftools version 1.2
- bedtools version 2.25.0

### Python dependencies
- Pyyaml version 3.11
- xlwt version 1.0.0 (for exporting results to an Excel file)

-----------------------------------------------------------
## Installation Instructions

### Downloading software and dependencies
- Download the latest STMP release from [here](https://github.com/AshleyLab/stmp/releases), or clone this repository.
- Download [ANNOVAR](http://annovar.openbioinformatics.org/en/latest/user-guide/download/) and [snpEFF](http://sourceforge.net/projects/snpeff/files/snpEff_latest_core.zip/download) and make sure they are copied/symlinked in a folder called `annovar` and `snpeff` within the `third_party` folder.
    - E.g. ANNOVAR would be linked/copied to `third_party/annovar` (this folder should contain all files from the ANNOVAR download, including `annotate_variation.pl` and `table_annovar.pl`)
    - E.g. snpeff would be linked/copied as `third_party/snpeff/snpEff` (this folder should include files such as `snpEff.jar`)
- Ensure Pyyaml is installed (via pip install, etc.)
- Ensure the appropriate versions of bedtools and bcftools (above) are installed and in the user/system PATH. These can be either downloaded directly from the corresponding websites or installed via a program such as bcbio.
- Run the appropriate ANNOVAR command to download the datasets specified in Appendix 1 (e.g. `annotate_variation.pl -buildver hg19 -downdb -webfrom annovar refGene humandb/` from within `third_party/annovar` to download the refGene dataset). 
    - **NOTE: `hg19_wgEncodeGencodeBasicV19Mrna.fa` is no longer provided by Annovar/UCSC and must instead be downloaded manually from `https://www.dropbox.com/s/icw1loscvpm6v84/hg19_wgEncodeGencodeBasicV19Mrna.fa?dl=0` and copied to the annovar/humandb folder. Without this file, `Annovar_ExonicFunc_wgEncodeGencodeBasicV19` and `Annovar_AAChange_wgEncodeGencodeBasicV19` will not show up correctly in the annotated output.**

- If you would like to run trio tools: 
   - Copy `code/annovar/summarize_annovarRDv2.pl` to `third_party/annovar`
   - Run the appropriate ANNOVAR command to download the datasets specified in Appendix 2 (e.g. `annotate_variation.pl -buildver hg19 -downdb -webfrom annovar refGene humandb/` from within `third_party/annovar` to download the refGene dataset).


### Setting up STMP
- Run `python stmp.py --update_db`. This will create a SQLite database file in the db folder and download and import the core datasets required for annotation and tiering.

### Testing STMP
- For a basic test of whether STMP has been installed and configured correctly, run `python stmp.py --test`. This will run a small VCF file (a subset of Genome in a Bottle sequence variants) through STMP annotation, tiering, and pgx. Output will be stored in `(stmp_dir)/data/test/output_unverified` by default, or can be stored in a different directory with the `--output_dir` parameter. Compare the output against our verified output `(stmp_dir)/data/test/output_verified` to see if it is the same, e.g. `diff -rq (stmp_dir)/data/test/output_unverified/ (stmp_dir)/data/test/output_verified/`.

-----------------------------------------------------------
## Running STMP
- To run STMP on an input VCF:  
		python stmp.py --vcf=(path to input VCF) --output_dir=(output directory)

Example (cd to the unzipped STMP release folder you downloaded):  
		python code/stmp.py --vcf=data/test/input_data/genome_in_a_bottle/subset.rs.vcf --output_dir=outputs/genome_in_a_bottle_output

This will run three different modules: annotation, tiering, and pharmacogenomics (pgx).

### 1) Annotation
This module annotates the input VCF with information from each of the above datasets. It outputs a TSV (tab-separated values) file with each annotation as a separate column (after the standard VCF columns). Annotation includes point annotation, functional annotation (using ANNOVAR and snpEff), and region (range) annotation using bedtools. Intermediate outputs of specific annotations (e.g. point annotations) are available in the scratch folder within the output directory. The final output (each of these three annotation types joined into a single file) is written as a .tsv file in the specified output directory.

### 2) Tiering
This module takes the annotated TSV from the previous step and prioritizes the variants into different tiers (below). It outputs the list of variants in each tier as an Excel worksheet (`tiered_output.xls`).It also outputs the variants and tiering metrics as text files (`tiering_allvars.metrics`, `tiering_allvars.tier0.txt`, `tiering_allvars.tier1.txt`, etc.).

- Tier 0: Variants classified as pathogenic or likely pathogenic according to ClinVar (with ClinVar star rating > 0 according to the new [mid-2015 guidelines](http://www.ncbi.nlm.nih.gov/clinvar/docs/details/)).

- Tier 1: Loss of function variants (splice dinucleotide disrupting, nonsense, nonstop, and frameshift indels.

- Tier 2: All rare variants cataloged in HGMD, regardless of functional annotation. Rarity is defined as minor allele frequency (MAF) no greater than 1% by default or according to use-defined criteria in any of the following population genetic surveys: ethnically- matched population in HapMap 2 and 3, the 1000 genomes phase 1 data33 from an ethnically-matched super population, and global allele frequency, the 1000 genomes pilot 1 project global allele frequency, 69 publicly available genomes released by Complete Genomics, and the NHLBI Grand Opportunity exome sequencing project global allele frequency.

- Tier 3: All non-rare missense and non-frameshift indels.

- Tier 4: All other rare exonic/splicing variants with ExAC tolerance z-score (`syn_z` or `mis_z` or `lof_z`) > 2


### 3) Pharmacogenomics (pgx)
This module takes in a VCF file and outputs several text files summarizing variants with known pharmacogenomic effects. These include effects on drug dosage, efficacy, toxicity, and other interactions, as well as whether any variants in the input file match known "star" alleles associated with clinical drug response for 6 genes (CYP2C19, CYP2C9, CYP2D6, SLCO1B1, TPMT, VKORC1). Each of these files is output in the specified output directory.

For additional options, run `python stmp.py -h`. For example, one can use the `--annotate_only` flag to run only the annotation module, the `--tiering_only` flag to run just the tiering module, or the `--pgx_only` flag to run just the pgx module. Note that tiering depends on the annotated output file, so annotation must be run before tiering.


### 4) Trio (separate script)
This module analyzes genome sequence data from a father, mother, and child. It takes as input a single VCF with different sample IDs for mother, father, and child.

Usage:
	python trio/trioPipeline.py input output path_to_annovar path_to_matrix offspringID fatherID motherID

Example:
(Note: as the combined file is large, you must download the HG002, HG003, and HG004 VCFs from this site ([ftp://ftp-trace.ncbi.nlm.nih.gov/giab/ftp/data/AshkenazimTrio/analysis/NIST_CallsIn2Technologies_05182015/](ftp://ftp-trace.ncbi.nlm.nih.gov/giab/ftp/data/AshkenazimTrio/analysis/NIST_CallsIn2Technologies_05182015/)) and manually combine them into a single VCF file using bcftools or similar. The sample command below assumes you have placed the combined file in the `data/test/input_data/trio` directory and called it `trio.combined.vcf`.)

	python code/trio/trioPipeline.py data/test/input_data/trio/trio.combined.vcf outputs/trio_output/ third_party/annovar/ code/trio/ HG002 HG003 HG004


-----------------------------------------------------------
## Customization
- Different datasets can be imported and used for annotation. To import additional datasets, modify `code/config/datasets.yml` to add information about the desired datasets. It is recommended that you make a backup copy of this file before modifying. Alternately, you can copy this file to a different location and use the `--config` flag to run stmp with it (e.g. `python stmp.py --config=(path to YAML file)`). For more information and examples regarding how to specify dataset information, see the specification file at `code/config/datasets_spec.yml` and the existing datasets in `code/config/datasets.yml`.


-----------------------------------------------------------
## Acknowledgement

When using this tool in published works, please cite the below publication:

Dewey FE, Grove ME, Priest JR, Waggott D, Batra P, Miller CL, et al. (2015) "Sequence to Medical Phenotypes: A Framework for Interpretation of Human Whole Genome DNA Sequence Data." PLoS Genet 11(10): e1005496. doi:10.1371/journal.pgen.1005496


-----------------------------------------------------------
## Appendices


### Appendix 1: List of ANNOVAR datasets to download for functional annotation
**NOTE: `hg19_wgEncodeGencodeBasicV19Mrna.fa` is no longer provided by Annovar/UCSC and must instead be downloaded manually from `https://www.dropbox.com/s/icw1loscvpm6v84/hg19_wgEncodeGencodeBasicV19Mrna.fa?dl=0` and copied to the annovar/humandb folder. Without this file, `Annovar_ExonicFunc_wgEncodeGencodeBasicV19` and `Annovar_AAChange_wgEncodeGencodeBasicV19` will not show up correctly in the annotated output.**

	GRCh37_MT_ensGeneMrna.fa
	GRCh37_MT_ensGene.txt
	hg19_example_db_generic.txt
	hg19_example_db_gff3.txt
	hg19_kgXref.txt
	hg19_knownGeneMrna.fa
	hg19_knownGene.txt
	hg19_MT_ensGeneMrna.fa
	hg19_MT_ensGene.txt
	hg19_refGeneMrna.fa
	hg19_refGene.txt
	hg19_wgEncodeGencodeBasicV19Mrna.fa - **see note above**
	hg19_wgEncodeGencodeBasicV19.txt



### Appendix 2: List of ANNOVAR datasets to download for trio tools
	decipher_chr.txt
	decipher_copy_edit10.txt
	decipher_gff.txt
	ex1.human.log
	galaxy_gff3.txt
	gff3test.txt
	gt_gff_test.txt
	hapmap_3.3.hg19_all.sites.txt
	hg18_cytoBand.txt
	hg18_example_db_generic.txt
	hg18_example_db_gff3.txt
	hg18_refGeneMrna.fa
	hg18_refGene.txt
	hg18_refLink.txt
	hg19_AFR.sites.2012_04.txt
	hg19_AFR.sites.2012_04.txt.idx
	hg19_ALL.sites.2010_11.txt
	hg19_ALL.sites.2011_05.txt
	hg19_ALL.sites.2011_05.txt.idx
	hg19_ALL.sites.2012_02.txt
	hg19_ALL.sites.2012_02.txt.idx
	hg19_ALL.sites.2012_04.txt
	hg19_ALL.sites.2012_04.txt.idx
	hg19_AMR.sites.2012_04.txt
	hg19_AMR.sites.2012_04.txt.idx
	hg19_ASN.sites.2012_04.txt
	hg19_ASN.sites.2012_04.txt.idx
	hg19_avsift.txt
	hg19_avsift.txt.idx
	hg19_cg46.txt
	hg19_cg46.txt.idx
	hg19_cg69.txt
	hg19_cg69.txt.idx
	hg19.clinvar.2.18.13.txt
	hg19_clinvarRegion.txt
	hg19_clinvarUrl.txt
	hg19_cosmic61.txt
	hg19_cosmic61.txt.idx
	hg19_cpgIslandExt.txt
	hg19_dgv.txt
	hg19_ensemblPseudogene.txt
	hg19_ensGeneMrna.fa
	hg19_ensGene.txt
	hg19_esp5400_aa.txt
	hg19_esp5400_aa.txt.idx
	hg19_esp5400_all.txt
	hg19_esp5400_all.txt.idx
	hg19_esp5400_ea.txt
	hg19_esp5400_ea.txt.idx
	hg19_esp6500_aa.txt
	hg19_esp6500_aa.txt.idx
	hg19_esp6500_all.txt
	hg19_esp6500_all.txt.idx
	hg19_esp6500_ea.txt
	hg19_esp6500_ea.txt.idx
	hg19_esp6500si_aa.txt
	hg19_esp6500si_aa.txt.idx
	hg19_esp6500si_all.txt
	hg19_esp6500si_all.txt.idx
	hg19_esp6500si_ea.txt
	hg19_esp6500si_ea.txt.idx
	hg19_EUR.sites.2012_04.txt
	hg19_EUR.sites.2012_04.txt.idx
	hg19_evofold.txt
	hg19_geneReviews.txt
	hg19_genomicSuperDups.txt
	hg19_gerp++gt2.txt
	hg19_gerp++gt2.txt.idx
	hg19_gwasCatalog.txt
	hg19.hapmap2and3_ASW.txt
	hg19.hapmap2and3_CEU.txt
	hg19.hapmap2and3_CHB.txt
	hg19.hapmap2and3_CHD.txt
	hg19.hapmap2and3_GIH.txt
	hg19.hapmap2and3_JPT.txt
	hg19.hapmap2and3_LWK.txt
	hg19.hapmap2and3_MEX.txt
	hg19.hapmap2and3_MKK.txt
	hg19.hapmap2and3_TSI.txt
	hg19.hapmap2and3_YRI.txt
	hg19_kgXref.txt
	hg19_knownBiocyc.txt
	hg19_knownGeneCEU.fa
	hg19_knownGeneMrna.fa
	hg19_knownGene.txt
	hg19_knownGene.txt.fa
	hg19_knownKegg.txt
	hg19_ljb_all.txt
	hg19_ljb_all.txt.idx
	hg19_omimGene.txt
	hg19_pgkbAnnot.txt
	hg19_pgkbUrl.txt
	hg19_phastConsElements46way.txt
	hg19_pseudogeneYale70.txt
	hg19_refGeneMrna.fa
	hg19_refGene.txt
	hg19_refLink.txt
	hg19.regulome.cat1.txt
	hg19_regulomeCat1.txt
	hg19_rmsk.txt
	hg19_snp130.txt
	hg19_snp130.txt.idx
	hg19_snp132.txt
	hg19_snp135.txt
	hg19_snp135.txt.idx
	hg19_snp137.txt
	hg19_targetScanS.txt
	hg19_tfbsConsSites.txt
	hg19_ucscGenePfam.txt
	hg19_wgEncodeBroadHistoneGm12878H3k27acStdSig.txt
	hg19_wgEncodeBroadHistoneGm12878H3k4me1StdSig.txt
	hg19_wgEncodeBroadHistoneGm12878H3k4me3StdSig.txt
	hg19_wgEncodeBroadHmmGm12878HMM.txt
	hg19_wgEncodeBroadHmmH1hescHMM.txt
	hg19_wgEncodeBroadHmmHmecHMM.txt
	hg19_wgEncodeBroadHmmHsmmHMM.txt
	hg19_wgEncodeBroadHmmHuvecHMM.txt
	hg19_wgEncodeBroadHmmNhekHMM.txt
	hg19_wgEncodeBroadHmmNhlfHMM.txt
	hg19_wgEncodeGencodeCompV14Mrna.fa
	hg19_wgEncodeGencodeCompV14.txt
	hg19_wgEncodeGencodeManualV4.txt
	hg19_wgEncodeRegDnaseClustered.txt
	hg19_wgEncodeRegDnaseClusteredV2.txt
	hg19_wgEncodeRegTfbsClustered.txt
	hg19_wgEncodeRegTfbsClusteredV2.txt
	hg19_wgRna.txt
