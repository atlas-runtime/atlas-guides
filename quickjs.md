# QuickJS Atlas Support

We have a modified QuickJS interpreter that supports Atlas.  The modifications
create a predefined global variable named atlas that provides a number of
entry points.

### atlas methods

  - atlas.allocate_clients (server-cnt)

    Allocates a pthread_mutex lock for each client (currently hardcoded to 1).
    Initializes node_crypto_info global.

  - atlas.socket()

    Creates a sockeet in the normal manner and returns it.  Not sure why we need
    this.

  - atlas.connect (socket, ip, port)

    Connects using the socket previously created in atlas.sockeet(0 to the
    specified ip/port.  Also generates a keypair with crypto_box_keypair().
    and stores it in node_crpto_info (but doesn't do anything with it).
    **Note that we just exit if we can't connect.  Need to handle this case
    somehow.**

  - atlas.send_pubkey(socket)

  - atlas.recv_pubkey(socket)

  - atlas.recv_encryption_key (socket)

  - atlas.import_modules() [quickjs.js_read_export_modules]

    Initializes the 100K module_export_table to zeros.  Returns an
    some sort of reference to the module_export_table (JSValue)

  - atlas.start_wrapping()

    Sets the boolean start_wrapping_atlas to 1.

  - atlas.execute_script (script-file)

    Executes the specified file in what appears to be the normal fashion
    (calling eval_buf()).  Adds the code `;globalThis.__atlas_done_exec=1;`
    to the end of the code to execute.  Its not clear if anything else
    special happens here.

  - atlas.read_modules()

    This returns a reference to the modules_entries_table[].  But it also
    appears to clear that table.  So its not apparent how the code that
    accesses it is able to read module names out of it.  Module names are
    written to itin quickjs.js_link_module()

### Internal data structures

  - quickjs: char module_entries_table[100000]

    This looks like a list of modules with an internal and external name.
    Each module is on a separate line.  I think this gets filled in when
    the 
