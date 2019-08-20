Data Preparation
================

This tutorial gives examples to prepare ``yews.Dataset`` from common seismic
dataets.

There are two steps for preparing a dataset: (i) retrieve phase information;
and (ii) retrieve waveform data.
Phase information can be retrieved from a catalog, an index file, or each
waveform files.
Waveform data can be directly loaded from cutted files or cutted from the
continuous waveforms.

Below are several typical use cases that ``yews`` provides tools to automate
the dataset preparation process.
Additional tools will be added by requrest in the future.
The tools from ``yews``  and other commond tools can be loaded by following
imports ::

    from pathlib import Path

    import numpy as np
    import pandas as pd
    from obspy import read
    from obspy import UTCDateTime as utc
    from pandas import DataFrame

    from yews.datasets.utils import get_files_under_dir
    from yews.datasets.utils import read_frame_obspy


Discrete Waveforms
------------------

Many online databases allow downloading discrete waveforms centered around the
picked arrival times.
This example uses a dataset from OKGS that consists of four-minute-long
discrete waveforms around the target arrival times.
P-wave, S-wave, and noise samples will be cutted to constructure a dataset for
training the `CPIC <https://www.sciencedirect.com/science/article/pii/S0031920118301407>`_
model.

The file names of these discrete waveforms contains all the information about
seismic phases.
Thus, we can retrieve the phase inforamtion by processing all the file names.
::

    # retrieve trace table
    files = get_files_under_dir('/data/ok_new','**/*.SAC')
    info = map(retrieve_info_from_path, files)
    trace_table = DataFrame(info)

    # retrive event table
    events = trace_table[['id','origin','latitude','longitude', 'depth',
                               'magnitude']].drop_duplicates('id')
    events.set_index('id', inplace=True, verify_integrity=True)
    events.to_csv('events.csv')

    # retrive phase table
    phases = trace_table[['id','phase','arrival','station', 'path']].drop_duplicates(subset=['id', 'phase', 'arrival', 'station'])
    phases['path'] = phases[['path']].applymap(path2pattern)
    phases.reset_index(drop=True, inplace=True)
    phases.to_csv('phases.csv')

    # retrive station table
    stations = trace_table[['station', 'path']].drop_duplicates('station')
    station_info = map(retrieve_station_info, stations['path'])
    stations = DataFrame(station_info)
    stations.set_index('station', inplace=True, verify_integrity=True)
    stations.to_csv('stations.csv')

Here, we use ``pandas.DataFrame`` to process the table retrieved from the files
names. It is also convenient to save the processed tables into ``.csv`` files.
They can be loaded for future usage as following ::

    events = pd.read_csv(paths[0], index_col='id')
    phases = pd.read_csv(paths[1])
    events = pd.read_csv(paths[2], index_col='station')

The three tables retrieved from above code contains most information needed of
a given datasets.
The ``phases`` table is particular important for retrieving the waveform data.
Each row of the ``phases`` table corresponds to a four-minute-long waveform,
from which we will cut one phase data and one noise data.
Here, we assume one minute before P-wave or one-minute after S-wave is the
quite regions which only contains noise waveform.
Thus, we can get the phase and noise waveform by the following code ::

    try:
        data = read_frame_obspy(str(row['path']))
    except ValueError:
        print(f"Waveform #{index} is broken. Skipped.")
        continue
    if data.shape != (3, 9600):
        print(f"Phase #{index} is invalid.")
        continue # skip broken data
    phase = row['phase']
    # 5 second before and 15 second after phase arrival
    samples_list.append(data[:, 4600:5400])
    targets_list.append([phase, index])
    # one minute before P or one minute after S
    if phase == 'P':
        samples_list.append(data[:, 1600:2400])
        targets_list.append(['N', index])
    elif phase == 'S':
        samples_list.append(data[:, 7800:8600])
        targets_list.append(['N', index])
    else:
        print(f"{index} has a invalid phase {phase}.")
        continue # skip unknown phases

In many cases, the dataset is too large to fit into the memory.
It needs to be broken into piece and later merged into one ``npy`` files.
This can be achieved by defining a ``group_size`` limit to each processing
block as following::

    group_size = 100000
    for index, row in phases.iterrows():
        if index % group_size == 0:
            print(index)
            # save previous group
            try:
                samples = np.stack(samples_list)
                targets = np.stack(targets_list)
                np.save(f'samples{index}.npy', samples)
                np.save(f'targets{index}.npy', targets)
            except NameError:
                pass
            # initialized new group
            samples_list = []
            targets_list = []
        try:
            data = read_frame_obspy(str(row['path']))
        except ValueError:
            print(f"Waveform #{index} is broken. Skipped.")
            continue
        if data.shape != (3, 9600):
            print(f"Phase #{index} is invalid.")
            continue # skip broken data
        phase = row['phase']
        # 5 second before and 15 second after phase arrival
        samples_list.append(data[:, 4600:5400])
        targets_list.append([phase, index])
        # one minute before P or one minute after S
        if phase == 'P':
            samples_list.append(data[:, 1600:2400])
            targets_list.append(['N', index])
        elif phase == 'S':
            samples_list.append(data[:, 7800:8600])
            targets_list.append(['N', index])
        else:
            print(f"{index} has a invalid phase {phase}.")
            continue # skip unknown phases

    samples = np.stack(samples_list)
    targets = np.stack(targets_list)
    np.save(f'samples{index+1}.npy', samples)
    np.save(f'targets{index+1}.npy', targets)

When combining the partial ``npy`` files, it is usually small enough to fit
into the memory ::

    sample_names = [str(p) for p in Path('.').glob('sample*.npy')]
    sample_names.sort(key=lambda f: int(''.join(filter(str.isdigit, f))))
    target_names = [str(p) for p in Path('.').glob('target*.npy')]
    target_names.sort(key=lambda f: int(''.join(filter(str.isdigit, f))))

    samples = np.concatenate(list(map(np.load, sample_names)), axis=0)
    targets = np.concatenate(list(map(np.load, target_names)), axis=0)

    # remove temp files
    [p.unlink() for p in Path('.').glob('samples*.npy')]
    [p.unlink() for p in Path('.').glob('targets*.npy')]

    # create new files
    np.save('samples.npy', samples)
    np.save('targets.npy', targets)

However, if the entire cutted dataset is still much larger than the memory
size, you can use the following code instead ::

    # create a memmap on disk to store the large dataset
    total = yews.datasets.utils.create_npy('combined.npy', shape, dtype)
    total[:num_first_file, :, :] = np.load('first.npy', mmap_mode='r')

    # continue doing it for all npy files, then flush and del the memmap
    # object.
    total.flush()
    del total


Continuous Waveforms
--------------------
