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


BaseDataset
-----------

.. autoclass:: BaseDataset
   :members: build_dataset
   :special-members:

DatasetArray
------------

.. autoclass:: DatasetArray
   :special-members:

DatasetFolder
-------------

.. autoclass:: DatasetFolder
   :special-members:

DatasetArrayFolder
------------------

.. autoclass:: DatasetArrayFolder
   :special-members:

Wenchuan
--------

.. autoclass:: Wenchuan
   :special-members:

Mariana (not published yet)
---------------------------

.. autoclass:: Mariana
   :special-members:

