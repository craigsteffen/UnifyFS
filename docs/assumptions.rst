================
Assumptions
================

In this section, we provide assumptions we make about the behavior of
applications that use UnifyFS, and about how UnifyFS currently functions.

---------------------------
Application Behavior
---------------------------
    - Workload supported is globally synchronous checkpointing.

    - I/O occurs in write and read phases. Files are not read and written at
      the same time. There is some (good) amount of time between the two phases.
      For example, files are written during checkpoint phases and only read
      during recovery or restart.

    - Processes on any node can read any byte in the file (not just local
      data), but the common case will be processes read only their local bytes.

    - Assume general parallel I/O concurrency semantics where processes can
      write to the same offset concurrently. We assume the outcome of concurrent
      writes to the same offset or other conflicting concurrent accesses is
      undefined. For example, if a command in the job renames a file while the
      parallel application is writing to it, the outcome is undefined. It could
      be a failure or not, depending on timing.

---------------------------
Consistency Model
---------------------------
One key aspect of UnifyFS is the idea of "laminating" a file.  After a file is
laminated, it becomes "set in stone" and its data is accessible across all the
nodes.  Laminated files are permanently read-only, and cannot be modified in
any way (but can be deleted).  If the application process group fails before a
file has been laminated, UnifyFS may delete the file.

The typical use case is to laminate your checkpoint files after they've been
written.  To laminate a file, first call fsync() to sync all your writes to the
server, then call chmod() to remove all the write bits.  Removing the write
bits does the actual lamination.  A typical checkpoint will look like this:

.. code-block:: C

  fp = fopen("checkpoint1.chk")
  write(fp, <checkpoint data>)
  fsync(fp)
  fclose(fp)
  chmod("checkpoint1.chk", 0444)

Future versions of UnifyFS may support different laminate semantics, such as
laminate on close(), or laminate via an explicit API call.

We define the laminated consistency model to enable certain optimizations
while supporting the perceived requirements of application checkpoints.
Since remote processes are not permitted to read arbitrary bytes within the
file until lamination,
global exchange of file data and/or data index information can be buffered
locally on each node until the point of lamination.
Since file contents cannot change after lamination,
aggressive caching may be used during the read-only phase with minimal locking.
Since a file may be lost on application failure unless laminated,
data redundancy schemes can be delayed until lamination.

Behavior before lamination:

  - open/close: A process may open/close a file multiple times.

  - write: A process may write to any part of a file. If two processes write
    to the same location, the value is undefined.

  - read: A process may read bytes it has written. Reading other bytes is
    invalid.

  - rename: A process may rename a file.

  - truncate: A process may truncate a file.

  - unlink: A process may delete a file.

Behavior after lamination:

  - open/close: A process may open/close a file multiple times.

  - write: All writes are invalid.

  - read: A process may read any byte in the file.

  - rename: A process may rename a file.

  - truncate: Truncation is invalid (considered to be a write operation).

  - unlink: A process may delete a file.

---------------------------
File System Behavior
---------------------------

    - The file system exists on node local storage only and is not persisted to 
      stable storage like a parallel file system (PFS). Can be coupled with

    - SymphonyFS or high level I/O or checkpoint library (VeloC) to move data to
      PFS periodically, or data can be moved manually

    - Can be used with checkpointing libraries (VeloC) or I/O libraries to
      support shared files on burst buffers

    - File system starts empty at job start. User job must populate the file
      system.

    - Shared file system namespace across all compute nodes in a job, even if
      an application process is not running on all compute nodes

    - Survives application termination and/or relaunch within a job

    - Will transparently intercept system level I/O calls of applications and
      I/O libraries

---------------------------
System Characteristics
---------------------------

    - There is some storage available for storing file data on a compute node,
      e.g. SSD or RAM disk

    - We can run user-level daemon processes on compute nodes concurrently with
      a user application
