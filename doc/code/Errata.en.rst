.. Licensed to the Apache Software Foundation (ASF) under one
   or more contributor license agreements. See the NOTICE file distributed with this work for
   additional information regarding copyright ownership. The ASF licenses this file to you under the
   Apache License, Version 2.0 (the "License"); you may not use this file except in compliance with
   the License. You may obtain a copy of the License at

   http://www.apache.org/licenses/LICENSE-2.0

   Unless required by applicable law or agreed to in writing, software distributed under the License
   is distributed on an "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express
   or implied. See the License for the specific language governing permissions and limitations under
   the License.

.. include:: ../common-defs.rst

.. _swoc-errata:
.. highlight:: cpp
.. default-domain:: cpp
.. |Errata| replace:: :code:`Errata`
.. |Rv| replace:: :code:`Rv`

******
Errata
******

|Errata| is an error reporting class. The goals are

*  Very fast in the success case.
*  Easy to accumulate error messages in a failure case.

An assumption is that error handling is always intrinsically expensive, in particular if errors are
reported or logged. Therefore in the error case it is better to be comprehensive and easy to use.
Conversely, the success case should be optimized for performance because it is all overhead.

If errors are to be reported, they should be as detailed as possible. |Errata| supports by allowing
error messages to accumulate. This means detailed error reports do not require passing large amounts
of context to generic functions so that errors there can be reported in context. Instead the calling
functions can add their local details to the error stack, passing these back to their callers. The
end result is both what exactly went wrong at the lowest level and the context in which the failure
occurred. E.g., for a failure to open a file, the file open logic can report the direct error
(e.g. "Permission denied") and the path, while the higher level function such as a configuration
file parser, can report it was the configuration file open that failed.

Definition
**********

.. code-block::

   #include <swoc/Errata.h>

.. class:: Errata

   :libswoc:`Reference documentation <Errata>`.

Usage
*****

The default |Errata| constructor creates an empty, successful state. This is extremely fast. An
error state is created by constructing an instance with an annotation and optional error code and severity.

The basic interface is provided by the :func:`Errata::note` method and variants thereof. This adds a
message along with a optional severity. Using |Errata| requires defining the default severity and
"failure" severity. The failure severity is the severity for which an |Errata| is considered to be
an error. There is also a "filter" severity - annotations that have a severity that is less than
this value are not added but are ignored. This can be overridden so that an application can
dynamically set the value to capture only annotations that are of sufficient interest.

An |Errata| instance also carries a error code which is intended to distinguish among errors of the
same severity in an easy to examine way. For interoperability with standard error handling
:code:`std::error_code` is used as the identifier. This allows constructing an instance from the
error return of system functions and for callers to check if the error is really an error. Note,
however, that severity and error code are independent.

|Errata| provides the :libswoc:`Rv` template which is intended for passing back return values or
error reports. The template parameter is the return type, to which is added an |Errata| instance.
The |Rv| instance implicitly converts to the return type so that if a function is changed from
returning :code:`T` to :code:`Rv<T>` the callers do not need to be changed.

Severity
========

The severity support has been a bit of an issue. Although the support seems a bit convoluted, in
practice users of |Errata| tend to have an already defined severity scale. The goal is to be able
to integrate that existing scale easily in to |Errata|.

For the purposes of the unit testing and example code the following is included to define the
severity levels. This is modeled as usual on the ``syslog`` severity levels.

.. code-block::

   static constexpr swoc::Errata::Severity ERRATA_DBG{0};
   static constexpr swoc::Errata::Severity ERRATA_DIAG{1};
   static constexpr swoc::Errata::Severity ERRATA_INFO{2};
   static constexpr swoc::Errata::Severity ERRATA_WARN{3};
   static constexpr swoc::Errata::Severity ERRATA_ERROR{4};

It is expected the application would already have the equivalent definitions and would not need any
special ones for |Errata|. The severity values are initialized during start up.

These levels can also be named. It is presumed the severity values are zero based and compact and
therefore the names can be provided in an instance of code:`MemSpan<TextView>`. Severity values outside
the span are printed as numbers. For the unit tests the names are declared as a global value.

.. code-block::

   std::array<swoc::TextView, 5> Severity_Names { {
     "Debug", "Diag", "Info", "Warn", "Error"
   }};

The initialization is done in :code:`test_Errata_init` which is called from :code:`main`.

