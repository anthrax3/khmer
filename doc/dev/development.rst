..
   This file is part of khmer, https://github.com/dib-lab/khmer/, and is
   Copyright (C) 2012-2015 Michigan State University
   Copyright (C) 2015-2016 The Regents of the University of California.
   It is licensed under the three-clause BSD license; see LICENSE.
   Contact: khmer-project@idyll.org

   Redistribution and use in source and binary forms, with or without
   modification, are permitted provided that the following conditions are
   met:

    * Redistributions of source code must retain the above copyright
      notice, this list of conditions and the following disclaimer.

    * Redistributions in binary form must reproduce the above
      copyright notice, this list of conditions and the following
      disclaimer in the documentation and/or other materials provided
      with the distribution.

    * Neither the name of the Michigan State University nor the names
      of its contributors may be used to endorse or promote products
      derived from this software without specific prior written
      permission.

   THIS SOFTWARE IS PROVIDED BY THE COPYRIGHT HOLDERS AND CONTRIBUTORS
   "AS IS" AND ANY EXPRESS OR IMPLIED WARRANTIES, INCLUDING, BUT NOT
   LIMITED TO, THE IMPLIED WARRANTIES OF MERCHANTABILITY AND FITNESS FOR
   A PARTICULAR PURPOSE ARE DISCLAIMED. IN NO EVENT SHALL THE COPYRIGHT
   HOLDER OR CONTRIBUTORS BE LIABLE FOR ANY DIRECT, INDIRECT, INCIDENTAL,
   SPECIAL, EXEMPLARY, OR CONSEQUENTIAL DAMAGES (INCLUDING, BUT NOT
   LIMITED TO, PROCUREMENT OF SUBSTITUTE GOODS OR SERVICES; LOSS OF USE,
   DATA, OR PROFITS; OR BUSINESS INTERRUPTION) HOWEVER CAUSED AND ON ANY
   THEORY OF LIABILITY, WHETHER IN CONTRACT, STRICT LIABILITY, OR TORT
   (INCLUDING NEGLIGENCE OR OTHERWISE) ARISING IN ANY WAY OUT OF THE USE
   OF THIS SOFTWARE, EVEN IF ADVISED OF THE POSSIBILITY OF SUCH DAMAGE.

   Contact: khmer-project@idyll.org

Development miscellany
======================

Third-party use
---------------

We ask that third parties who build upon the codebase to do so from a
versioned release. This will help them determine when bug fixes apply and
generally make it easier to collaborate. If more intensive modifications happen
then we request that the repository is forked, again preferably from a version
tag.

Build framework
---------------

`make` should build everything, including tests and "development" code.

git and GitHub strategies
-------------------------

Still in the works, but read `this
<http://scottchacon.com/2011/08/31/github-flow.html>`__.

Make a branch on dib-lab (preferred so others can contribute) or fork the
repository and make a branch there.

Each piece or fix you are working on should have its own branch; make a pull-
request to dib-lab/master to aid in code review, testing, and feedback.

If you want your code integrated then it needs to be mergable

Example pull request update using the command line:

 #. Clone the source of the pull request (if needed)
     ``git clone git@github.com:mr-c/khmer.git``
 #. Checkout the source branch of the pull request
     ``git checkout my-pull-request``
 #. Pull in the destination of the pull request and resolve any conflicts
     ``git pull git@github.com:dib-lab/khmer.git master``
 #. Push your update to the source of the pull request ``git push``
 #. Jenkins will automatically attempt to build and test your pull requests.

Code coverage
-------------

Jenkins calculates code coverage for every build. Navigate to the results from
the master node first to view the coverage information.

Code coverage should never go down and new functionality needs to be tested.

Pipelines
---------

All khmer scripts used by a published recommended analysis pipeline must be
included in ``scripts/`` and meet the standards therein implied.

Command line scripts
--------------------

Python command-line scripts should use '-' instead of '_' in the name.
(Only filenames containing code for import should use _.)

Please follow the command-line conventions used under scripts/.  This
includes most especially standardization of '-x' to be hash table size,
'-N' to be number of hash tables, and '-k' to always refer to the
k-mer size.

