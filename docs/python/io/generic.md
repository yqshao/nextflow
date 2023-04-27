# Generic information

The IO module in TIPS allows for the translation between different atomistic
data formats, with a special focus for atomistic machine learning tasks.

## Available formats

| Format      | Read               | Write              | Note                           |
|-------------|--------------------|--------------------|--------------------------------|
| ase/asetraj | :white_check_mark: | :white_check_mark: | ASE Trajectory obj or files    |
| cp2k        | :white_check_mark: | :no_entry_sign:    | CP2K data (pos, frc, and cell) |
| deepmd      | :white_check_mark: | :no_entry_sign:    | DeePMD format                  |
| extxyz      | :no_entry_sign:    | :white_check_mark: | Extended XYZ format            |
| lammps      | :white_check_mark: | :no_entry_sign:    | LAMMPS dump format             |
| pinn        | :white_check_mark: | :white_check_mark: | PiNN-style TFRecord format     |
| runner      | :no_entry_sign:    | :white_check_mark: | RuNNer format                  |


## Units and formats

TIPS uses a unit system compatible to ASE internally, that is:

- energy in eV
- length in Å

Some formats does not have a fixed unit system, or a different unit standard,
those are documented in the format-specific documentations.

## The `load_ds` funciton

The `load_ds` function is a universal entry point for dataset loaders in TIPS.

=== "Usage"

    ```Python
    from tips.io import load_ds

    ds = load_ds('path/to/dataset', fmt=deepmd-raw')
    ds = ds.join(ds) # datasets can be joined together
    ds.convert('dataset.yml', fmt='pinn') # the and converted to different formats
    print(ds)
    ```

=== "Output"

    ```Python
    # printing the  dataset shows basic information about the dataset
    <tips.io.Dataset:
     fmt: DeePMD raw
     size: 100
     elem: 8, 1
     spec:
         cell: 3x3, float
         elem: [None], int
         force: [None, 3], float
         coord: [None, 3], float
         energy: [], float>
    ```

The function returns a `Dataset`-class object, its usage is detailed below.

## The `Dataset` class

::: tips.io.dataset.Dataset
    options:
      heading_level: 3

## Custom reader/writer

It is possible to extend TIPS by registering extra reader/writers, an example
for custom reader/writer can be found below:

``` python
from tips.io.utils import tips_reader, tips_convert

@tips_reader('my-ase')
def load_ase(traj):
    """ An example reader for ASE Atoms

    The function should return a tips Dataset, by specifying at least an generator which
    yields elements in the dataset one by one, and the metadata specifying the data structure.
    (the generator is redundent ni the below case because an ASE trajectory is indexable
    and has a defined size, such a generator will be defined automatically by tips)

    Args:
        traj: list of atoms

    Returns:
        tips.io.Dataset
    """
    from tips.io import Dataset
    meta = {
      'spec': {
        'elems': {'shape': [None], 'dtype': 'int32'},
        'coord': {'shape': [None, 3], 'dtype': 'float32'},
        'cell': {'shape': [3, 3], 'dtype': 'float32'}
      },
      'size': len(traj),
      'fmt': 'Custom ASE Format'
    }

    def indexer(i):
        atoms = traj[i]
        data = {
           'elems': atoms.numbers
           'coord': atoms.positions,
           'cell': atoms.cell,
        }
        return data

    def generator():
        for i in range(meta['size']):
            yield indexer(i)

    return Dataset(generator=generator, meta=meta, indexer=indexer)


@tips_convert('my-ase')
def ds_to_ase(dataset):
    """ An example data converter to ASE trajectory

    The function must takes on dataset and optionally extra keyword arguments as inputs.
    There is no limitaton on the return values.

    Args:
        dataset (tips.io.Dataset): a dataset object
    """
    from ase import Atoms
    traj = [
       Atoms(data['elems'],
             positions=data['coord'],
             cell=data['cell'])
       for data in dataset
    ]
    return  traj
```

The additonal format will be available for data loading and conversion:

``` python
ds = load_ds([Atoms['H'], Atoms['Cu']], fmt='my-ase')
traj = ds.convert(fmt='my-ase')
```

