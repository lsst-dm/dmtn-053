..
  Technote content.

  See https://developer.lsst.io/docs/rst_styleguide.html
  for a guide to reStructuredText writing.

  Do not put the title, authors or other metadata in this document;
  those are automatically added.

  Use the following syntax for sections:

  Sections
  ========

  and

  Subsections
  -----------

  and

  Subsubsections
  ^^^^^^^^^^^^^^

  To add images, add the image file (png, svg or jpeg preferred) to the
  _static/ directory. The reST syntax for adding the image is

  .. figure:: /_static/filename.ext
     :name: fig-label

     Caption text.

   Run: ``make html`` and ``open _build/html/index.html`` to preview your work.
   See the README at https://github.com/lsst-sqre/lsst-technote-bootstrap or
   this repo's README for more info.

   Feel free to delete this instructional comment.

:tocdepth: 1

.. Please do not modify tocdepth; will be fixed when a new Sphinx theme is shipped.

.. sectnum::

.. Add content below. Do not include the document title.

Introduction
============

While processing processing HSC data using the LSST stack on the batch farm of `CC-IN2P3 <https://cc.in2p3.fr/en>`_, a rather low CPU efficiency (i.e. the ratio CPU time/wall time) was observed. 
At CC-IN2P3, both the raw HSC images and the resulting reduced images are stored in a GPFS cluster. It was hypothesized that the observed batch jobs' undesired behavior could be linked to some GPFS client thrashing, so we decided to investigate.

This note summarizes what we observed in the first analysis pass. It was first published as a `LSST community post <https://community.lsst.org/t/observations-on-i-o-activity-induced-by-ingestimages-py-and-processccd-py/2131>`_, where you can find the feedback provided by experts on the LSST software framework.

Testing environment
===================

Our deliberately simple test is composed of two steps:

#. ingest a single raw image (one CCD) using ``ingestImage.py``
#. use ``processCcd.py`` for processing a single CCD image

For our tests we use the LSST weekly version ``w_2017_29`` (Python 3) on a compute node running CentOS 7. The compute node runs the GPFS client and is connected to the GPFS cluster by a 10 Gbps network link. The GPFS client is configured to use 8 GB of RAM for caching data.

Image ingestion
===============

The HSC raw images are found in the ``raw`` directory of our testing environment. The command we use for building the image registry is:

.. prompt:: bash

  ingestImages.py data ./raw/HSCA01151351.fits --mode link

As a result of this, among other things, a file ``registry.sqlite3`` is created in the ``data`` directory. In our case, both the ``data`` directory and the ``raw`` directory reside on GPFS.

.. code-block:: none

   $ tree data
   data
   ├── _mapper
   ├── registry.sqlite3
   └── SSP_UDEEP_SXDS
       └── 2014-11-18
           └── 01052
               └── HSC-R
                   └── HSC-0011512-105.fits -> /sps/lsst/dev/fabio/hscIO/ingest/raw/HSCA01151351.fits

    4 directories, 3 files

In the process of populating ``registry.sqlite3``, the `sqlite3`_ library creates some temporary files (with extensions ``-journal`` or ``-wal``) in the same directory as the final destination file, i.e. ``data``. It also repeatedly acquires and releases locks on those temporary files. When the population of the registry file is finished, the temporary file is renamed to its final name ``registry.sqlite3``.

.. _sqlite3: https://www.sqlite.org/index.html

Locking files or portion of files in a shared file system is a potentially costly operation since a synchronization of all the nodes using the file system is required to honor POSIX semantics. According to the `SQLite documentation`_, there is a way to instruct the library where to create temporary files. It didn't work in our tests: setting the values of one of the environmental variables ``SQLITE_TMPDIR``, ``TMPDIR`` or ``TMP`` had no effect when using a test program linking against the sqlite3 shared library shipped with the stack.

.. _SQLite documentation: https://www.sqlite.org/tempfiles.html

It would be worth considering the command line tasks of the stack to create any temporary file in the local storage of the compute node (either local scratch disk or even RAM disk) and to copy those files back to their final destination (i.e. the ``data`` directory in GPFS in this particular case) when they are no longer needed. The command line tasks could look for the value of the POSIX ``TMPDIR`` variable for the location to store those temporary files. Processing sites would then set the ``TMPDIR`` variable to the appropriate location for each job to use for scratch storage.



Process CCD
===========

The command for this test is:

.. prompt:: bash

  processCcd.py input --output output --id visit=38080 ccd=23

To our knowledge, there is nothing special with this particular visit and this particular CCD. The size of the input image file of CCD 23 is 17 MB and its format is FITS.

At the end of the process several files are created under the ``output`` directory:

.. code-block:: none

   $ tree output
   output
   ├── 01318
   │   └── HSC-I
   │       ├── corr
   │       │   ├── BKGD-0038080-023.fits
   │       │   └── CORR-0038080-023.fits
   │       ├── output
   │       │   ├── ICSRC-0038080-023.fits
   │       │   ├── SRC-0038080-023.fits
   │       │   ├── SRCMATCH-0038080-023.fits
   │       │   └── SRCMATCHFULL-0038080-023.fits
   │       ├── processCcd_metadata
   │       │   └── 0038080-023.boost
   │       └── thumbs
   │           ├── flattened-0038080-023.png
   │           └── oss-0038080-023.png
   ├── config
   │   ├── packages.pickle
   │   └── processCcd.py
   ├── repositoryCfg.yaml
   └── schema
       ├── icSrc.fits
       └── src.fits

   8 directories, 14 files

As in the previous step, we collected the I/O activity using the ``strace(1)`` utility and then analysed its output. In the table below you can find the summary of the activity related to some of the files generated in this step. The **Read** column is the amount of data read using the ``read(2)`` system call when populating the file and analogously, the **Write** column is the amount of data written via the ``write(2)`` system call.


.. table:: Summary of the I/O activity on selected files generated by the ``processCcd.py`` command above.

   +---------------------------------------------------------+----------------+-------------+------------+
   | File Name                                               | File Size (MB) |  Read (MB)  | Write (MB) |
   +=========================================================+================+=============+============+
   | output/01318/HSC-I/output/ICSRC-0038080-023.fits        |            1   |  265        |       3    |
   +---------------------------------------------------------+----------------+-------------+------------+
   | output/01318/HSC-I/output/SRC-0038080-023.fits          |           12   |  2299       |       24   |
   +---------------------------------------------------------+----------------+-------------+------------+
   | output/01318/HSC-I/output/SRCMATCH-0038080-023.fits     |            0   |  0          |       0    |
   +---------------------------------------------------------+----------------+-------------+------------+
   | output/01318/HSC-I/output/SRCMATCHFULL-0038080-023.fits |            0   |  47         |       1    |
   +---------------------------------------------------------+----------------+-------------+------------+
   | output/01318/HSC-I/corr/BKGD-0038080-023.fits           |            0   |  1          |       0    |
   +---------------------------------------------------------+----------------+-------------+------------+
   | output/01318/HSC-I/corr/CORR-0038080-023.fits           |           98   |  13         |      98    |
   +---------------------------------------------------------+----------------+-------------+------------+
   | output/schema/icSrc.fits                                |            0   |  15         |       0    |
   +---------------------------------------------------------+----------------+-------------+------------+
   | output/schema/src.fits                                  |           0    |  0          |       0    |   
   +---------------------------------------------------------+----------------+-------------+------------+

Notice that for instance, for generating the file ``SRC-0038080-023.fits`` which has a final size of 12 MB, the process read 2299 MB, that is, 191 times the file final size. In the same way, writing 1 MB to the file ``ICSRC-0038080-023.fits`` required reading 265 MB from it, or 265 times its size.

This looks really suspicious and is likely unintended. If we look in detail what is happening at the file system level, we can see a pattern:

* write some FITS key-value pairs in the first HDU header (11520 bytes)
* **set the file position to 0**
* **read all the contents of the file written so far**
* write some data to the file (typically a FITS HDU, that is, 2880 bytes)
* **set the file position to 0**
* **read all the contents of the file written so far**
* write some data to the file (typically a FITS HDU, that is, 2880 bytes)
* **set the file position to 0**
* **read all the contents of the file written so far**
* and so on...

It is not clear why it is necessary to re-read the whole file before each write operation. But if this is the intended behavior, this may be done in a scratch area local to the compute node and copy the result to the final destination when appropriate. Given the sizes of the generated files, the amount of storage local to the compute node is unlikely to be the limiting factor.

The details of all the I/O activity on those 2 files, as reported by ``strace(1)`` is available `here <https://gist.github.com/airnandez/2a1af126c809f21b8097382502a02f31>`_.


Conclusion
==========

The work on understading the I/O activity induced by the LSST command line tasks is just starting. We consider this a very important ingredient for designing the storage infrastructure that best suits the needs of bulk LSST data processing. Initial results using the LSST software with precursor datasets show that there are several aspects of the behavior of this software that needs to be understood and fed back to the developers.


.. note::

   **This technote is not yet published.**

   In this note we present some aspects of the observed I/O behavior of the command line tasks ingestImages.py and processCcd.py when used for processing HSC data and the issues the current implementation may raise for processing data at the scale needed for LSST

.. Add content here.

.. .. rubric:: References

.. Make in-text citations with: :cite:`bibkey`.

.. .. bibliography:: local.bib lsstbib/books.bib lsstbib/lsst.bib lsstbib/lsst-dm.bib lsstbib/refs.bib lsstbib/refs_ads.bib
..    :encoding: latex+latin
..    :style: lsst_aa
