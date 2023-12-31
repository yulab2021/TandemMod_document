.. _run_examples:

Run examples
==================================
This sections gives examples on how to use the three modes of TandemMod.

Train m6A model using IVET m6A dataset
********************
IVET datasets have been uploaded to GEO database under the accession number `GSE227087 <https://www.ncbi.nlm.nih.gov/geo/query/acc.cgi?acc=GSE227087>`_. To train a m6A detection model, the followinng two fast5 files (m6A-modified and unmodified) are required.
::
    IVET_DRS_m6A.tar.gz 
    IVET_DRS_unmodified.tar.gz 
    

In this demo, subsets of the two datasets were taken for demonstration purposes due to the large size of the original datasets. The demo datasets were located undelr ``./demo/IVET/`` directory.
::
    demo
    └── IVET
        ├── IVET_m6A
        │   └── IVET_m6A.fast5
        └── IVET_unmod
            └── IVET_unmod.fast5


**1. Guppy basecalling**

Basecalling converts the raw signal generated by Oxform Nanopore sequencing to DNA/RNA sequence. Guppy is used for basecalling in this step. In some nanopore datasets, the sequence information is already contained within the FAST5 files. In such cases, the basecalling step can be skipped as the sequence data is readily available.
::
    #m6A 
    guppy_basecaller -i demo/IVET/IVET_m6A -s demo/IVET/IVET_m6A_guppy --num_callers 40 --recursive --fast5_out --config rna_r9.4.1_70bps_hac.cfg
    
    #unmodified
    guppy_basecaller -i demo/IVET/IVET_unmod -s demo/IVET/IVET_unmod_guppy --num_callers 40 --recursive --fast5_out --config rna_r9.4.1_70bps_hac.cfg

**2. Multi-reads FAST5 files to single-read FAST5 files**

Convert multi-reads FAST5 files to single-read FAST5 files. If the data generated by the sequencing device is already in the single-read format, this step can be skipped.
::
    #m6A 
    multi_to_single_fast5 -i demo/IVET/IVET_m6A_guppy -s demo/IVET/IVET_m6A_guppy_single --recursive
    
    #unmodified
    multi_to_single_fast5 -i demo/IVET/IVET_unmod_guppy -s demo/IVET/IVET_unmod_guppy_single --recursive

**3. Tombo resquiggling**

In this step, the sequence obtained by basecalling is aligned or mapped to a reference genome or a known sequence. Then the corrected sequence is then associated with the corresponding current signals. The resquiggling process is typically performed in-place. No separate files are generated in this step.
::
    #m6A
    tombo resquiggle --overwrite --basecall-group Basecall_1D_001 demo/IVET/IVET_m6A_guppy_single  demo/IVET_reference.fa --processes 40 --fit-global-scale --include-event-stdev
    
    #unmodified
    tombo resquiggle --overwrite --basecall-group Basecall_1D_001 demo/IVET/IVET_unmod_guppy_single  demo/IVET_reference.fa --processes 40 --fit-global-scale --include-event-stdev

**4. Map reads to reference**

