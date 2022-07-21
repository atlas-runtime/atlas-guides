# Atlas Internals

### Workers

Atlas spawns a worker thread for each remote server that it might use.
This is implemented in `atlas-worker.js`.

### Atlas-wrapper.js

  - spawn_workers()

    Spawns a thread for each server.  Once the last worker is started,
    it starts execution of the script.  this includes:

      - calling atlas.import_modules()
      - calling atlas.start_wrapping()
      - calling atlas.execute_script (main program we are running)

  - wrapObject (nobj, real_name, import_name)

    This creates a proxy around exported functions and method of a class.
    
