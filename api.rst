
API reference
*************

**toplevel(index_class=None, *, fields_sep: str = '-')**

   Turn decorated class into an indexing schema.

   Decorated class is equipped with methods and attributes to use with
   a ``ParquetSet`` instance. It has to be defined as one would define
   a class decorated by ``@dataclass``.

   :Parameters:
      **fields_sep** (*str**, **default '.'*) – Character to use as
      separator between fields of the dataclass.

   :Returns:
      *Decorated class.*

   ``fields_sep``

      Fields separator (can’t assign).

      :Type:
         str

   ``depth``

      Number of levels, including ‘toplevel’ (can’t assign).

      :Type:
         int

   -[ Notes ]-

   ``@dataclass`` is actually called when decorating with
   ``@toplevel`` with parameters set to:

      *  ``order=True``,

      *  ``frozen=True``

   When class is instantiated, a validation step is conducted on
   attributes types and values.

      *  An instance can only be composed with ``int``, ``str`` or a
         dataclass object coming in last position;

      *  Value of attribute can not incorporate forbidden characters
         like ``/`` and ``self.fields_sep``.

**sublevel(index_class)**

   Define a subdirectory level.

   This decorator really is an alias of ``@dataclass`` decorator, with
   parameters set to:

      *  ``order=True``,

      *  ``frozen=True``

**class ParquetSet(basepath: str, indexer:
Type[dataclasses.dataclass])**

   Sorted list of keys (indexes to parquet datasets).

   ``basepath``

      Directory path to the set of parquet datasets.

      :Type:
         str

   ``indexer``

      Indexer schema (class) to be used to index parquet datasets.

      :Type:
         Type[dataclass]

   ``keys``

      Set of indexes of existing parquet datasets.

      :Type:
         SortedSet

   -[ Notes ]-

   ``SortedSet`` is the data structure retained for ``keys`` instead
   of ``SortedList`` as its ``__contains__`` appears faster.

**write(dirpath: str, data: Union[pandas.DataFrame,
vaex.dataframe.DataFrame], max_row_group_size: Optional[int] = None,
compression: str = 'SNAPPY', cmidx_expand: bool = False, cmidx_levels:
Optional[List[str]] = None, ordered_on: Optional[Union[str,
Tuple[str]]] = None, duplicates_on: Optional[Union[str, List[str],
List[Tuple[str]]]] = None, max_nirgs: Optional[int] = None)**

   Write data to disk at location specified by path.

   :Parameters:
      *  **dirpath** (*str*) – Directory where writing pandas
         dataframe.

      *  **data** (*Union**[**pDataFrame**, **vDataFrame**]*) – Data
         to write.

      *  **max_row_group_size** (*int**, **optional*) – Max row group
         size. If not set, default to ``6_345_000``, which for a
         dataframe with 6 columns of ``float64``/``int64`` results in
         a memory footprint (RAM) of about 290MB.

      *  **compression** (*str**, **default SNAPPY*) – Algorithm to
         use for compressing data. This parameter is fastparquet
         specific. Please see fastparquet documentation for more
         information.

      *  **cmidx_expand** (*bool**, **default False*) – If *True*,
         expand column index into a column multi-index. This parameter
         is only used at creation of the dataset. Once column names
         are set, they cannot be modified by use of this parameter.

      *  **cmidx_levels** (*List**[**str**]**, **optional*) – Names of
         levels to be used when expanding column names into a
         multi-index. If not provided, levels are given names ‘l1’,
         ‘l2’, …

      *  **ordered_on** (*Union**[**str**, **Tuple**[**str**]**]
         **optional*) –

         Name of the column with respect to which dataset is in
         ascending order. If column multi-index, name of the column is
         a tuple. This parameter is optional, and required so that
         data overlaps between new data and existing recorded row
         groups can be identified. This parameter is compulsory when
         ``duplicates_on`` is used. If not set, default to ``None``.
         It has two effects:

            *  it allows knowing ‘where’ to insert new data into
               existing data, i.e. completing or correcting past
               records (but it does not allow to remove prior data).

            *  it ensures that two consecutive row groups do not have
               duplicate values in column defined by ``ordered_on``
               (only in row groups to be written). This implies that
               all possible duplicates in ``ordered_on`` column will
               lie in the same row group.

      *  **duplicates_on** (*Union**[**str**, **List**[**str**]**,
         **List**[**Tuple**[**str**]**]**]**, **optional*) – Column
         names according which ‘row duplicates’ can be identified
         (i.e. rows sharing same values on these specific columns) so
         as to drop them. Duplicates are only identified in new data,
         and existing recorded row groups that overlap with new data.
         If duplicates are dropped, only last is kept. To identify row
         duplicates using all columns, empty list ``[]`` can be used
         instead of all columns names. If not set, default to
         ``None``, meaning no row is dropped.

      *  **max_nirgs** (*int**, **optional*) –

         Max expected number of ‘incomplete’ row groups. A ‘complete’
         row group is one which size is ‘close to’
         ``max_row_group_size`` (>=90%). To evaluate number of
         ‘incomplete’ row groups, only those at the end of an existing
         dataset are accounted for. ‘Incomplete’ row groups in the
         middle of ‘complete’ row groups are not accounted for (they
         can be created by insertion of new data ‘in the middle’ of
         existing data). If not set, default to ``None``.

            *  ``None`` value induces no coalescing of row groups. If
               there is no drop of duplicates, new data is
               systematically appended.

            *  A value of ``0`` or ``1`` means that new data should
               systematically be merged to the last existing one to
               ‘complete’ it (if it is not ‘complete’ already).

   -[ Notes ]-

   *  When writing a dataframe with this function,

      *  index of dataframe is not written to disk.

      *  parquet file scheme is ‘hive’ (one row group per parquet
         file).

   *  Coalescing incomplete row groups is triggered depending 2
      conditions, either actual number of incomplete row groups is
      larger than ``max_nirgs`` or number of rows for all incomplete
      row groups (at the end of the dataset) is enough to make a new
      complete row group (reaches ``max_row_group_size``). This latter
      assessment is however only triggered if ``max_nirgs`` is set.
      Otherwise, new data is simply appended, without prior check.

   *  When ``duplicates_on`` is set, duplicate search is made row
      group to be written per row group to be written. A *row group to
      be written* is made from the merge between new data, and
      existing recorded row groups which overlap. For this reason
      ``ordered_on`` parameter is compulsory when using
      ``duplicates_on``, so as to be able to position new data with
      respect to existing row groups and also to cluster this data
      (new and overlapping recorded) into row group which have
      distinct values in ``ordered_on`` column. If 2 rows are
      duplicates according values in indicated columns but are not in
      the same row group, first duplicates will not be dropped.

   *  As per logic of previous comment, duplicates need to be gathered
      by row group to be identified, they need consequently to share
      the same *index*, defined by the value in ``ordered_on``.
      Extending this logic, ``ordered_on`` is added to
      ``duplicates_on`` if not already part of it.

   *  For simple data appending, i.e. without need to check where to
      insert data and without need to drop duplicates, it is advised
      to keep ``ordered_on`` and ``duplicates_on`` parameters set to
      ``None`` as these parameters will trigger unnecessary
      evaluations.
