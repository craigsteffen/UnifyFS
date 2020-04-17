================
Overview
================

UnifyFS is a user level file system currently under active development. An
application can use node-local storage as burst buffers for shared files.
UnifyFS is designed to support both checkpoint/restart which is the most
important I/O workload for HPC and other common I/O workloads as well. With
UnifyFS, applications can write to fast, scalable, node-local burst buffers as
easily as they do the parallel file system. This section will provide a high
level design of UnifyFS. It will describe the UnifyFS library and the UnifyFS
daemon.

This is a new paragraph.  We will see if lines that are created with the GUI
editor will have trailing white space problems.

---------------------------
High Level Design
---------------------------

.. image:: images/design-high-lvl.png

UnifyFS will present a shared namespace (e.g., /unifyfs as a mount point) to
all compute nodes in a users job allocation. There are two main components of
UnifyFS: the UnifyFS library and the UnifyFS daemon. The UnifyFS library (also
referred to as the UnifyFS client library) is linked into the user application
and is responsible for intercepting I/O calls from the user application and
then sending the I/O requests on to a UnifyFS server to be handled. The UnifyFS
client library uses the ECP `GOTCHA <https://github.com/LLNL/GOTCHA>`_ software
as its primary mechanism for intercepting I/O calls. Each UnifyFS daemon (also
referred to as a UnifyFS server daemon) runs as a daemon on a compute node in
the users allocation. The UnifyFS server is responsible for handling the I/O
requests from the UnifyFS library. On each compute node, there will be user
application processes running as well as tool daemon processes. The user
application is linked with the UnifyFS client library and a high-level I/O
library, e.g. HDF5, ADIOS, or PnetCDF. The UnifyFS server daemon also runs on
the compute node and is linked with the MDHIM library which is used for
metadata services.