.. code-block::

   void test_Errata_init() {
     swoc::Errata::DEFAULT_SEVERITY = ERRATA_ERROR;
     swoc::Errata::FAILURE_SEVERITY = ERRATA_WARN;
     swoc::Errata::SEVERITY_NAMES = swoc::MemSpan<swoc::TextView>(Severity_Names.data(), Severity_Names.size());
   }

If there is no external initialization, then there are three levels of severity 0..2 with
the names "Info", "Warning", and "Error". The default severity is "Error" (2) with a failure threshold
of 2 ("Error").

By default annotations do not have a severity, that is a property of the :code:`Errata`. However a
severity can be added to an annotation. If this is done the the severity of the :code:`Errata` is
updated to that severity if it is larger (more severe) than the current severity. Annotations can be
filtered by adjusting the value of :code:`swoc::Errata::FILTER_SEVERITY`. If a severity is provided
with an annotation (via some variant of the :code:`note` method) then this is checked against
:code:`FILTER_SEVERITY` and if it is less than that the annotation is not added. This enables an
application to dynamically control the verbosity of errors without changing how they are generated.
The :code:`Errata` serverity is updated as appropriate even if the annotation is discarded.

Examples
========

There are two fundamental cases for use of |Errata|. The first is the leaf function that creates
the original |Errata| instance. The other case is code which calls an |Errata| returning function.
Loading a configuration file will be used as an example.

A leaf function could be one that, given a path, opens the file and loads it in to a :code:`std::string`
using :libswoc:`file::load`.

.. code-block::

   Errata load_file(swoc::path const& path) {
      std::error_code ec;
      std::string content = swoc::file::load(path, ec);
      if (ec) {
         return Errata(ec, ERRATA_ERROR, "Failed to open file {}.", path);
      }
      // config parsing logic
   }

The call site might look like

.. code-block::

   Errata load_config(swoc::path const& path) {
      if (Errata errata = this->load_file(path) ; ! errata.is_ok()) {
         return std::move(errata.note("While opening configuration file."));
      }
      // ... more code.

Printing the result would look like::

   Error Failed to open file thing.txt EPERM [1].
      While opening configuration file.

While some functions will return |Errata| the more common case is to use |Rv| to carry back either a
value (successful) or an error report (failure).

Extending
=========

There are some variants of :code:`note` which are intended for use by "helper" methods. The most
common example would be the desire to have a function that creates an "Error" level :code:`Errata`.
This could be done with

.. literalinclude:: ../../unit_tests/test_Errata.cc
   :start-after: DOC -> NoteInfo
   :end-before: DOC -< NoteInfo

This can then be used on an instance like::

   NoteInfo(errata, "Looking at {} values.", count);

Design Notes
************

I have carted around variants of this class for almost two decades, the evolution being driven
primarily by the evolving capabilities of C++ rather than a fundamental change in the design
philosophy. The original impetus was as noted in the introduction, to be able to generate a
very detailed and thorough error report on a failure, which as much context as possible. In some
sense, to have a stack trace without actually crashing.

Recently (Jan 2022) I've restructured |Errata| based on additional usage experience.

*  The error code was removed for a while but has been added back. Although not that common, it is
   not rare that having a code is useful for determining behavior for an error. The most common
   example is :code:`EAGAIN` and equivalent - the caller might well want to log and try again rather
   than immediately fail. To be most useful the error code was made to be :code:`std::error_code`
   which covers both C++ error codes and standard "errno" errors.

*  The error code and severity was moved from messages to the |Errata| instance. The internals were
   already promoting the highest severity of the messages to be the overall severity, and error codes
   for messages other than the front message seemed useless. However, this was changed again due to
   user request so that annotation serverity is available but optional. This can be used for
   additional display (where annotations are printed with the severity, if present) and for filtering
   (only sufficiently severe annotations are added).

*  The reference counting was removed. With the advent of move semantics there is much less need
   to make cheap copies with copy on write support. Because of the change from general memory allocation to
   use of an internal `MemArena` it is no longer possible to share messages between instances which
   makes the copy on write use of the reference count useless. For these two reasons reference was
   removed along with any copying. Use must either explicitly copy via the :code:`note` method or
   "move" the value, which intermediate functions must now use. I think this is worth that cost
   because there are edge cases that are hard to handle with reference counting which don't arise
   with pure move semantics.

*  Annotations are now appended instead of prepended. A big change but in many cases the order was
   already messed up because I think this was assumed but sometimes accomodated. It's debatable which
   is the better style but overall append makes some small bits cleaner.
