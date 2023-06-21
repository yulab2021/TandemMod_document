
Usage
=====


Source code
********************

The source code and data processing scripts are available on `GitHub<https://github.com/yulab2021/TandemMod>`_. You can download them by using the git clone command::

    git clone https://github.com/yulab2021/TandemMod.git

TandemMod offers three modes: de novo training, transfer learning, and prediction. Researchers can train from scratch, fine-tune pre-trained models, or apply existing models for predictions. It provides a user-friendly solution for studying RNA modifications.

Training from scratch
********************
The de novo training mode in TandemMod enables users to train the model from scratch using their own datasets. To train a TandemMod model, both modified and modification-free Direct RNA Sequencing (DRS) data are required.

Before training, the raw FAST5 files need to undergo the `data processing procedure<data_preprocessing>` . This process generates feature files specific to each modification type. The feature files should follow a naming convention that reflects the modification type they represent::

    |-- data
    |   |-- unmodified.feature.tsv
    |   |-- m6A.feature.tsv


Where to find the results
-------------------------

Write me ...
