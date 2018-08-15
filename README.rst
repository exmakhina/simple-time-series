##################
Simple Time Series
##################


This is a presentation of a very simple and compact "standard" for time series.

The gist of it:

- A data format:

  - A text-based representation is YAML-based, it basically looks like this:

    .. code:: yaml

       - time: 0
         a: 1.3
       - time: 1.33
       - time: 2.00
         b: 1.5

    or this:

    .. code:: yaml

       - span: [2017-01, 2017-02]
         elec_consumption_kWh: 31.337
         water_consumption_m3: 0.05
       - time: [2017-02, 2017-03]
         elec_consumption_kWh: 1.337
         water_consumption_m3: 0.03

- Considerations, eg. for compaction / collapsing non-events during real-time
  writing


Such data is typically generated in engineering contexts, where some data may
not change for long periods, and exhaustively logging may be a waste of space.


YAML-Based Text Data Format
###########################

- List of entries

- Each entry is a dictionary containing a "time" key and its value or a "span"
  key and its value, and other keys and values.

- The time value, used to express entries corresponding to sampling points,
  is either a real in seconds, or an ISO8601 timestamp

- The span value, used to express entries corresponding to cumulated data
  between 2 time points, is represented by a list containing 2 time values,
  the second point should be considered excluded (as in :math:`[a,b)`).

- Must be YAML-compatible (can be parsed as YAML, although it's not very fast
  compared to other)

- Keys must be text

- Values may be anything, although they'll commonly be booleans, integers, or
  reals

- Absence of time data means "no change"

- Absence of span data means "nothing to count"

- Data may be prefixed by a YAML header, may be included in YAML subsections

  .. code:: yaml

     !time-series
     some-key: some-values
     series:
     - time: 0
       temperature_degC: 51.4


Considerations
##############

Compaction
**********

- Compaction (not writing redundant data) can drastically reduce the size of
  time series data

- To be safe with data that may be plotted, introduction of redundancy may be
  useful, eg. when a value changed, provide the previous value at the previous
  time step.

  Indeed, the plotting tool may decide to join sparse points with a straight
  line, considering the (x,y) series a polygon.
  A compaction for such tools needs to consider that removal of redundant points
  must keep the same polygon shape.

- To be less safe but more compact, consider outputting a value when it's
  introduced or when it changes, but programmatically deal with that in cases
  the data may be plotted.
  The problem is that the previous time point is lost, so one would have to
  substract an epsilon to the current time point.


Suggested pseudo-code for writing compact entries, with:

- A periodic 1-second redundant write (for keepalive purposes)
- Addition of previous state before a new value, so the resulting data
  can be plotted by something simple.

.. code:: python

    # keep track of the last time value
    last = time()

    # keep track of last values (not sparse)
    values_ = dict()
    # initialize with known variables set as None
    for k in state.keys():
        values_[k] = None

    # store "last changes" structure
    changes_ = dict()
    changes_.update(state)

    # store whether a first entry was written
    did_first_write = False

    # store last write time (to compute the 1-second redudancy)
    last_write = time()

    while True:
        now = time()

        values = dict()
        values.update(state)

        changes = dict()
        for k, v in sorted(values.items()):
            v_ = values_.get(k, None)
            if v_ is not None and v != v_:
                changes[k] = v

        force_write = now > last_write + 1.0
        if changes or force_write:
            with io.open("example.yml", "a", encoding="utf-8") as f:

                need_past_update = []
                for k, v in changes.items():
                    if k not in changes_:
                        # was not changed before but changed now,
                        # so we need to provide a past reference
                        # for the value
                        v_ = values_[k]
                        need_past_update.append((k, v_))

                if need_past_update:
                    if not did_first_write:
                        # need a header for reference data written
                        # if need_past_update
                        f.write("- time: %.3f\n" % last)

                    for k, v in need_past_update:
                        f.write("  %s: %s\n" % (k, v))

                writes = dict()
                if force_write:
                    # write not only the current changes, but all values
                    writes.update(values_)
                writes.update(changes)

                # update "last values" structure
                values_.update(values)

                if writes:
                    f.write("- time: %.3f\n" % now)
                    for k, v in writes.items():
                        if v is not None:
                            f.write("  %s: %s\n" % (k, v))

                    did_first_write = True
                    last_write = now

        last = now


Misc
****

- Existence of an entry, even if it just contains the *time* key, may be used
  for keepalive purposes.

- This is a given, but new keys may appear at any point in the entry stream


