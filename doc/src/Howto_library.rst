Library interface to LAMMPS
===========================

As described on the :doc:`Build basics <Build_basics>` doc page, LAMMPS
can be built as a library, so that it can be called by another code,
used in a :doc:`coupled manner <Howto_couple>` with other codes, or
driven through a :doc:`Python interface <Python_head>`.

All of these methodologies use a C-style interface to LAMMPS that is
provided in the files src/library.cpp and src/library.h.  The
functions therein have a C-style argument list, but contain C++ code
you could write yourself in a C++ application that was invoking LAMMPS
directly.  The C++ code in the functions illustrates how to invoke
internal LAMMPS operations.  Note that LAMMPS classes are defined
within a LAMMPS namespace (LAMMPS\_NS) if you use them from another C++
application.

The examples/COUPLE and python/examples directories have example C++
and C and Python codes which show how a driver code can link to LAMMPS
as a library, run LAMMPS on a subset of processors, grab data from
LAMMPS, change it, and put it back into LAMMPS.

The file src/library.cpp contains the following functions for creating
and destroying an instance of LAMMPS and sending it commands to
execute.  See the documentation in the src/library.cpp file for
details.

.. note::

   You can write code for additional functions as needed to define
   how your code talks to LAMMPS and add them to src/library.cpp and
   src/library.h, as well as to the :doc:`Python interface <Python_head>`.
   The added functions can access or change any internal LAMMPS data you
   wish.


.. parsed-literal::

   void lammps_open(int, char \*\*, MPI_Comm, void \*\*)
   void lammps_open_no_mpi(int, char \*\*, void \*\*)
   void lammps_close(void \*)
   int lammps_version(void \*)
   void lammps_file(void \*, char \*)
   char \*lammps_command(void \*, char \*)
   void lammps_commands_list(void \*, int, char \*\*)
   void lammps_commands_string(void \*, char \*)
   void lammps_free(void \*)

The lammps\_open() function is used to initialize LAMMPS, passing in a
list of strings as if they were :doc:`command-line arguments <Run_options>` when LAMMPS is run in stand-alone mode
from the command line, and a MPI communicator for LAMMPS to run under.
It returns a ptr to the LAMMPS object that is created, and which is
used in subsequent library calls.  The lammps\_open() function can be
called multiple times, to create multiple instances of LAMMPS.

LAMMPS will run on the set of processors in the communicator.  This
means the calling code can run LAMMPS on all or a subset of
processors.  For example, a wrapper script might decide to alternate
between LAMMPS and another code, allowing them both to run on all the
processors.  Or it might allocate half the processors to LAMMPS and
half to the other code and run both codes simultaneously before
syncing them up periodically.  Or it might instantiate multiple
instances of LAMMPS to perform different calculations.

The lammps\_open\_no\_mpi() function is similar except that no MPI
communicator is passed from the caller.  Instead, MPI\_COMM\_WORLD is
used to instantiate LAMMPS, and MPI is initialized if necessary.

The lammps\_close() function is used to shut down an instance of LAMMPS
and free all its memory.

The lammps\_version() function can be used to determined the specific
version of the underlying LAMMPS code. This is particularly useful
when loading LAMMPS as a shared library via dlopen(). The code using
the library interface can than use this information to adapt to
changes to the LAMMPS command syntax between versions. The returned
LAMMPS version code is an integer (e.g. 2 Sep 2015 results in
20150902) that grows with every new LAMMPS version.

The lammps\_file(), lammps\_command(), lammps\_commands\_list(), and
lammps\_commands\_string() functions are used to pass one or more
commands to LAMMPS to execute, the same as if they were coming from an
input script.

Via these functions, the calling code can read or generate a series of
LAMMPS commands one or multiple at a time and pass it through the library
interface to setup a problem and then run it in stages.  The caller
can interleave the command function calls with operations it performs,
calls to extract information from or set information within LAMMPS, or
calls to another code's library.

The lammps\_file() function passes the filename of an input script.
The lammps\_command() function passes a single command as a string.
The lammps\_commands\_list() function passes multiple commands in a
char\*\* list.  In both lammps\_command() and lammps\_commands\_list(),
individual commands may or may not have a trailing newline.  The
lammps\_commands\_string() function passes multiple commands
concatenated into one long string, separated by newline characters.
In both lammps\_commands\_list() and lammps\_commands\_string(), a single
command can be spread across multiple lines, if the last printable
character of all but the last line is "&", the same as if the lines
appeared in an input script.

The lammps\_free() function is a clean-up function to free memory that
the library allocated previously via other function calls.  See
comments in src/library.cpp file for which other functions need this
clean-up.

The file src/library.cpp also contains these functions for extracting
information from LAMMPS and setting value within LAMMPS.  Again, see
the documentation in the src/library.cpp file for details, including
which quantities can be queried by name:


.. parsed-literal::

   int lammps_extract_setting(void \*, char \*)
   void \*lammps_extract_global(void \*, char \*)
   void lammps_extract_box(void \*, double \*, double \*,
                           double \*, double \*, double \*, int \*, int \*)
   void \*lammps_extract_atom(void \*, char \*)
   void \*lammps_extract_compute(void \*, char \*, int, int)
   void \*lammps_extract_fix(void \*, char \*, int, int, int, int)
   void \*lammps_extract_variable(void \*, char \*, char \*)

The extract\_setting() function returns info on the size
of data types (e.g. 32-bit or 64-bit atom IDs) used
by the LAMMPS executable (a compile-time choice).

The other extract functions return a pointer to various global or
per-atom quantities stored in LAMMPS or to values calculated by a
compute, fix, or variable.  The pointer returned by the
extract\_global() function can be used as a permanent reference to a
value which may change.  For the extract\_atom() method, see the
extract() method in the src/atom.cpp file for a list of valid per-atom
properties.  New names could easily be added if the property you want
is not listed.  For the other extract functions, the underlying
storage may be reallocated as LAMMPS runs, so you need to re-call the
function to assure a current pointer or returned value(s).


.. parsed-literal::

   double lammps_get_thermo(void \*, char \*)
   int lammps_get_natoms(void \*)

   int lammps_set_variable(void \*, char \*, char \*)
   void lammps_reset_box(void \*, double \*, double \*, double, double, double)

The lammps\_get\_thermo() function returns the current value of a thermo
keyword as a double precision value.

The lammps\_get\_natoms() function returns the total number of atoms in
the system and can be used by the caller to allocate memory for the
lammps\_gather\_atoms() and lammps\_scatter\_atoms() functions.

The lammps\_set\_variable() function can set an existing string-style
variable to a new string value, so that subsequent LAMMPS commands can
access the variable.

The lammps\_reset\_box() function resets the size and shape of the
simulation box, e.g. as part of restoring a previously extracted and
saved state of a simulation.


.. parsed-literal::

   void lammps_gather_atoms(void \*, char \*, int, int, void \*)
   void lammps_gather_atoms_concat(void \*, char \*, int, int, void \*)
   void lammps_gather_atoms_subset(void \*, char \*, int, int, int, int \*, void \*)
   void lammps_scatter_atoms(void \*, char \*, int, int, void \*)
   void lammps_scatter_atoms_subset(void \*, char \*, int, int, int, int \*, void \*)

The gather functions collect peratom info of the requested type (atom
coords, atom types, forces, etc) from all processors, and returns the
same vector of values to each calling processor.  The scatter
functions do the inverse.  They distribute a vector of peratom values,
passed by all calling processors, to individual atoms, which may be
owned by different processors.

.. warning::

   These functions are not compatible with the
   -DLAMMPS\_BIGBIG setting when compiling LAMMPS.  Dummy functions
   that result in an error message and abort will be substituted
   instead of resulting in random crashes and memory corruption.

The lammps\_gather\_atoms() function does this for all N atoms in the
system, ordered by atom ID, from 1 to N.  The
lammps\_gather\_atoms\_concat() function does it for all N atoms, but
simply concatenates the subset of atoms owned by each processor.  The
resulting vector is not ordered by atom ID.  Atom IDs can be requested
by the same function if the caller needs to know the ordering.  The
lammps\_gather\_subset() function allows the caller to request values
for only a subset of atoms (identified by ID).
For all 3 gather function, per-atom image flags can be retrieved in 2 ways.
If the count is specified as 1, they are returned
in a packed format with all three image flags stored in a single integer.
If the count is specified as 3, the values are unpacked into xyz flags
by the library before returning them.

The lammps\_scatter\_atoms() function takes a list of values for all N
atoms in the system, ordered by atom ID, from 1 to N, and assigns
those values to each atom in the system.  The
lammps\_scatter\_atoms\_subset() function takes a subset of IDs as an
argument and only scatters those values to the owning atoms.


.. parsed-literal::

   void lammps_create_atoms(void \*, int, tagint \*, int \*, double \*, double \*,
                            imageint \*, int)

The lammps\_create\_atoms() function takes a list of N atoms as input
with atom types and coords (required), an optionally atom IDs and
velocities and image flags.  It uses the coords of each atom to assign
it as a new atom to the processor that owns it.  This function is
useful to add atoms to a simulation or (in tandem with
lammps\_reset\_box()) to restore a previously extracted and saved state
of a simulation.  Additional properties for the new atoms can then be
assigned via the lammps\_scatter\_atoms() or lammps\_extract\_atom()
functions.


.. _lws: http://lammps.sandia.gov
.. _ld: Manual.html
.. _lc: Commands_all.html
