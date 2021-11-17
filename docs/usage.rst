=====
Installation/Usage
=====


To install :code:`msm_we`:

.. code-block:: bash

    cd </path/to/msm_we>

    conda env create -f environment.yml
    conda activate hamsm_env
    pip install .

.. highlight:: python

To use :code:`msm_we` in a project::

    from msm_we import msm_we


The basic workflow of this module is as follows.

Model building and preparation
------------------------------

1. Run a weighted ensemble simulation, and produce a :code:`west.h5` file.

2. Augment :code:`west.h5` with the full coordinate information in each :code:`<iterXXX/auxdata/coord>`.

    :code:`msm_we/scripts/collectCoordinates/collectCoordinates.py` has an example of this.
    This functionality will likely eventually be rolled into the main package.

3. Define the :code:`processCoordinates(self, coords)` function and monkey-patch it in.

    This function is responsible for featurization. It should take in an array of all coordinates,
    and produce an array of feature-coordinates.

    ::

        # A trivial example processCoordinates
        def processCoordinates(self, coords):
            # Do some stuff in here
            # This is NOT a complete example!
            return coords

        # Monkey-patch, i.e. replace the placeholder processCoordinates
        #   in msm_we.modelWE with your function
        msm_we.modelWE.processCoordinates = processCoordinates


    It's important to do this monkey-patching at the module level, i.e. on :code:`msm_we.modelWE`, rather
    than on an instance of a :code:`msm_we.modelWE` object.

4. Create the model object.

    .. code-block:: python

        model = msm_we.modelWE()

5. Initialize the model.

    .. code-block:: python

        model.initialize(file_paths, reference_structure_file, model_name,
                        basis_pcoord_bounds, target_pcoord_bounds,
                        dim_reduce_method, tau, pcoord_ndim)


    :code:`file_paths` is a list of paths to your WESTPA h5 files.

    :code:`reference_structure_file` is a file containing a topology describing your system.

    :code:`model_name` is what it sounds like, and is used to label various output files.

    :code:`basis_pcoord1_bounds` is a list of [[pcoord0 lower bound, pcoord1 upper bound], [pcoord1 lower bound, pcoord1 upper bound], ...]
            in pcoord-space for the basis state

    :code:`target_pcoord1_bounds` is a list of [[pcoord0 lower bound, pcoord1 upper bound], [pcoord1 lower bound, pcoord1 upper bound], ...]
            in pcoord-space for the target state

    :code:`dim_reduce_method` is the dimensionality reduction method ("pca", "vamp", or "none")

    :code:`tau` is the resampling time, or the length of one WE iteration in physical units.

    :code:`pcoord_ndim` is the dimensionality of the progress coordinate.

6. Load all coords and pcoords up to the last iteration you want to use for analysis with

    .. code-block:: python

        model.get_iterations()
        model.get_coordSet(last_iter)

    where `last_iter` is the number of iterations you have (AKA, the last iteration it'll load data from.)

7. Prepare dimensionality reduction transformer by running

    .. code-block:: python

        model.dimReduce()

8. Do clustering with

    .. code-block:: python

        model.cluster_coordinates(n_clusters)

9. Create the flux matrix with

    .. code-block:: python

        model.get_fluxMatrix(lag, first_iter, last_iter)

    a. Clean disconnected states and sort the flux matrix with

    .. code-block:: python

        model.organize_fluxMatrix()

Analysis
--------

10. Normalize the flux matrix to produce a transition matrix with

    .. code-block:: python

        model.get_Tmatrix()

11. Obtain steady-state distribution with

    .. code-block:: python

        model.get_steady_state()

    Note: This may fail or encounter difficulties for datasets where no target flux has been obtained.
    This can happen with either incomplete sampling to your target state, or with equilibrium data.
    This is because it uses the flux estimate as a convergence criterion.
    If the flux is 0, then it's not meaningful to  look at convergence of 0, so it'll just run
    for the maximum number of iterations. You can specify :code:`max_iters=1` to avoid unnecessary
    iteration, or you can use :code:`model.get_steady_state_algebraic`.

12. Update cluster structures

    .. code-block:: python

        model.update_cluster_structures()

13. Obtain steady-state target flux with

    .. code-block:: python

        model.get_steady_state_target_flux()

Streaming
---------

:code:`msm_we` supports streaming dimensionality reduction and clustering when dimensionality reduction is
done through PCA or not done.

Streaming dimensionality reduction is automatically done for PCA.

To use streaming clustering, pass :code:`streaming=True` to :code:`cluster_coordinates()`.

Streaming is not supported for VAMP, because I don't know of a streaming implementation of VAMP dimensionality reduction.
