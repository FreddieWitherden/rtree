.. _performance:

Performance
------------------------------------------------------------------------------

See the `benchmarks.py`_ file for a comparison of various query methods
and how much acceleration can be obtained from using Rtree.

.. _benchmarks.py: https://github.com/Toblerity/rtree/blob/main/benchmarks/benchmarks.py

There are a few simple things that will improve performance.

Use stream loading
..............................................................................

This will substantially (orders of magnitude in many cases) improve
performance over :py:meth:`~rtree.index.Index.insert` by allowing the data to
be pre-sorted

.. code-block:: pycon

   >>> from rtree import index
   >>> def generator_function(somedata):
   ...     for i, obj in enumerate(somedata):
   ...         yield (i, (obj.xmin, obj.ymin, obj.xmax, obj.ymax), obj)
   ...
   >>> r = index.Index(generator_function(somedata))  # doctest: +SKIP

After bulk loading the index, you can then insert additional records into
the index using :py:meth:`~rtree.index.Index.insert`

Override :py:data:`~rtree.index.Index.dumps` to use the highest pickle protocol
...............................................................................

.. code-block:: pycon

    >>> import pickle
    >>> import rtree
    >>> class FastRtree(rtree.Rtree):
    ...     def dumps(self, obj):
    ...         return pickle.dumps(obj, -1)
    ...
    >>> r = FastRtree()

.. topic:: Update from January 2024

   Pickling is currently broken and awaiting a pull request to fix it.
   For more information, see the `pull request on GitHub`_.

.. _pull request on GitHub: https://github.com/Toblerity/rtree/pull/197

Use objects="raw"
...............................................................................

In any :py:meth:`~rtree.index.Index.intersection` or
:py:meth:`~rtree.index.Index.nearest` or query, use ``objects="raw"`` keyword
argument:

.. code-block:: pycon

    >>> xmin, ymin, xmax, ymax = 0.0, 0.0, 1.0, 1.0
    >>> objs = r.intersection((xmin, ymin, xmax, ymax), objects="raw")


Adjust index properties
...............................................................................

Adjust :py:class:`rtree.index.Property` appropriate to your index.

   * Set your :py:data:`~rtree.index.Property.leaf_capacity` to a higher value
     than the default 100. 1000+ is fine for the default pagesize of 4096 in
     many cases.

   * Increase the :py:data:`~rtree.index.Property.fill_factor` to something
     near 0.9. Smaller fill factors mean more splitting, which means more
     nodes. This may be bad or good depending on your usage.

Limit dimensionality to the amount you need
...............................................................................

Don't use more dimensions than you actually need. If you only need 2, only use
two. Otherwise, you will waste lots of storage and add that many more floating
point comparisons for each query, search, and insert operation of the index.

Use the correct query method
...............................................................................

Use :py:meth:`~rtree.index.Index.count` if you only need a count and
:py:meth:`~rtree.index.Index.intersection` if you only need the ids.
Otherwise, lots of data may potentially be copied.  If possible also
make use of the bulk query methods suffixed with `_v`.
