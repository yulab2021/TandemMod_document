.. _run_examples:

Run examples
==================================
This sections gives examples on how to use the three modes of TandemMod.

Train m6A model using IVET m6A dataset
********************
IVET datasets have been uploaded to GEO database under the accession number GSE227087. To train a m6A detection model, the followinng two fast5 files (m6A-modified and unmodified) are required::

    IVET_DRS_unmodified.tar.gz 
    IVET_DRS_m6A.tar.gz 

In this demo, subsets of the two datasets were taken for demonstration purposes due to the large size of the original datasets::
    demo
    └── IVET_m6A
        ├── IVET_m6A
        │   └── IVET_m6A.fast5
        └── IVET_unmod
            └── IVET_unmod.fast5


minimap2 -ax map-ont demo/reference_transcripts.fasta demo/IVET/IVET_m6A_guppy/pass/*.fastq >demo/test.sam

guppy basecalling

guppy_basecaller -i demo/IVET/IVET_m6A -s demo/IVET/IVET_m6A_guppy --num_callers 40 --recursive --fast5_out --config rna_r9.4.1_70bps_hac.cfg
guppy_basecaller -i demo/IVET/IVET_unmod -s demo/IVET/IVET_unmod_guppy --num_callers 40 --recursive --fast5_out --config rna_r9.4.1_70bps_hac.cfg

multi_to_single

multi_to_single_fast5 -i demo/IVET/IVET_m6A_guppy -s demo/IVET/IVET_m6A_guppy_single --recursive
multi_to_single_fast5 -i demo/IVET/IVET_unmod_guppy -s demo/IVET/IVET_unmod_guppy_single --recursive

#annoate
tombo preprocess annotate_raw_with_fastqs --fast5-basedir demo/IVET/IVET_m6A_guppy_single --fastq-filenames demo/IVET/IVET_m6A_guppy/pass/*.fastq

#tombo
tombo resquiggle --overwrite --basecall-group Basecall_1D_001 demo/IVET/IVET_m6A_guppy_single  demo/IVET_reference.fa --processes 40 --fit-global-scale --include-event-stdev
tombo resquiggle --overwrite --basecall-group Basecall_1D_001 demo/IVET/IVET_unmod_guppy_single  demo/IVET_reference.fa --processes 40 --fit-global-scale --include-event-stdev

#map

cat demo/IVET/IVET_m6A_guppy/pass/*.fastq >demo/IVET/IVET_m6A.fastq
minimap2 -ax map-ont demo/IVET_reference.fa demo/IVET/IVET_m6A.fastq >demo/IVET/IVET_m6A.sam

cat demo/IVET/IVET_unmod_guppy/pass/*.fastq >demo/IVET/IVET_unmod.fastq
minimap2 -ax map-ont demo/IVET_reference.fa demo/IVET/IVET_unmod.fastq >demo/IVET/IVET_unmod.sam

#feature extraction


python scripts/extract_signal_from_fast5.py -p 40 --fast5 demo/IVET/IVET_m6A_guppy_single --reference demo/IVET_reference.fa --sam demo/IVET/IVET_m6A.sam --output demo/IVET/m6A.signal.tsv --clip=10
python scripts/extract_feature_from_signal.py  --signal_file demo/IVET/m6A.signal.tsv --clip 10 --output demo/IVET/m6A.feature.tsv --motif DRACH


Train m6A model using curlcake m6A dataset
********************




Transfer m6A model to m7G using ELIGOS dataset
********************




Predict m6A sites in human cell lines
********************