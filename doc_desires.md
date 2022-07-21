# Desired Documentation for Atlas

We would like to better understand the technical approach and details
of the current implementation.  The following sections list specific
items of interest.  Documentation could be added to the code directly
or stored externally.  If code documentation is used it would be
great to have an external list of documented items to guide
learning about the code (especially in QuickJS where our mods are
a small subset of the overall code).

There is no need to spend time making the documentation pretty in
any way.  Markdown seems more than sufficient as a format.

### Overall

  - An overview of the overall design.  What are the components (QuickJS
    mods, worker, client, etc).  How do those components interact.

  - What are the external inputs in the system (eg, server addresses)?

  - What are the current limitations?  How should those limitations be
    addressed?

  - What do we need to address before providing the tool directly to
    users?

### QuickJS changes

  - An overview of the basic design of the modifications.  What are they
    trying to accomplish?  How do they interact with the javascript code.
    There is a start on this in atlas-guides/quickjs.md

  - A description of each entry point/parameters/return value
    added to QuickJS (eg, `atlas.execute_script (script_file)`)

  - A description of the global data used by the modifications (eg,
    `char module_entries_table[100000]`)

### Atlas Worker

  - An overview of the worker design.  There is a (very short) start to
    this in atlas-guides/atlas-internals.md

  - A description of each external entry point/parameters/return value
    to the worker.

  - A description of the global data.

### Atlas Client

  - An overview of the client design including a description of how the
    client interacts with QuickJS and the worker.

  - Description of each argument and how the system is intended to be used.

  - Internal (code) documentation of each function

  - A description of any global data.

  - Any application specific code that is part of atlas and not the
    application.  For example, the code that runs the benchmarks seems
    to be application specific, yet part of atlas
