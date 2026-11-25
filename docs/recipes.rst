.. _recipes:

Recipes
#######

The `integration recipes <https://github.com/NOAA-PSL/ufs2arco/tree/main/tests/integration>`_
use these sections:

* ``mover``: execution, batching, and restarting.
* ``directories``: output, cache, and log paths.
* ``source``: input dataset and data selection.
* ``transforms``: operations applied to the selected data.
* ``target``: output layout and chunk sizes.
* ``attrs``: global metadata for the output dataset.

Use the integration recipes as examples. The sections below list the configurable
options; keep those supported by your chosen source, transforms, and target.
See :doc:`moving_data` for running recipes.

mover
=====

.. code-block:: yaml

   mover:
     name: datamover       # datamover (serial) or mpidatamover (MPI)
     batch_size: 1         # Required for serial; omit for MPI (one sample per rank)
     start: 0              # Zero-based batch index; nonzero resumes an existing store

If a job stops at its wall-clock limit, set ``start`` to the first unfinished
batch's zero-based index and rerun the same recipe to continue writing to the existing store.

.. _recipe-directories:

directories
===========

All three paths are required. Use dedicated paths for each conversion. Relative
paths use the working directory; environment variables expand, but ``~`` does not.

.. code-block:: yaml

   directories:
     zarr: output/dataset.zarr
     cache: cache/dataset
     logs: logs/dataset

.. _recipe-source:

source
======

Choose one registered ``name`` and its required selectors:

.. list-table::
   :header-rows: 1
   :widths: 40 30 30

   * - Name
     - Required options
     - Details
   * - ``aws_aorc``
     - ``time``
     - :doc:`aorc`
   * - ``conus404``
     - ``time``, ``uri``
     - :doc:`conus404`
   * - ``gcs_era5``
     - ``time``, ``uri``
     - :doc:`era5`
   * - ``aws_gefs_archive``
     - ``t0``, ``fhr``, ``member``
     - :doc:`gefs_archive`
   * - ``gfs_archive``
     - ``t0``, ``fhr``
     - :doc:`gfs_archive`
   * - ``aws_hrrr_archive``
     - ``t0``, ``fhr``
     - :doc:`hrrr_archive`
   * - ``aws_hrrr_archive_alaska``
     - ``t0``, ``fhr``
     - Alaska HRRR
   * - ``gcs_replay_atmosphere``
     - ``time``, ``uri``
     - :doc:`replay`
   * - ``gcs_replay_ocean``
     - ``time``, ``uri``
     - :doc:`replay`

For example, these are GFS selectors and the optional shared selection settings:

.. code-block:: yaml

   source:
     name: gfs_archive
     t0: {start: "2022-01-01T00", end: "2022-01-01T12", freq: 12h}
     fhr: {start: 0, end: 6, step: 6}
     variables: [t2m, t]              # Source variable names
     levels: [500, 850]              # Select existing 500 and 850 hPa pressure levels
     use_nearest_levels: false       # true selects the nearest available level
     slices:
       sel:                         # Coordinate bounds, inclusive
         latitude: [30, 50]
         longitude: [260, 290]
       # isel: {level: [0, 5]}       # Index bounds; stop excluded

``time``/``t0`` accepts `pandas.date_range options <https://pandas.pydata.org/docs/reference/api/pandas.date_range.html>`_
including ``start``, ``end``, ``freq``, ``periods``, and ``inclusive`` (default
``both``). Quote dates; each archive's available dates and cadence are listed
on its source page linked above.
``fhr``/``member`` requires integer ``start``, ``end``, ``step``; end is included
when reached. GEFS member ``0`` is the control. Anemoi requires one forecast hour.

``variables`` may be omitted to select all fields for cloud sources; set it
explicitly for GRIB sources. Omit ``levels`` to retain all levels: pressure
archives use hPa, Replay atmosphere native hybrid levels, Replay ocean meters,
and CONUS404 native indices. With ``use_nearest_levels: true``, a request for
840 hPa selects 850 hPa if that is the nearest available level. The default is
``false``, except for Replay ocean (``true``). Keep full columns for pressure/ocean
diagnostics. Slices accept ``[start, stop, step]`` and ``null`` bounds. Two-bound
latitude ``sel`` handles reversed coordinates; use horizontal ``isel`` for HRRR/CONUS404
and include staggered dimensions where needed (see :doc:`conus404`).

Additional source-specific keys (omit elsewhere):

.. code-block:: yaml

   # AORC, ERA5, Replay, CONUS404 (instead of t0/fhr):
   time: {start: "2017-01-01T00", end: "2017-01-02T18", freq: 6h}
   # ERA5, Replay atmosphere/ocean, CONUS404:
   uri: path/to/input.zarr           # Required input-store URI
   # ERA5 and Replay only:
   local: false                      # true opens a local store
   accumulate_variables: [tp]         # Sum source buckets over the output interval
   # GFS, GEFS, CONUS HRRR only:
   accum_hrs: {accum_tp: 6}           # Select a GRIB accumulation window in hours
   # GEFS only:
   member: {start: 0, end: 2, step: 1}
   # CONUS404 only:
   storage_options: {}               # Optional Azure overrides; default: signed STAC options

Accumulation names depend on the source. GRIB windows cannot exceed ``fhr.step``
(or the single selected forecast hour). Cloud accumulation requires regular
intervals; see :ref:`cloud-zarr-accumulation`.

For multiple sources, replace ``source`` with ``multisource:`` containing a list
of entries, each with ``source`` and optional ``transforms``/``target`` mappings;
see `replay.anemoi.yaml <https://github.com/NOAA-PSL/ufs2arco/blob/main/tests/integration/replay.anemoi.yaml>`_.
Use an Anemoi target, matching sample coordinates/batch counts and horizontal
grids, and distinct channel names. Common transforms override each entry's same
transform key; entry target values override common target values. Overrides
replace whole values, not nested keys. Only the first entry computes forcings.

.. _recipe-transforms:

transforms
==========

Optional. Keep only needed operations. They run in the order shown below,
regardless of YAML order. Example values illustrate syntax; comments give
defaults and requirements. Keep support fields selected until their transform.

.. code-block:: yaml

   transforms:
     destagger: {Z: level, U: longitude, V: latitude}  # CONUS404; horizontal needs xwrf
     multiply: {variable: 1000}      # Scalar or broadcastable array
     divide: {variable: 9.80665}     # Scaling does not update units
     ocean_density:                 # Replay ocean; defaults shown
       temperature: temp
       salinity: so
       name: rho
       ref_pressure: 0.0            # Pa; null for approximate in-situ pressure
       subtract_1000: false
       eos: wright                  # wright/wright_full (also null), or wright_red
     mixed_layer_depth:             # Replay ocean; defaults shown
       temperature: temp
       salinity: so
       thresholds: [0.03]           # Positive kg/m3; scalar or list
       names: null                  # Default mld; multiple thresholds need explicit names
       ref_pressure: 0.0            # Only 0 supported
       eos: wright
       pathological_to_nan: true
       mld_search_depth: 200.0       # m; these five keys validate the native grid
       min_levels_in_search: 10
       max_level_spacing: 25.0       # m
       expected_level_count: 75     # null disables count check
       minimum_deepest_level: 5000.0  # m; null disables depth check
     ocean_heat_content:            # Replay ocean; defaults shown
       temperature: temp
       depths: [700, 2000, null]     # m; null means local bottom
       names: null                  # Auto names; otherwise one name per depth
       interfaces: null             # Native boundaries; null reconstructs from centers
       rho0: 1035.0                  # kg/m3
       heat_capacity: 3991.86795711963  # J/(kg K)
       require_full_depth: true
       output_units: GJ m-2         # GJ m-2 or J m-2
     rotate_vectors:
       vector_pairs: [[u10, v10]]   # Required; grid-relative Lambert GRIB winds
     layers_to_pressure:            # Replay atmosphere or CONUS404
       variables: {tmp: t}          # Required input-to-output names
       levels_hpa: [500, 850]        # Required distinct positive integers <= 1100
       pressure_var: pressfc        # Surface pressure in Pa (PSFC for CONUS404)
       temperature_var: tmp         # Replay support: K
       humidity_var: spfh           # Replay support: kg/kg
       thickness_var: delz          # Replay support: negative layer thickness, m
       below_ground: mask           # Only mask supported
       level_name: level
       interface_name: phalf
       layer2pressure_support_var: null  # Set P for CONUS404 (3D pressure in Pa)
       keep_variables: []           # Retain named fields without a level dimension
     fv_vertical_regrid:
       weight_var: delz             # Required layer-weight variable
       interfaces: [1, 10, 20]      # Required increasing target boundaries
       use_nearest_interfaces: false
       keep_weight_var: false
       valid_mask: null             # Optional broadcastable boolean/array, not a variable name
     fv_vertical_regrid_ocean:       # Replay ocean; requires field named temp
       interfaces: [0, 100, 500]    # Required depth boundaries, m
       weight_var: dz
       use_nearest_interfaces: true
       keep_weight_var: false
       source_interfaces: null      # Optional native boundaries; overrides inferred ones
     horizontal_regrid:             # Requires xESMF and an existing target grid
       target_grid_path: /path/to/grid.nc  # Required; environment variables expand
       regridder_kwargs:            # Required; forwarded to xesmf.Regridder
         method: bilinear          # Required
         filename: /path/to/weights.nc  # Optional; default uses method/grid sizes
         reuse_weights: true       # Default true if file exists; false if absent
       open_target_kwargs: null     # Optional xarray.open_dataset keyword mapping
       source_is_on_gaussian_grid: false
     mappings:
       log: [variable]              # Also accepts one name; creates log_<name>
       round: [mask_variable]       # Creates round_<name>; removes original
     rename: {old_name: new_name}   # Variables/coordinates/dimensions, after other transforms

``layers_to_pressure`` must run separately from finite-volume regridding and
ocean diagnostics. Replay pressure interpolation needs its full native column;
CONUS404 uses ``P``/``PSFC`` instead. See :ref:`replay-pressure-levels` and
:ref:`conus404-pressure-levels`. Ocean diagnostics need native columns and
MLD/heat content need one complete vertical chunk; see :ref:`replay-ocean` for
units and grid requirements. Vertical averaging uses ``(lower, upper]`` bins;
retained weights are nominal sums, unaffected by ``valid_mask``.

``regridder_kwargs`` also accepts YAML-compatible `xESMF options <https://xesmf.readthedocs.io/en/stable/user_api.html#xesmf.frontend.Regridder>`_
(e.g. ``periodic``, ``unmapped_to_nan``, ``ignore_degenerate``, ``parallel``,
``extrap_method``, ``extrap_dist_exponent``, ``extrap_num_src_pnts``).
``open_target_kwargs`` accepts `xarray.open_dataset options <https://docs.xarray.dev/en/stable/generated/xarray.open_dataset.html>`_.
Use weights for the selected grids; filenames based only on grid sizes can collide.

.. _recipe-target:

target
======

Choose ``base`` (named dimensions/variables; aliases ``analysis``, ``forecast``),
``anemoi`` (packed training layout), or ``anemoi_inference_with_forcings``
(initial conditions plus future forcings). See :doc:`base_target` and
:doc:`anemoi_target`.

.. code-block:: yaml

   target:
     name: base                      # Default base
     chunks:                         # Required; dimension names after target.rename
       t0: 1
       fhr: 1
       level: -1                     # -1 means the entire dimension
       latitude: -1
       longitude: -1
     rename: {}                      # Optional old-to-new dimension/variable names
     forcings: []                    # Optional computed features, listed below

Keep sample chunks at ``1``: ``time`` for analyses, ``t0``/``fhr`` for forecasts,
plus ``member`` for GEFS. Anemoi uses
``chunks: {time: 1, variable: -1, ensemble: 1, cell: -1}``.
Chunks change storage, not data selection or grid resolution.

Additional Anemoi options (defaults shown):

.. code-block:: yaml

   statistics_period: {}             # Optional inclusive start/end strings; default full period
   compute_temporal_residual_statistics: false  # Statistics of adjacent-time differences
   sort_channels_by_levels: false    # Numeric level suffix ordering
   use_level_index: false            # Name levels by position instead of coordinate value
   channel_rename: {}                # Rename packed channels, e.g. {t_500: temperature_500}
   variables_with_nans: null          # Allowed channels; exact name, else case-insensitive substring
   resolution: null                  # Metadata only; does not regrid
   transformed_dims: {}              # Standard anemoi only; e.g. {latitude: y, longitude: x}
   multistep_input: 1                 # Inference-with-forcings only; input time count
   multisource_masks: []              # Optional mask rules below; mapping or list

Available ``forcings``:
``latitude``, ``cos_latitude``, ``sin_latitude``,
``longitude``, ``cos_longitude``, ``sin_longitude``,
``julian_day``, ``cos_julian_day``, ``sin_julian_day``,
``cos_local_time``, ``sin_local_time``, ``cos_solar_zenith_angle``, ``insolation``,
``cos_year_progress``, ``sin_year_progress``, ``cos_day_progress``, ``sin_day_progress``.

Mask rules use packed channel names after merging. Without ``static_mask``,
NaNs in ``mask_from`` determine masked cells per sample. To reuse a fitted mask,
add ``static_mask``; see :doc:`static_masks`.

.. code-block:: yaml

   multisource_masks:
     - mask_from: temp_0              # Required reference channel
       mask_variables: [temp_0]      # Default []; channels to mask
       output: ocean_mask            # Optional new mask channel
       land_value: 1                 # Default 1: encoded value for masked cells
       ocean_value: 0                # Default 0: encoded value for retained cells
       static_mask:                  # Optional; omit to use per-sample NaNs
         path: /path/to/mask.zarr     # Required existing artifact
         mode: reuse                 # Default reuse; only supported mode
         variable: temp_0             # Default mask_from
         method: any_valid           # Optional policy check: any_valid/all_valid/minimum_valid_fraction
         # minimum_valid_fraction: 0.9  # Required only with minimum_valid_fraction method

.. _recipe-attrs:

attrs
=====

Optional global metadata; arbitrary descriptive keys are accepted. Avoid
replacing generated recipe, missing-data, or target metadata.

.. code-block:: yaml

   attrs:
     description: Regional temperature dataset
     institution: NOAA
