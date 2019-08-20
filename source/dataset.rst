yews.datasets
====================

All datasets are subclasses of :class:`yews.datasets.BaseDataset`
i.e, they have ``__getitem__`` and ``__len__`` methods implemented.
Hence, they can all be passed to a `torch.utils.data.DataLoader`_
which can load multiple samples parallelly using ``torch.multiprocessing`` workers.
For example: ::

    waveform_data = yews.datasets.Folder('path/to/waveform_folder_root/')
    data_loader = torch.utils.data.DataLoader(waveform_data,
                                              batch_size=4,
                                              shuffle=True,
                                              num_workers=args.nThreads)

.. _`torch.utils.data.DataLoader`:
   https://pytorch.org/docs/stable/data.html#torch.utils.data.DataLoader

.. currentmodule:: yews.datasets

The following function can be used to verify if an object can be used as a
dataset:

.. autofunction:: is_dataset

The following datasets are available:

.. contents:: Datasets
    :local:

All the datasets have almost similar API. They all have two common arguments:
``transform`` and  ``target_transform`` to transform the input and target
respectively.

Standard Dataset
------------------
This is the default dataset used by ``yews`` that consists of two ``npy`` files
under a directory. The first ``samples.npy`` contains of the input data array
and the second ``targets.npy`` contains the output data array. Note that
additional files can exists in the same directy, but they will be ignored by
the ``Dataset`` class.

.. autoclass:: Dataset
   :special-members:


Basic Abstract Datasets
-----------------------
The ``BaseDataset`` class defines the common behavior of all ``dataset-like``
objects. Additional ``DirDataset`` and ``FileDataset`` add more constraints on
the specific form of the dataset. Note that ``Dataset`` class inherent from the
``DirDataset`` class.

.. autoclass:: BaseDataset
   :members: build_dataset, export_dataset
   :special-members:

.. autoclass:: DirDataset
   :special-members:

.. autoclass:: FileDataset
   :special-members:

Packaged Datasets
-----------------
Commonly used datasets are packaged and listed in this section. These datasets
can be downloaded directly.

.. autoclass:: Wenchuan
   :special-members:

.. autoclass:: SCSN
   :special-members:

.. autoclass:: Mariana
   :special-members:

