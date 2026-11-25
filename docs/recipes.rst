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

Use an integration recipe as a starting point, then use the options below to
adjust it for your data. Some options apply only to particular sources or targets.
See :doc:`moving_data` for how to run a recipe.

mover
=====

.. code-block:: yaml

   mover:
     name: datamover               # datamover (serial) or mpidatamover (MPI)
     batch_size: 1                 # Required for serial; omit for MPI (one sample per rank)
     start: 0                      # Zero-based batch index; nonzero resumes an existing store

If your job runs out of wall-clock time, set ``start`` to the first unfinished
batch and rerun the same recipe to continue from there. Batch numbering starts at 0.

.. _recipe-directories:

directories
===========

Set all three paths, using separate directories for each conversion. Relative
paths start from the directory where you run the command. You can use environment
variables in these paths, but ``~`` is not expanded.

.. code-block:: yaml

   directories:
     zarr: output/dataset.zarr
     cache: cache/dataset
     logs: logs/dataset

.. _recipe-source:

source
======

Set ``name`` to one of the sources below and provide its required options:

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

For example, this selects GFS temperature at two pressure levels over a region:

.. code-block:: yaml

   source:
     name: gfs_archive
     t0: {start: "2022-01-01T00", end: "2022-01-01T12", freq: 12h}
     fhr: {start: 0, end: 6, step: 6}
     variables: [t2m, t]           # Source variable names
     levels: [500, 850]            # Select existing 500 and 850 hPa pressure levels
     use_nearest_levels: false     # true selects the nearest available level
     slices:
       sel:                        # Coordinate bounds, inclusive
         latitude: [30, 50]
         longitude: [260, 290]
       # isel: {level: [0, 5]}       # Index bounds; stop excluded

``time``/``t0`` accepts `pandas.date_range options <https://pandas.pydata.org/docs/reference/api/pandas.date_range.html>`_
including ``start``, ``end``, ``freq``, ``periods``, and ``inclusive`` (default
``both``). Put dates in quotes. The source pages linked above list each archive's
available dates and time intervals.

For ``fhr`` and ``member``, provide integer ``start``, ``end``, and ``step`` values.
The end value is included if the step reaches it. GEFS member ``0`` is the control
member. Anemoi does not currently support multiple forecast hours in one source.
Use initial conditions (``fhr: {start: 0, end: 0, step: 1}``) or a single lead
time, such as ``fhr: {start: 6, end: 6, step: 1}``.

List the source fields you want in ``variables``. Cloud sources read all fields
if you leave it out; GRIB sources need an explicit list. Leave out ``levels`` to
keep all available levels. Level values are in hPa for pressure archives, native
hybrid levels for Replay atmosphere, meters for Replay ocean, and native layer
indices for CONUS404. With ``use_nearest_levels: true``, a request for
840 hPa selects 850 hPa if that is the nearest available level. The default is
``false``, except for Replay ocean (``true``). Keep full columns for pressure/ocean
diagnostics.

Slices also accept ``[start, stop, step]`` and ``null`` for an open bound.
Latitude ``sel`` slices with two bounds work with ascending or descending
coordinates. Use horizontal ``isel`` slices for HRRR and CONUS404, including
staggered dimensions where needed for WRF-based datasets (see :doc:`conus404`).

The following options apply only to the sources noted in the comments:

.. code-block:: yaml

   # AORC, ERA5, Replay, CONUS404 (instead of t0/fhr):
   time: {start: "2017-01-01T00", end: "2017-01-02T18", freq: 6h}
   # ERA5, Replay atmosphere/ocean, CONUS404:
   uri: path/to/input.zarr         # Required input-store URI
   # ERA5 and Replay only:
   local: false                    # true opens a local store
   accumulate_variables: [tp]      # Sum source buckets over the output interval
   # GFS, GEFS, CONUS HRRR only:
   accum_hrs: {accum_tp: 6}        # Select a GRIB accumulation window in hours
   # GEFS only:
   member: {start: 0, end: 2, step: 1}
   # CONUS404 only:
   storage_options: {}             # Optional Azure overrides; default: signed STAC options

Accumulation names depend on the source. GRIB windows cannot exceed ``fhr.step``
(or the single selected forecast hour). Cloud accumulation requires regular
intervals; see :ref:`cloud-zarr-accumulation`.

``multisource`` combines fields from several source entries into one Anemoi
dataset. The entries can read the same archive with different variables or
transforms, or read different archives:

* Use separate entries for one archive when groups of variables need different
  processing. For example, `replay.anemoi.yaml <https://github.com/NOAA-PSL/ufs2arco/blob/main/tests/integration/replay.anemoi.yaml>`_
  applies a log transform to humidity in one entry and handles temperature in
  another.
* To combine different archives, such as Replay atmosphere and Replay ocean,
  bring their fields onto the same horizontal grid before merging. Matching
  resolution alone is not enough: coordinates and cell ordering must also match.

Replace the top-level ``source`` with a ``multisource`` list. Each entry has its
own ``source`` and may have ``transforms`` and ``target`` settings. The rules are:

* Use ``anemoi`` or ``anemoi_inference_with_forcings`` as the target.
* All entries must use the same sample dimensions and values, with the same
  number of batches. For example, two Replay entries must select the same times.
  Forecast entries must also match ``t0``, ``fhr``, and ``member`` where present.
* Give each output channel a unique name, using per-entry renaming if needed.
* Top-level transforms override the same transform in an entry. An entry's
  target settings override the top-level target settings. Each override replaces
  the whole value, including nested options.
* Forcings are computed for the first entry only.

The current driver does not align different initialization times or forecast
hours. Combining initial-condition fields with precipitation from a previous
initialization's six-hour forecast is therefore not supported. GFS and ERA5
also cannot be combined directly: GFS uses ``t0``/``fhr`` sample dimensions,
while ERA5 uses ``time``.

.. _recipe-transforms:

transforms
==========

Add the transforms you need from the options below. They run in the order shown,
regardless of their order in the YAML file. The comments identify defaults and
required settings. For example, CONUS404 pressure interpolation needs ``P`` and
``PSFC`` in ``source.variables`` as well as the fields you want to interpolate.

.. code-block:: yaml

   transforms:
     destagger: {Z: level, U: longitude, V: latitude}  # Currently only applicable to CONUS404
     multiply: {variable: 1000}                        # Scalar or broadcastable array
     divide: {variable: 9.80665}                       # Scaling does not update units
     ocean_density:                                    # Replay ocean; defaults shown
       temperature: temp
       salinity: so
       name: rho
       ref_pressure: 0.0                               # Pa; null for approximate in-situ pressure
       subtract_1000: false
       eos: wright                                     # wright/wright_full (also null), or wright_red
     mixed_layer_depth:                                # Replay ocean; defaults shown
       temperature: temp
       salinity: so
       thresholds: [0.03]                              # Positive kg/m3; scalar or list
       names: null                                     # Default mld; multiple thresholds need explicit names
       ref_pressure: 0.0                               # Only 0 supported
       eos: wright
       pathological_to_nan: true
       mld_search_depth: 200.0                         # m; these five keys validate the native grid
       min_levels_in_search: 10
       max_level_spacing: 25.0                         # m
       expected_level_count: 75                        # null disables count check
       minimum_deepest_level: 5000.0                   # m; null disables depth check
     ocean_heat_content:                               # Replay ocean; defaults shown
       temperature: temp
       depths: [700, 2000, null]                       # m; null means local bottom
       names: null                                     # Auto names; otherwise one name per depth
       interfaces: null                                # Native boundaries; null reconstructs from centers
       rho0: 1035.0                                    # kg/m3
       heat_capacity: 3991.86795711963                 # J/(kg K)
       require_full_depth: true
       output_units: GJ m-2                            # GJ m-2 or J m-2
     rotate_vectors:
       vector_pairs: [[u10, v10]]                      # Required; grid-relative Lambert GRIB winds
     layers_to_pressure:                               # Replay atmosphere or CONUS404
       variables: {tmp: t}                             # Required input-to-output names
       levels_hpa: [500, 850]                          # Required distinct positive integers <= 1100
       pressure_var: pressfc                           # Surface pressure in Pa (PSFC for CONUS404)
       temperature_var: tmp                            # Replay support: K
       humidity_var: spfh                              # Replay support: kg/kg
       thickness_var: delz                             # Replay support: negative layer thickness, m
       below_ground: mask                              # Only mask supported
       level_name: level
       interface_name: phalf
       layer2pressure_support_var: null                # Set P for CONUS404 (3D pressure in Pa)
       keep_variables: []                              # Use [PSFC] to keep CONUS404 surface pressure
     fv_vertical_regrid:
       weight_var: delz                                # Required layer-weight variable
       interfaces: [1, 10, 20]                         # Required increasing target boundaries
       use_nearest_interfaces: false
       keep_weight_var: false
       valid_mask: null                                # Optional broadcastable boolean/array, not a variable name
     fv_vertical_regrid_ocean:                         # Replay ocean; requires field named temp
       interfaces: [0, 100, 500]                       # Required depth boundaries, m
       weight_var: dz
       use_nearest_interfaces: true
       keep_weight_var: false
       source_interfaces: null                         # Optional native boundaries; overrides inferred ones
     horizontal_regrid:                                # Requires xESMF and an existing target grid
       target_grid_path: /path/to/grid.nc              # Required; environment variables expand
       regridder_kwargs:                               # Required; forwarded to xesmf.Regridder
         method: bilinear                              # Required
         filename: /path/to/weights.nc                 # Optional; default uses method/grid sizes
         reuse_weights: true                           # Default true if file exists; false if absent
       open_target_kwargs: null                        # Optional xarray.open_dataset keyword mapping
       source_is_on_gaussian_grid: false
     mappings:
       log: [variable]                                 # Also accepts one name; creates log_<name>
       round: [mask_variable]                          # Creates round_<name>; removes original
     rename: {old_name: new_name}                      # Variables/coordinates/dimensions, after other transforms

``layers_to_pressure`` interpolates fields to specific pressure levels, such as
500 and 850 hPa, using linear interpolation in log-pressure. It does not average
layers. Replay needs its full native column and the listed support variables;
CONUS404 uses ``P`` and ``PSFC``. To keep CONUS404 surface pressure in the output,
set ``keep_variables: [PSFC]``. Otherwise, ``PSFC`` is used for interpolation and
then dropped. See :ref:`replay-pressure-levels` and :ref:`conus404-pressure-levels`.

``fv_vertical_regrid`` instead combines native layers into broader layers bounded
by ``interfaces``. It computes a weighted average in each layer using
``weight_var`` (for example, ``delz``), and labels the result with the boundary
midpoint. It does not interpolate to pressure surfaces. The ocean version uses
native layer thicknesses and accounts for the wet water column. Bins use
``(lower, upper]`` boundaries; retained weights are nominal sums, unaffected by
``valid_mask``.

Use separate source entries for pressure interpolation and finite-volume
regridding or ocean diagnostics. Ocean diagnostics need native columns, and
MLD/heat content need one complete vertical chunk. See :ref:`replay-ocean` for
units and grid requirements.

``regridder_kwargs`` also accepts YAML-compatible `xESMF options <https://xesmf.readthedocs.io/en/stable/user_api.html#xesmf.frontend.Regridder>`_
(e.g. ``periodic``, ``unmapped_to_nan``, ``ignore_degenerate``, ``parallel``,
``extrap_method``, ``extrap_dist_exponent``, ``extrap_num_src_pnts``).
``open_target_kwargs`` accepts `xarray.open_dataset options <https://docs.xarray.dev/en/stable/generated/xarray.open_dataset.html>`_.
The default weights filename contains the method and grid sizes, not the
coordinates. Two grids with the same number of points but different locations
can therefore produce the same filename. Reusing that file would apply weights
for the wrong grid. Set a different ``filename`` for each pair of source and
target grids.

.. _recipe-target:

target
======

Choose ``base`` for named variables and dimensions, ``anemoi`` for the packed
training layout, or ``anemoi_inference_with_forcings`` for initial conditions
and future forcings. The names ``analysis`` and ``forecast`` are aliases for
``base``. See :doc:`base_target` and :doc:`anemoi_target` for details.

.. code-block:: yaml

   target:
     name: base                    # Default base
     chunks:                       # Required; dimension names after target.rename
       t0: 1
       fhr: 1
       level: -1                   # -1 means the entire dimension
       latitude: -1
       longitude: -1
     rename: {}                    # Optional old-to-new dimension/variable names
     forcings: []                  # Optional computed features, listed below

Keep sample chunks at ``1``: ``time`` for analyses, ``t0``/``fhr`` for forecasts,
plus ``member`` for GEFS. Anemoi uses
``chunks: {time: 1, variable: -1, ensemble: 1, cell: -1}``.
Chunks change storage, not data selection or grid resolution.

Additional Anemoi options (defaults shown):

.. code-block:: yaml

   statistics_period: {}                        # Optional inclusive start/end strings; default full period
   compute_temporal_residual_statistics: false  # Statistics of adjacent-time differences
   sort_channels_by_levels: false               # Numeric level suffix ordering
   use_level_index: false                       # Name levels by position instead of coordinate value
   channel_rename: {}                           # Rename packed channels, e.g. {t_500: temperature_500}
   variables_with_nans: null                    # Allowed channels; exact name, else case-insensitive substring
   resolution: null                             # Metadata only; does not regrid
   transformed_dims: {}                         # Standard anemoi only; e.g. {latitude: y, longitude: x}
   multistep_input: 1                           # Inference-with-forcings only; input time count
   multisource_masks: []                        # Optional mask rules below; mapping or list

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
     - mask_from: temp_0           # Required reference channel
       mask_variables: [temp_0]    # Default []; channels to mask
       output: ocean_mask          # Optional new mask channel
       land_value: 1               # Default 1: encoded value for masked cells
       ocean_value: 0              # Default 0: encoded value for retained cells
       static_mask:                # Optional; omit to use per-sample NaNs
         path: /path/to/mask.zarr  # Required existing artifact
         mode: reuse               # Default reuse; only supported mode
         variable: temp_0          # Default mask_from
         method: any_valid         # Optional policy check: any_valid/all_valid/minimum_valid_fraction
         # minimum_valid_fraction: 0.9  # Required only with minimum_valid_fraction method

.. _recipe-attrs:

attrs
=====

Use ``attrs`` to add global metadata to the output. You can choose your own keys,
but avoid overwriting metadata generated by ufs2arco.

.. code-block:: yaml

   attrs:
     description: Regional temperature dataset
     institution: NOAA
