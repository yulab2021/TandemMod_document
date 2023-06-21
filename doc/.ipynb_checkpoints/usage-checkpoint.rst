
Usage
=====


Source code
********************

The source code and data processing scripts are available on `GitHub <https://github.com/yulab2021/TandemMod>`_. You can download them by using the git clone command::

    git clone https://github.com/yulab2021/TandemMod.git

TandemMod offers three modes: de novo training, transfer learning, and prediction. Researchers can train from scratch, fine-tune pre-trained models, or apply existing models for predictions. It provides a user-friendly solution for studying RNA modifications.

Training from scratch
********************
The de novo training mode in TandemMod enables users to train the model from scratch using their own datasets. To train a TandemMod model, both modified and modification-free Direct RNA Sequencing (DRS) data are required.

Before training, the raw FAST5 files need to undergo the `data processing procedure <data_preprocessing>`_ . This process generates feature files specific to each modification type. The feature files should follow a naming convention that reflects the modification type they represent::

    |-- data
    |   |-- A.feature.tsv
    |   |-- m6A.feature.tsv

In order to evaluate the performance during the training process, it is important to have a separate test dataset. Here's a script that randomly splits the feature file into training and test sets::

    python scripts/train_test_split.py --input_file A.feature.tsv --train_file A_train.feature.tsv --test_file A_test.feature.tsv --train_ratio 0.8
    python scripts/train_test_split.py --input_file m6A.feature.tsv --train_file m6A_train.feature.tsv --test_file m6A_test.feature.tsv --train_ratio 0.8

To train the TandemMod model using both the training dataset, you can set the ``-run_mode`` argument to "train". This allows the model to be trained from scratch. Test data are required to evaluation the model performance.

    python scripts/TandemMod.py -run_mode train \
          -new_model model/m6A.pkl \
          -train_data_mod data/m6A_train.feature.tsv \
          -train_data_unmod data/A_train.feature.tsv \
          -test_data_mod data/m6A_test.feature.tsv \
          -test_data_unmod data/A_test.feature.tsv 
          -epoch 100

The training process can be stopped manually based on the performance on the test set or by setting the maximum number of epochs. You can monitor the performance of the model on the test set during training and decide when to stop based on your desired criteria, such as reaching a certain accuracy or loss threshold. Alternatively, you can set a specific number of epochs as the maximum value for training using the ``-epoch`` argument. This allows the model to train for a fixed number of iterations, regardless of the performance on the test set. After the specified number of epochs, the training process will automatically stop. By providing these options, you have the flexibility to control the training process based on your specific requirements and preferences. The training process should be something like this::
    
    Epoch 2-2 Train acc: 0.853227, Test Acc: 0.801561, time: 0.684026
    Epoch 2-3 Train acc: 0.857492, Test Acc: 0.809284, time: 0.689912
    Epoch 2-4 Train acc: 0.859884, Test Acc: 0.810469, time: 0.695631
    Epoch 2-5 Train acc: 0.863527, Test Acc: 0.812851, time: 0.701268
    Epoch 2-6 Train acc: 0.865912, Test Acc: 0.814036, time: 0.701268


Transfer learning
********************
In transfer learning mode, you can used a pretrained model to retrain the bottom layers to identify new modification. This mode leverages the knowledge acquired by the pretrained model on a large dataset and applies it to a specific target task with potentially limited data. To fine-tune the TandemMod model using other dataset, you can set the ``-run_mode`` argument to "transfer".


Prediction
********************