Command line thoughts:

   If a filename is required, typically UNIX commands don't use a flag to
   specify it.

   Also, positional arguments typically aren't used with multiple files.

   CTB's overall philosophy is that new files, with new names, should
   be created as the result of filtering etc.; this allows easy
   chaining of commands.  We're thinking about how best to allow
   override of this, e.g. ::

      filter-abund.py <ct file> <filename> [ -o <filename.keep> ]

----

All code in ``scripts/`` must have automated tests; see
``tests/test_scripts.py``. Otherwise it belongs in ``sandbox/``.

When files are overwritten, they should only be opened to be overwritten
after the input files have been shown to exist.  That prevents stupid
command line mistakes from trashing important files.

A general error should be signaled by exit code `1` and success by `0`. Linux
supports exit codes from `0` to `255` where the value `1` means a general
error. An exit code of `-1` will get converted to `255`.

----

CLI reading:

   http://stackoverflow.com/questions/1183876/what-are-the-best-practices-for-implementing-a-cli-tool-in-perl

   http://catb.org/esr/writings/taoup/html/ch11s06.html

   http://figshare.com/articles/tutorial_pdf/643388

Python / C integration
----------------------

The Python extension that wraps the C++ core of khmer lives in
``khmer/_khmer.cc``

This wrapper code is tedious and annoying so we use a static analysis tool to
check for correctness.

https://gcc-python-plugin.readthedocs.io/en/latest/cpychecker.html

Developers using Ubuntu Precise will want to install the gcc-4.6-plugin-dev
package

Example usage: ::

	CC="/home/mcrusoe/src/gcc-plugin-python/gcc-python-plugin/gcc-with-cpychecker
	--maxtrans=512" python setup.py build_ext 2>&1 | less

False positives abound: ignore errors about the C++ standard library. This tool
is primarily useful for reference count checking, error-handling checking, and
format string checking.

Errors to ignore: "Unhandled Python exception raised calling 'execute' method",
"AttributeError: 'NoneType' object has no attribute 'file'"

Warnings to address: ::

        khmer/_khmer.cc:3109:1: note: this function is too complicated
        for the reference-count checker to fully analyze: not all paths were
        analyzed

Adjust --maxtrans and re-run. ::

	khmer/_khmer.cc:2191:61: warning: Mismatching type in call to
	Py_BuildValue with format code "i" [enabled by default]
	  argument 2 ("D.68937") had type
	    "long long unsigned int"
	  but was expecting
	    "int"
	  for format code "i"

See below for a format string cheat sheet One also benefits by matching C type
with the function signature used later.

"I" for unsigned int
"K" for unsigned long long a.k.a khmer::HashIntoType.

Read handling
-------------

Several bugs have gone unnoticed due to inconsistencies in read handling.
On the C++ side, there are an abundance of ``consume`` functions for loading
Fasta/Fastq sequences. On the Python side, read handling is sometimes delegated
to the C++ library, and sometimes handled in Python using screed.

In an attempt to normalize read handling in Python, the functions in
``khmer/utils.py`` should be used whenever possible.  Here,
``broken_paired_reader`` in ``khmer/utils.py`` should be used to do all
paired-end sequence handling, and sequence loading should
go through ``khmer.utils.clean_input_reads(iter)``; this is a
generator that wraps the iterator produced by ``screed.open``, and it
adds a ``cleaned_seq`` attribute to screed ``Record`` objects.  This
attribute should be used for any k-mer or graph operations, while
the normal ``sequence`` attribute is what should be written out.
``write_record`` and ``write_record_pair`` should be used to output
records.  All of these functions are aware of FASTA and FASTQ records,
too.

For applying operations to collections of reads, the ``ReadBundle`` class is
available.  This is used to wrap a collection of reads for examination and
processing in situations where (for example) something should be done to
either both reads in a pair, or neither.

Some basic rules of sequence handling in khmer are:

* consume and produce "broken paired" format, such that pairs of sequences
  always stay together; see ``khmer.utils.broken_paired_reader``.

* when looking at the coverage of reads (for trimming or digital normalization)
  always consider pairs; see ``khmer.utils.ReadBundle(...)``.

* only apply graph or k-mer operations to sequences consisting only of ATCG;
  typically this will be ``record.cleaned_seq``.  See
  ``khmer.utils.clean_input_read(...)``.