minimap2 is used to map basecalled sequences to reference transcripts. The output sam file serves as the input for the subsequent feature extraction step. 
::
    #m6A
    cat demo/IVET/IVET_m6A_guppy/pass/*.fastq >demo/IVET/IVET_m6A.fastq
    minimap2 -ax map-ont demo/IVET_reference.fa demo/IVET/IVET_m6A.fastq >demo/IVET/IVET_m6A.sam

    #unmodified
    cat demo/IVET/IVET_unmod_guppy/pass/*.fastq >demo/IVET/IVET_unmod.fastq
    minimap2 -ax map-ont demo/IVET_reference.fa demo/IVET/IVET_unmod.fastq >demo/IVET/IVET_unmod.sam

**5. Feature extraction**

Extract signals and features from resquiggled fast5 files using the following python scripts.
::
    #m6A
    python scripts/extract_signal_from_fast5.py -p 40 --fast5 demo/IVET/IVET_m6A_guppy_single --reference demo/IVET_reference.fa --sam demo/IVET/IVET_m6A.sam --output demo/IVET/m6A.signal.tsv --clip 10
    python scripts/extract_feature_from_signal.py  --signal_file demo/IVET/m6A.signal.tsv --clip 10 --output demo/IVET/m6A.feature.tsv --motif DRACH
    
    #unmodified
    python scripts/extract_signal_from_fast5.py -p 40 --fast5 demo/IVET/IVET_unmod_guppy_single --reference demo/IVET_reference.fa --sam demo/IVET/IVET_unmod.sam --output demo/IVET/unmod.signal.tsv --clip 10
    python scripts/extract_feature_from_signal.py  --signal_file demo/IVET/unmod.signal.tsv --clip 10 --output demo/IVET/unmod.feature.tsv --motif DRACH

In the feature extraction step, the motif pattern should be provided using the argument ``--motif``. The base symbols of the motif follow the IUB code standard. Here is the full definition of IUB base symbols:

+-------------+-------------+
| IUB Base    | Expansion   |
+=============+=============+
| A           | A           |
+-------------+-------------+
| C           | C           |
+-------------+-------------+
| G           | G           |
+-------------+-------------+
| T           | T           |
+-------------+-------------+
| M           | AC          |
+-------------+-------------+
| V           | ACG         |
+-------------+-------------+
| R           | AG          |
+-------------+-------------+
| H           | ACT         |
+-------------+-------------+
| W           | AT          |
+-------------+-------------+
| D           | AGT         |
+-------------+-------------+
| S           | CG          |
+-------------+-------------+
| B           | CGT         |
+-------------+-------------+
| Y           | CT          |
+-------------+-------------+
| N           | ACGT        |
+-------------+-------------+
| K           | GT          |
+-------------+-------------+



**6. Train-test split**

The train-test split is performed randomly, ensuring that the data points in each set are representative of the overall dataset. The default split ratios are 80% for training and 20% for testing. The train-test split ratio can be customized by using the argument ``--train_ratio`` to accommodate the specific requirements of the problem and the size of the dataset.

The training set is used to train the model, allowing it to learn patterns and relationships present in the data. The testing set, on the other hand, is used to assess the model's performance on new, unseen data. It serves as an independent evaluation set to measure how well the trained model generalizes to data it has not encountered before. By evaluating the model on the testing set, we can estimate its performance, detect overfitting (when the model performs well on the training set but poorly on the testing set) and assess its ability to make accurate predictions on new data.
::
    usage: train_test_split.py [-h] [--input_file INPUT_FILE]
                               [--train_file TRAIN_FILE] [--test_file TEST_FILE]
                               [--train_ratio TRAIN_RATIO]
    
    Split a feature file into training and testing sets.
    
    optional arguments:
      -h, --help                  show this help message and exit
      --input_file INPUT_FILE     Path to the input feature file
      --train_file TRAIN_FILE     Path to the train feature file
      --test_file TEST_FILE       Path to the test feature file
      --train_ratio TRAIN_RATIO   Ratio of instances to use for training (default: 0.8)

    #m6A
    python scripts/train_test_split.py --input_file demo/IVET/m6A.feature.tsv --train_file demo/IVET/m6A.train.feature.tsv --test_file demo/IVET/m6A.test.feature.tsv --train_ratio 0.8
    
    #unmodified
    python scripts/train_test_split.py --input_file demo/IVET/unmod.feature.tsv --train_file demo/IVET/unmod.train.feature.tsv --test_file demo/IVET/unmod.test.feature.tsv --train_ratio 0.8


**7. Train m6A model**

To train the TandemMod model using your own dataset from scratch, you can set the ``--run_mode`` argument to "train". TandemMod accepts both modified and unmodified feature files as input. Additionally, test feature files are necessary to evaluate the model's performance. You can specify the model save path by using the argument ``--new_model``. The model's training epochs can be defined using the argument ``--epochs``, and the model states will be saved at the end of each epoch. TandemMod will preferentially use the ``GPU`` for training if CUDA is available on your device; otherwise, it will utilize the ``CPU`` mode. The training process duration can vary, depending on the size of your dataset and the computational capacity, and may last for several hours. 
::
    python scripts/TandemMod.py --run_mode train \
      --new_model demo/model/m6A.demo.IVET.pkl \
      --train_data_mod demo/IVET/m6A.train.feature.tsv \
      --train_data_unmod demo/IVET/unmod.train.feature.tsv \
      --test_data_mod demo/IVET/m6A.test.feature.tsv \
      --test_data_unmod demo/IVET/unmod.test.feature.tsv \
      --epoch 100

During training process, the following information can be used to monitor and evaluate the performance of the model:
::
    device= cpu
    train process.
    data loaded.
    start training...
    Epoch 0-0 Train acc: 0.494000,Test Acc: 0.581081,time0:00:08.936393
    Epoch 1-0 Train acc: 0.514000,Test Acc: 0.817568,time0:00:06.084542
    Epoch 2-0 Train acc: 0.796000,Test Acc: 0.668919,time0:00:06.000019
    Epoch 3-0 Train acc: 0.672000,Test Acc: 0.770270,time0:00:07.456637
    Epoch 4-0 Train acc: 0.786000,Test Acc: 0.763514,time0:00:06.132852
    Epoch 5-0 Train acc: 0.824000,Test Acc: 0.834459,time0:00:06.584059
    Epoch 6-0 Train acc: 0.810000,Test Acc: 0.814189,time0:00:06.600892
    Epoch 7-0 Train acc: 0.780000,Test Acc: 0.790541,time0:00:07.301838

After the data processing and model training, the following files should be generated by TandemMod. The trained model ``m6A.demo.IVET.pkl`` will be saved in the ``./demo/model/`` folder. You can utilize this model for making predictions in the future.
::
    demo
    ├── IVET
    │   ├── IVET_m6A
    │   ├── IVET_m6A.fastq
    │   ├── IVET_m6A_guppy
    │   ├── IVET_m6A_guppy_single
    │   ├── IVET_m6A.sam
    │   ├── IVET_unmod
    │   ├── IVET_unmod.fastq
    │   ├── IVET_unmod_guppy
    │   ├── IVET_unmod_guppy_single
    │   ├── IVET_unmod.sam
    │   ├── m6A.feature.tsv
    │   ├── m6A.signal.tsv
    │   ├── m6A.test.feature.tsv
    │   ├── m6A.train.feature.tsv
    │   ├── unmod.feature.tsv
    │   ├── unmod.signal.tsv
    │   ├── unmod.test.feature.tsv
    │   └── unmod.train.feature.tsv
    ├── IVET_reference.fa
    └── model
           └── m6A.demo.IVET.pkl


Train m6A model using curlcake m6A dataset
********************
Curlcake datasets are publicly available at the GEO database under the accession code `GSE124309 <https://www.ncbi.nlm.nih.gov/geo/query/acc.cgi?acc=GSE124309>`_. In this demo, subsets of the curcake datasets (m6A-modified and unmodified) were taken for demonstration purposes due to the large size of the original datasets. The demo datasets were located under ``./demo/curlcake/`` directory.
::
    demo
    └── curlcake
        ├── curlcake_m6A
        │   └── curlcake_m6A.fast5
        └── curlcake_unmod
            └── curlcake_unmod.fast5

**1. Guppy basecalling**

Basecalling converts the raw signal generated by Oxform Nanopore sequencing to DNA/RNA sequence. Guppy is used for basecalling in this step. In some nanopore datasets, the sequence information is already contained within the FAST5 files. In such cases, the basecalling step can be skipped as the sequence data is readily available.
::
    #m6A 
    guppy_basecaller -i demo/curlcake/curlcake_m6A -s demo/curlcake/curlcake_m6A_guppy --num_callers 40 --recursive --fast5_out --config rna_r9.4.1_70bps_hac.cfg
    
    #unmodified
    guppy_basecaller -i demo/curlcake/curlcake_unmod -s demo/curlcake/curlcake_unmod_guppy --num_callers 40 --recursive --fast5_out --config rna_r9.4.1_70bps_hac.cfg

**2. Multi-reads FAST5 files to single-read FAST5 files**

Convert multi-reads FAST5 files to single-read FAST5 files. If the data generated by the sequencing device is already in the single-read format, this step can be skipped.
::
    #m6A 
    multi_to_single_fast5 -i demo/curlcake/curlcake_m6A_guppy -s demo/curlcake/curlcake_m6A_guppy_single --recursive
    
    #unmodified
    multi_to_single_fast5 -i demo/curlcake/curlcake_unmod_guppy -s demo/curlcake/curlcake_unmod_guppy_single --recursive

**3. Tombo resquiggling**

In this step, the sequence obtained by basecalling is aligned or mapped to a reference genome or a known sequence. Then the corrected sequence is then associated with the corresponding current signals. The resquiggling process is typically performed in-place. No separate files are generated in this step. Curlcake reference file can be download `here <https://www.ncbi.nlm.nih.gov/geo/download/?acc=GSE124309&format=file&file=GSE124309%5FFASTA%5Fsequences%5Fof%5FCurlcakes%2Etxt%2Egz>`_. 
::
    #m6A
    tombo resquiggle --overwrite --basecall-group Basecall_1D_001 demo/curlcake/curlcake_m6A_guppy_single  demo/curlcake_reference.fa --processes 40 --fit-global-scale --include-event-stdev
    
    #unmodified
    tombo resquiggle --overwrite --basecall-group Basecall_1D_001 demo/curlcake/curlcake_unmod_guppy_single  demo/curlcake_reference.fa --processes 40 --fit-global-scale --include-event-stdev

**4. Map reads to reference**

minimap2 is used to map basecalled sequences to reference transcripts. The output sam file serves as the input for the subsequent feature extraction step. 
::
    #m6A
    cat demo/curlcake/curlcake_m6A_guppy/pass/*.fastq >demo/curlcake/curlcake_m6A.fastq
    minimap2 -ax map-ont demo/curlcake_reference.fa demo/curlcake/curlcake_m6A.fastq >demo/curlcake/curlcake_m6A.sam

    #unmodified
    cat demo/curlcake/curlcake_unmod_guppy/pass/*.fastq >demo/curlcake/curlcake_unmod.fastq
    minimap2 -ax map-ont demo/curlcake_reference.fa demo/curlcake/curlcake_unmod.fastq >demo/curlcake/curlcake_unmod.sam

**5. Feature extraction**

Extract signals and features from resquiggled fast5 files using the following python scripts.
::
    #m6A
    python scripts/extract_signal_from_fast5.py -p 40 --fast5 demo/curlcake/curlcake_m6A_guppy_single --reference demo/curlcake_reference.fa --sam demo/curlcake/curlcake_m6A.sam --output demo/curlcake/m6A.signal.tsv --clip=10
    python scripts/extract_feature_from_signal.py  --signal_file demo/curlcake/m6A.signal.tsv --clip 10 --output demo/curlcake/m6A.feature.tsv --motif DRACH
    
    #unmodified
    python scripts/extract_signal_from_fast5.py -p 40 --fast5 demo/curlcake/curlcake_unmod_guppy_single --reference demo/curlcake_reference.fa --sam demo/curlcake/curlcake_unmod.sam --output demo/curlcake/unmod.signal.tsv --clip=10
    python scripts/extract_feature_from_signal.py  --signal_file demo/curlcake/unmod.signal.tsv --clip 10 --output demo/curlcake/unmod.feature.tsv --motif DRACH

In the feature extraction step, the motif pattern should be provided using the argument ``--motif``. The base symbols of the motif follow the IUB code standard. 


**6. Train-test split**

The train-test split is performed randomly, ensuring that the data points in each set are representative of the overall dataset. The default split ratios are 80% for training and 20% for testing. The train-test split ratio can be customized by using the argument ``--train_ratio`` to accommodate the specific requirements of the problem and the size of the dataset.

The training set is used to train the model, allowing it to learn patterns and relationships present in the data. The testing set, on the other hand, is used to assess the model's performance on new, unseen data. It serves as an independent evaluation set to measure how well the trained model generalizes to data it has not encountered before. By evaluating the model on the testing set, we can estimate its performance, detect overfitting (when the model performs well on the training set but poorly on the testing set) and assess its ability to make accurate predictions on new data.
::
    usage: train_test_split.py [-h] [--input_file INPUT_FILE]
                               [--train_file TRAIN_FILE] [--test_file TEST_FILE]
                               [--train_ratio TRAIN_RATIO]
    
    Split a feature file into training and testing sets.
    
    optional arguments:
      -h, --help                  show this help message and exit
      --input_file INPUT_FILE     Path to the input feature file
      --train_file TRAIN_FILE     Path to the train feature file
      --test_file TEST_FILE       Path to the test feature file
      --train_ratio TRAIN_RATIO   Ratio of instances to use for training (default: 0.8)

    #m6A
    python scripts/train_test_split.py --input_file demo/curlcake/m6A.feature.tsv --train_file demo/curlcake/m6A.train.feature.tsv --test_file demo/curlcake/m6A.test.feature.tsv --train_ratio 0.8
    
    #unmodified
    python scripts/train_test_split.py --input_file demo/curlcake/unmod.feature.tsv --train_file demo/curlcake/unmod.train.feature.tsv --test_file demo/curlcake/unmod.test.feature.tsv --train_ratio 0.8


**7. Train m6A model**

To train the TandemMod model using your own dataset from scratch, you can set the ``--run_mode`` argument to "train". TandemMod accepts both modified and unmodified feature files as input. Additionally, test feature files are necessary to evaluate the model's performance. You can specify the model save path by using the argument ``--new_model``. The model's training epochs can be defined using the argument ``--epochs``, and the model states will be saved at the end of each epoch. TandemMod will preferentially use the ``GPU`` for training if CUDA is available on your device; otherwise, it will utilize the ``CPU`` mode. The training process duration can vary, depending on the size of your dataset and the computational capacity, and may last for several hours. 
::
    python scripts/TandemMod.py --run_mode train \
      --new_model demo/model/m6A.demo.curlcake.pkl \
      --train_data_mod demo/curlcake/m6A.train.feature.tsv \
      --train_data_unmod demo/curlcake/unmod.train.feature.tsv \
      --test_data_mod demo/curlcake/m6A.test.feature.tsv \
      --test_data_unmod demo/curlcake/unmod.test.feature.tsv \
      --epoch 100

During training process, the following information can be used to monitor and evaluate the performance of the model:
::
    device= cpu
    train process.
    data loaded.
    start training...
    Epoch 0-0 Train acc: 0.482000,Test Acc: 0.788462,time0:00:07.666192
    Epoch 1-0 Train acc: 0.514000,Test Acc: 0.211538,time0:00:04.977504
    Epoch 2-0 Train acc: 0.496000,Test Acc: 0.211538,time0:00:05.498799
    Epoch 3-0 Train acc: 0.694000,Test Acc: 0.432692,time0:00:05.893204
    Epoch 4-0 Train acc: 0.814000,Test Acc: 0.639423,time0:00:06.149194
    Epoch 5-0 Train acc: 0.806000,Test Acc: 0.711538,time0:00:05.443221
    Epoch 6-0 Train acc: 0.828000,Test Acc: 0.831731,time0:00:05.706294
    Epoch 7-0 Train acc: 0.808000,Test Acc: 0.846154,time0:00:05.674450
    Epoch 8-0 Train acc: 0.804000,Test Acc: 0.822115,time0:00:05.956936


After the data processing and model training, the following files should be generated by TandemMod. The trained model ``m6A.demo.curlcake.pkl`` will be saved in the ``./demo/model/`` folder. You can utilize this model for making predictions in the future.
::
    demo
    ├── curlcake
    │   ├── curlcake_m6A
    │   ├── curlcake_m6A.fastq
    │   ├── curlcake_m6A_guppy
    │   ├── curlcake_m6A_guppy_single
    │   ├── curlcake_m6A.sam
    │   ├── curlcake_unmod
    │   ├── curlcake_unmod.fastq
    │   ├── curlcake_unmod_guppy
    │   ├── curlcake_unmod_guppy_single
    │   ├── curlcake_unmod.sam
    │   ├── m6A.feature.tsv
    │   ├── m6A.signal.tsv
    │   ├── m6A.test.feature.tsv
    │   ├── m6A.train.feature.tsv
    │   ├── unmod.feature.tsv
    │   ├── unmod.signal.tsv
    │   ├── unmod.test.feature.tsv
    │   └── unmod.train.feature.tsv
    ├── curlcake_reference.fa
    └── model
           └── m6A.demo.curlcake.pkl


Transfer m6A model to m7G using ELIGOS dataset
********************

To transfer the pretrained m6A model to an m7G prediction model using the ELIGOS dataset, you can follow these steps:

* Obtain the ELIGOS dataset: Download or access the ELIGOS m7G dataset, which consists of the necessary data (m7G-modified and unmodified) for training and testing.

* Prepare the data: Preprocess the ELIGOS dataset to extact features for transfer learning.

* Load the pretrained m6A model: Load the pretrained m6A model that you want to transfer to predict m7G modifications. This model should have been previously trained on a relevant m6A dataset.

* Train the modified model: Use the ELIGOS m7G dataset to fine-tune the model's parameters using transfer learning techniques.

* Evaluate the performance: Assess the performance of the transferred m7G model on the m7G testing set from the ELIGOS dataset.

By following these steps, you can transfer the knowledge gained from the pretrained m6A model to predict m7G modifications using the ELIGOS dataset.

ELIGOS datasets are publicly available at the SRA database under the accession code `SRP166020 <https://www.ncbi.nlm.nih.gov/sra/?term=SRP166020>`_. In this demo, subsets of the ELIGOS datasets (m7G-modified and unmodified) were taken for demonstration purposes due to the large size of the original datasets. The demo datasets were located under ``./demo/ELIGOS/`` directory.
::
    demo
    └── ELIGOS
        ├── ELIGOS_m7G
        │   └── ELIGOS_m7G.fast5
        └── ELIGOS_unmod
            └── ELIGOS_unmod.fast5

**1. Guppy basecalling**

Basecalling converts the raw signal generated by Oxform Nanopore sequencing to DNA/RNA sequence. Guppy is used for basecalling in this step. In some nanopore datasets, the sequence information is already contained within the FAST5 files. In such cases, the basecalling step can be skipped as the sequence data is readily available.
::
    #m7G 
    guppy_basecaller -i demo/ELIGOS/ELIGOS_m7G -s demo/ELIGOS/ELIGOS_m7G_guppy --num_callers 40 --recursive --fast5_out --config rna_r9.4.1_70bps_hac.cfg
    
    #unmodified
    guppy_basecaller -i demo/ELIGOS/ELIGOS_unmod -s demo/ELIGOS/ELIGOS_unmod_guppy --num_callers 40 --recursive --fast5_out --config rna_r9.4.1_70bps_hac.cfg

**2. Multi-reads FAST5 files to single-read FAST5 files**

Convert multi-reads FAST5 files to single-read FAST5 files. If the data generated by the sequencing device is already in the single-read format, this step can be skipped.
::
    #m7G 
    multi_to_single_fast5 -i demo/ELIGOS/ELIGOS_m7G_guppy -s demo/ELIGOS/ELIGOS_m7G_guppy_single --recursive
    
    #unmodified
    multi_to_single_fast5 -i demo/ELIGOS/ELIGOS_unmod_guppy -s demo/ELIGOS/ELIGOS_unmod_guppy_single --recursive

**3. Tombo resquiggling**

In this step, the sequence obtained by basecalling is aligned or mapped to a reference genome or a known sequence. Then the corrected sequence is then associated with the corresponding current signals. The resquiggling process is typically performed in-place. No separate files are generated in this step. ELIGOS reference file can be download `here <https://oup.silverchair-cdn.com/oup/backfile/Content_public/Journal/nar/49/2/10.1093_nar_gkaa620/1/gkaa620_supplemental_files.zip?Expires=1690555116&Signature=Mv7ppemTnplIZAvv6G3W-lob1eQwK5IvNeIIF-1GM8Jy93AdT6ALUynRjW3HQAyCMgkMW-0WnXktuVJfKDCUXiiwvjZ9z5iO5LksCl1e6yEA5dgRlr-FVUrDbj81NIfUJNhKReo5gxRYc~f7wbFZRcy9CcSB-D1DloUmv-4qdcydr35sM-YDKgfyNfaE-ZKnCZZ1KydDNtx7oRfYHCof-a3oHSNgxn5DFM9bGCq147cw6i9B1bCURAPLltdPzR4i7cBXmIRoNZuVkjLe8EktJPg47v9ElqlPUlZfAqoaESbmPtEs8NLoX~~82o~eMrjwomK4W5CzgwAZhJJIeelr7A__&Key-Pair-Id=APKAIE5G5CRDK6RD3PGA>`_. 
::
    #m7G
    tombo resquiggle --overwrite --basecall-group Basecall_1D_001 demo/ELIGOS/ELIGOS_m7G_guppy_single  demo/ELIGOS_reference.fa --processes 40 --fit-global-scale --include-event-stdev
    
    #unmodified
    tombo resquiggle --overwrite --basecall-group Basecall_1D_001 demo/ELIGOS/ELIGOS_unmod_guppy_single  demo/ELIGOS_reference.fa --processes 40 --fit-global-scale --include-event-stdev

**4. Map reads to reference**

minimap2 is used to map basecalled sequences to reference transcripts. The output sam file serves as the input for the subsequent feature extraction step. 
::
    #m7G
    cat demo/ELIGOS/ELIGOS_m7G_guppy/pass/*.fastq >demo/ELIGOS/ELIGOS_m7G.fastq
    minimap2 -ax map-ont demo/ELIGOS_reference.fa demo/ELIGOS/ELIGOS_m7G.fastq >demo/ELIGOS/ELIGOS_m7G.sam

    #unmodified
    cat demo/ELIGOS/ELIGOS_unmod_guppy/pass/*.fastq >demo/ELIGOS/ELIGOS_unmod.fastq
    minimap2 -ax map-ont demo/ELIGOS_reference.fa demo/ELIGOS/ELIGOS_unmod.fastq >demo/ELIGOS/ELIGOS_unmod.sam

**5. Feature extraction**

Extract signals and features from resquiggled fast5 files using the following python scripts.
::
    #m7G
    python scripts/extract_signal_from_fast5.py -p 40 --fast5 demo/ELIGOS/ELIGOS_m7G_guppy_single --reference demo/ELIGOS_reference.fa --sam demo/ELIGOS/ELIGOS_m7G.sam --output demo/ELIGOS/m7G.signal.tsv --clip=10
    python scripts/extract_feature_from_signal.py  --signal_file demo/ELIGOS/m7G.signal.tsv --clip 10 --output demo/ELIGOS/m7G.feature.tsv --motif NNGNN
    
    #unmodified
    python scripts/extract_signal_from_fast5.py -p 40 --fast5 demo/ELIGOS/ELIGOS_unmod_guppy_single --reference demo/ELIGOS_reference.fa --sam demo/ELIGOS/ELIGOS_unmod.sam --output demo/ELIGOS/unmod.signal.tsv --clip=10
    python scripts/extract_feature_from_signal.py  --signal_file demo/ELIGOS/unmod.signal.tsv --clip 10 --output demo/ELIGOS/unmod.feature.tsv --motif NNGNN

In the feature extraction step, the motif pattern should be provided using the argument ``--motif``. The base symbols of the motif follow the IUB code standard. 


**6. Train-test split**

The train-test split is performed randomly, ensuring that the data points in each set are representative of the overall dataset. The default split ratios are 80% for training and 20% for testing. The train-test split ratio can be customized by using the argument ``--train_ratio`` to accommodate the specific requirements of the problem and the size of the dataset.

The training set is used to train the model, allowing it to learn patterns and relationships present in the data. The testing set, on the other hand, is used to assess the model's performance on new, unseen data. It serves as an independent evaluation set to measure how well the trained model generalizes to data it has not encountered before. By evaluating the model on the testing set, we can estimate its performance, detect overfitting (when the model performs well on the training set but poorly on the testing set) and assess its ability to make accurate predictions on new data.
::
    usage: train_test_split.py [-h] [--input_file INPUT_FILE]
                               [--train_file TRAIN_FILE] [--test_file TEST_FILE]
                               [--train_ratio TRAIN_RATIO]
    
    Split a feature file into training and testing sets.
    
    optional arguments:
      -h, --help                  show this help message and exit
      --input_file INPUT_FILE     Path to the input feature file
      --train_file TRAIN_FILE     Path to the train feature file
      --test_file TEST_FILE       Path to the test feature file
      --train_ratio TRAIN_RATIO   Ratio of instances to use for training (default: 0.8)

    #m7G
    python scripts/train_test_split.py --input_file demo/ELIGOS/m7G.feature.tsv --train_file demo/ELIGOS/m7G.train.feature.tsv --test_file demo/ELIGOS/m7G.test.feature.tsv --train_ratio 0.8
    
    #unmodified
    python scripts/train_test_split.py --input_file demo/ELIGOS/unmod.feature.tsv --train_file demo/ELIGOS/unmod.train.feature.tsv --test_file demo/ELIGOS/unmod.test.feature.tsv --train_ratio 0.8


**7. Train m7G model**

To transfer the pretrained TandemMod model to new types of modifications, you can set the ``--run_mode`` argument to "transfer". TandemMod accepts both modified and unmodified feature files as input. Additionally, test feature files are necessary to evaluate the model's performance. You can specify the pretrained model by using the argument ``--pretrained_model`` and the new model save path by using the argument ``--new_model``. The model's training epochs can be defined using the argument ``--epochs``, and the model states will be saved at the end of each epoch. TandemMod will preferentially use the ``GPU`` for training if CUDA is available on your device; otherwise, it will utilize the ``CPU`` mode. The training process duration can vary, depending on the size of your dataset and the computational capacity, and may last for several hours. 
::
    usage: TandemMod.py [-h] --run_mode RUN_MODE
                        [--pretrained_model PRETRAINED_MODEL]
                        [--new_model NEW_MODEL] [--train_data_mod TRAIN_DATA_MOD]
                        [--train_data_unmod TRAIN_DATA_UNMOD]
                        [--test_data_mod TEST_DATA_MOD]
                        [--test_data_unmod TEST_DATA_UNMOD]
                        [--feature_file FEATURE_FILE]
                        [--predict_result PREDICT_RESULT] [--epoch EPOCH]
    
    TandemMod, multiple types of RNA modification detection.
    
    optional arguments:
      -h, --help                               show this help message and exit
      --run_mode RUN_MODE                      Run mode. Default is train
      --pretrained_model PRETRAINED_MODEL      Pretrained model file.
      --new_model NEW_MODEL                    New model file to be saved.
      --train_data_mod TRAIN_DATA_MOD          Train data file, modified.
      --train_data_unmod TRAIN_DATA_UNMOD      Train data file, unmodified.
      --test_data_mod TEST_DATA_MOD            Test data file, modified.
      --test_data_unmod TEST_DATA_UNMOD        Test data file, unmodified.
      --epoch EPOCH                            Training epoch

    python scripts/TandemMod.py --run_mode transfer \
      --pretrained_model demo/model/m6A.demo.IVET.pkl \
      --new_model demo/model/m7G.demo.ELIGOS.transfered_from_IVET_m6A.pkl \
      --train_data_mod demo/ELIGOS/m7G.train.feature.tsv \
      --train_data_unmod demo/ELIGOS/unmod.train.feature.tsv \
      --test_data_mod demo/ELIGOS/m7G.test.feature.tsv \
      --test_data_unmod demo/ELIGOS/unmod.test.feature.tsv \
      --epoch 100

During training process, the following information can be used to monitor and evaluate the performance of the transfered model:
::
    device= cpu
    transfer learning process.
    data loaded.
    start training...
    Epoch 0-0 Train acc: 0.544000,Test Acc: 0.489786,time0:00:08.688707
    Epoch 1-0 Train acc: 0.674000,Test Acc: 0.857939,time0:00:05.190997
    Epoch 2-0 Train acc: 0.748000,Test Acc: 0.813835,time0:00:05.426035
    Epoch 3-0 Train acc: 0.778000,Test Acc: 0.753946,time0:00:05.180632
    Epoch 4-0 Train acc: 0.854000,Test Acc: 0.776230,time0:00:05.236281
    Epoch 5-0 Train acc: 0.886000,Test Acc: 0.817549,time0:00:05.219122
    Epoch 6-0 Train acc: 0.926000,Test Acc: 0.889044,time0:00:05.470729



After the data processing and model training, the following files should be generated by TandemMod. The trained model ``m7G.demo.ELIGOS.transfered_from_IVET_m6A.pkl`` will be saved in the ``./demo/model/`` folder. You can utilize this fine-tuned model for making predictions in the future.
::
    demo
    ├── ELIGOS
    │   ├── ELIGOS_m7G
    │   ├── ELIGOS_m7G.fastq
    │   ├── ELIGOS_m7G_guppy
    │   ├── ELIGOS_m7G_guppy_single
    │   ├── ELIGOS_m7G.sam
    │   ├── ELIGOS_unmod
    │   ├── ELIGOS_unmod.fastq
    │   ├── ELIGOS_unmod_guppy
    │   ├── ELIGOS_unmod_guppy_single
    │   ├── ELIGOS_unmod.sam
    │   ├── m7G.feature.tsv
    │   ├── m7G.signal.tsv
    │   ├── m7G.test.feature.tsv
    │   ├── m7G.train.feature.tsv
    │   ├── unmod.feature.tsv
    │   ├── unmod.signal.tsv
    │   ├── unmod.test.feature.tsv
    │   └── unmod.train.feature.tsv
    ├── ELIGOS_reference.fa
    └── model
           ├── m6A.demo.IVET.pkl
           └── m7G.demo.ELIGOS.transfered_from_IVET_m6A.pkl



Predict m6A sites in human cell lines
********************

HEK293T nanopore data is publicly available and can be downloaded from the `SG-NEx project <https://groups.google.com/g/sg-nex-updates>`_. In this demo, subset of the HEK293T nanopore data was taken for demonstration purposes due to the large size of the original datasets. The demo datasets were located under ``./demo/HEK293T/`` directory.
::
    demo
    └── HEK293T
        └── HEK293T_fast5
            └── HEK293T.fast5

**1. Guppy basecalling**

Basecalling converts the raw signal generated by Oxform Nanopore sequencing to DNA/RNA sequence. Guppy is used for basecalling in this step. In some nanopore datasets, the sequence information is already contained within the FAST5 files. In such cases, the basecalling step can be skipped as the sequence data is readily available.
::
    guppy_basecaller -i demo/HEK293T/HEK293T_fast5 -s demo/HEK293T/HEK293T_fast5_guppy --num_callers 40 --recursive --fast5_out --config rna_r9.4.1_70bps_hac.cfg
    

**2. Multi-reads FAST5 files to single-read FAST5 files**

Convert multi-reads FAST5 files to single-read FAST5 files. If the data generated by the sequencing device is already in the single-read format, this step can be skipped.
::
    multi_to_single_fast5 -i demo/HEK293T/HEK293T_fast5_guppy -s demo/HEK293T/HEK293T_fast5_guppy_single --recursive


**3. Tombo resquiggling**

In this step, the sequence obtained by basecalling is aligned or mapped to a reference genome or a known sequence. Then the corrected sequence is then associated with the corresponding current signals. The resquiggling process is typically performed in-plac. No separate files are generated in this step. GRCh38 transcripts file can be download `here <https://www.ncbi.nlm.nih.gov/datasets/genome/GCF_000001405.40/>`_. 
::
    tombo resquiggle --overwrite --basecall-group Basecall_1D_001 demo/HEK293T/HEK293T_fast5_guppy_single  demo/GRCh38_subset_reference.fa --processes 40 --fit-global-scale --include-event-stdev


**4. Map reads to reference**

minimap2 is used to map basecalled sequences to reference transcripts. The output sam file serves as the input for the subsequent feature extraction step. 
::
    cat demo/HEK293T/HEK293T_fast5_guppy/pass/*.fastq >demo/HEK293T/HEK293T.fastq
    minimap2 -ax map-ont demo/GRCh38_subset_reference.fa demo/HEK293T/HEK293T.fastq >demo/HEK293T/HEK293T.sam


**5. Feature extraction**

Extract signals and features from resquiggled fast5 files using the following python scripts.
::
    python scripts/extract_signal_from_fast5.py -p 40 --fast5 demo/HEK293T/HEK293T_fast5_guppy_single --reference demo/GRCh38_subset_reference.fa --sam demo/HEK293T/HEK293T.sam --output demo/HEK293T/HEK293T.signal.tsv --clip=10
    python scripts/extract_feature_from_signal.py  --signal_file demo/HEK293T/HEK293T.signal.tsv --clip 10 --output demo/HEK293T/HEK293T.feature.tsv --motif DRACH
    


In the feature extraction step, the motif pattern should be provided using the argument ``--motif``. The base symbols of the motif follow the IUB code standard. 


**7. Predict m6A sites**

To predict m6A sites in HEK293T nanopore data using a pretrained model, you can set the ``--run_mode`` argument to "predict".  You can specify the pretrained model by using the argument ``--pretrained_model``. 
::
    python scripts/TandemMod.py --run_mode predict \
          --pretrained_model demo/model/m6A.demo.IVET.pkl \
          --feature_file demo/HEK293T/HEK293T.feature.tsv \
          --predict_result demo/HEK293T/HEK293T.prediction.tsv


During the prediction process, TandemMod generates the following files. The prediction result file is named "HEK293T.prediction.tsv". 
::
    demo
    └── HEK293T
        ├── HEK293T_fast5
        ├── HEK293T_fast5_guppy
        ├── HEK293T_fast5_guppy_single
        ├── HEK293T.fastq
        ├── HEK293T.feature.tsv
        ├── HEK293T.prediction.tsv
        ├── HEK293T.sam
        └── HEK293T.signal.tsv

The prediction result "demo/HEK293T/HEK293T.prediction.tsv" provides prediction labels along with the corresponding modification probabilities, which can be utilized for further analysis.
::
    transcript_id   site    motif   read_id                                 prediction   probability
    XM_005261965.4  10156   AAACA   60e0f6a3-2166-4730-9a10-8f8aaa750b37    unmod        0.1364245
    XM_005261965.4  10164   AAACT   60e0f6a3-2166-4730-9a10-8f8aaa750b37    unmod        0.034915127
    XM_005261965.4  10229   GAACC   60e0f6a3-2166-4730-9a10-8f8aaa750b37    unmod        0.4773725
    XM_005261965.4  10241   GGACC   60e0f6a3-2166-4730-9a10-8f8aaa750b37    unmod        0.11096856
    XM_005261965.4  10324   GGACT   60e0f6a3-2166-4730-9a10-8f8aaa750b37    mod          0.908553
    XM_005261965.4  10362   AAACA   60e0f6a3-2166-4730-9a10-8f8aaa750b37    unmod        0.2004475
    XM_005261965.4  10434   AGACA   60e0f6a3-2166-4730-9a10-8f8aaa750b37    unmod        0.1934688
    XM_005261965.4  10498   GGACC   60e0f6a3-2166-4730-9a10-8f8aaa750b37    unmod        0.1313666
    XM_005261965.4  10507   AAACA   60e0f6a3-2166-4730-9a10-8f8aaa750b37    unmod        0.030169742
    XM_005261965.4  10511   AAACT   60e0f6a3-2166-4730-9a10-8f8aaa750b37    unmod        0.020174831
    XM_005261965.4  10592   AGACT   60e0f6a3-2166-4730-9a10-8f8aaa750b37    mod          0.7666112

The execution time for each demonstration is estimated to be approximately 3-10 minutes.
