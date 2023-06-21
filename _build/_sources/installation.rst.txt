.. _installation:

Installation
==================================
The following modules are needed to run TandemMod.


.. list-table:: Required modules
   :widths: 50 50
   :header-rows: 1

   * - samtools
     - 1.3.1
   * - minimap2
     - 2.17-r941
   * - python 
     - 3.7.3
   * - h5py
     - 2.9.0
   * - statsmodels
     - 0.10.0
   * - joblib 
     - 0.16.0
   * - pysam
     - 0.16.0.1
   * - tqdm
     - 4.39.0
   * - scikit-learn
     - 0.22
   * - torch
     - 1.9.1
   * - scipy
     - 1.6.3
   * - guppy
     - 6.1.5
   * - tombo
     - 1.5.1
   * - ont_vbz_hdf_plugin
     - 1.0.1
   * - ont-fast5-api
     - 4.1.1

Conda is recommended for package management, you can create a new conda environment and then install the packages. Here's an example of how you can do it. Create a new conda environment::
    
    conda create -n tandemmod_env python=3.7.3

Activate the newly created environment::

    conda activate tandemmod_env

Install the required modules::

    conda install -c bioconda samtools=1.3.1
    conda install -c bioconda minimap2=2.17
    conda install -c anaconda h5py=2.9.0
    conda install -c conda-forge statsmodels=0.10.0
    conda install -c conda-forge joblib=0.16.0
    conda install -c bioconda pysam=0.16.0.1
    conda install -c conda-forge tqdm=4.39.0
    conda install -c anaconda scikit-learn=0.22
    conda install -c pytorch torch=1.9.1
    conda install -c anaconda scipy=1.6.3
    conda install -c bioconda ont-tombo=1.5.1
    conda install -c bioconda ont_vbz_hdf_plugin=1.0.1
    conda install -c bioconda ont-fast5-api=4.1.1

Or, you can use pip to install the modules::

    pip install samtools==1.3.1
    pip install minimap2==2.17-r941
    pip install h5py==2.9.0
    pip install statsmodels==0.10.0
    pip install joblib==0.16.0
    pip install pysam==0.16.0.1
    pip install tqdm==4.39.0
    pip install scikit-learn==0.22
    pip install torch==1.9.1
    pip install scipy==1.6.3
    pip install ont-tombo==1.5.1
    pip install ont_vbz_hdf_plugin==1.0.1
    pip install ont-fast5-api==4.1.1

Guppy can be obtained from `Oxford Nanopore Technologies <https://nanoporetech.com/>`_ or from this `mirror <https://mirror.oxfordnanoportal.com/software/analysis/ont-guppy-cpu-6.1.5-1.el7.x86_64.rpm>`_. Install Guppy using dpkg::

    alien ont-guppy-cpu-6.1.5-1.el7.x86_64.rpm
    dpkg -i ont-guppy-cpu-6.1.5-1.el7.x86_64.deb

``libhdf5`` and ``libcrypto`` are required for running guppy.


We have also provided a yaml file in the repository so you can install dependent through the configuration file::

    conda env create -f TandemMod.yaml


