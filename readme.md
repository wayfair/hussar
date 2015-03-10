# hussar

This project consists of a handful of scripts that support usage of the static
analysis capabilities of HHVM within a PHP codebase. The scripts ultimately
produce a formatted report of errors discovered in the analysis, but are also
used to prepare the codebase and to directly compare reports.


### Quick start

  1. Clone this repository: `git clone git@github.com:wayfair/hussar`
  2. Edit `config.example` with your project's details and save as `config`.
  3. If necessary for your project, add a file named `local-patch` to override
     the default functions provided in `lib/patching`.
  4. `cd` to the root of your PHP codebase
  5. Run `/path/to/hussar/bin/main -F -r master`
  6. Run `less /path/to/hussar/reports/latest` to view the report

  > **Note:**
  >
  > This project requires `hhvm <= 3.3.x`. The `-t analyze` flag is not present
  > in more recent versions of HHVM and there does not appear to be equivalent
  > functionality for analyzing PHP code (see [issue-3987][]).


### Requirements

`hussar` was developed in a GNU/Linux environment and uses many standard
utilities that we do not list here (eg, `find`, `grep`, `rsync`, `mkfifo` etc).
The following additional packages are required to use the tool:

   * `hhvm <= 3.3.x` (see note, above)
   * `bash >= 4.0` (for associative array support)
   * `git` (not required for all scripts, see *Usage* section below)
   * `php` (for building the `.class-index` and parsing HHVM output)
   * `perl` (for patching files in place)


### Motivation

PHP projects that are unable to run directly on HHVM can still take advantage of
identifying errors found by its strong typing and static analysis of code.
Unfortunately the output provided by HHVM isn't so friendly, and some prep work
is required before analysis can be performed at all since the static analyzer
lacks support for some common PHP functionality (eg, autoloading). The purpose
of this project is to facilitate use of the static analysis capabilities of HHVM
so that it can be included as part of a regular test, build, and deploy cycle.


### Usage

The scripts under `bin/` are designed to be able to run independently, with the
`bin/main` script wrapping the functionality of the others. For typical PHP
projects using `git`, try the following commands:

``` bash
# Commands expect to run within the root of the codebase
$ cd /path/to/codebase

# Analyze all files in the codebase
$ /path/to/hussar/bin/main -F

# View the latest full report
$ less /path/to/hussar/reports/latest
```

Projects are likely to have a number of errors in their codebase and developers
shouldn't have to see the full list every time. The tool can restrict output to
just those errors that might be introduced by a developer's changes:

``` bash
# Analyze for errors in a patchfile (compared to origin/master)
$ /path/to/hussar/bin/main -p /path/to/file.patch

# Analyze for errors in a branch (compared to merge-base with origin/master)
$ /path/to/hussar/bin/main -r feature/branch -t 'fork-point'
```

Options can be combined to handle more complex cases:

``` bash
# Analyze file.patch applied to feature/branch and compare to origin/master:
$ /path/to/hussar/bin/main -p /path/to/file.patch -r feature/branch

# As above, but compare to feature/base instead:
$ /path/to/bin/main -p /path/to/file.patch -r feature/branch -t feature/base
```

For these more complex uses, it can be helpful to enable debugging output:

``` bash
# Enable debugging messages
$ /path/to/hussar/bin/main -d -p /path/to/file.patch
```

Projects not using `git` can still take advantage of the tool. Patch the
codebase manually (see next section) and run the `bin/make-report` command:

``` bash
# Make report for default branch
$ hg up default
$ /path/to/hussar/bin/make-report -p -r $(hg id)
```

The `bin/make-diff-report` script should easily translate to other source
control systems but is not currently supported outside `git`. The other scripts
are more tightly coupled to `git` commands, but it should be possible to
translate them as well.


### Details

#### Limitations of HHVM static analysis

Being *static* analysis, HHVM requires a few codebase modifications to support
some standard PHP features and make the results of the analysis meaningful.
Applying these modifications is the responsibility of the `bin/patch-files`
script, which modifies files to correct for the following limitations:

   * **Autoprepend and Autoloading:**

     The `auto_prepend_file` directive is not supported when running `hhvm`
     with the `-t analyze` flag (see [issue-2990][]). Autoloading is also
     unsupported (see [issue-253][]) and classes meant to be included via
     `spl_autoload_register` or `hh\autoload_set_paths` are instead reported as
     `UnknownClass`. These limitations result in a cascade of false-positive
     errors for any files that depend on those features. To resolve these
     issues, the script inserts hard `require_once` statements into every file,
     using a regex to identify what classes a file contains.

   * **PHP incompatibilities and third-party packages:**

     The HHVM team has done a great job reimplementing PHP functionality, but
     there are still [a number of incompatibilities][incompatibilities] beyond
     the two described above, and often projects use third-party code that can
     clutter reports. When present, the script adds `require_once` statements
     for all files under the `stubs/` directory to the `auto_prepend.php` script
     so that these objects are always defined.

   * **Runtime defined state:**

     The whole point of static analysis is to discover errors without executing
     the program, so it's no surprise that HHVM emits some false-positive errors
     for states known only at runtime. Constants whose values are defined within
     conditional expressions produce `DeclaredConstantTwice` errors when in
     reality only one branch would be taken at runtime. HHVM does not store the
     value of conditionally defined constants, so concatenating such constants
     with strings (as in `require_once PROJECT_ROOT . '/path/to/file.php';`)
     results in `UnknownClass` or `UndefinedFunction` errors. The script can be
     configured to patch such constants to static string values.

[issue-3987]: https://github.com/facebook/hhvm/issues/3987
[issue-2990]: https://github.com/facebook/hhvm/issues/2990
[issue-253]: https://github.com/facebook/hhvm/issues/253
[incompatibilities]: https://github.com/facebook/hhvm/issues?q=is%3Aopen+is%3Aissue+label%3A%22php5+incompatibility%22

#### About the `.class-index` and file patch caches

Patching and analyzing a large codebase can take several minutes, a delay that
discourages developers from using the tool to avoid interrupting their workflow.
While `hussar` can't do much to improve the run time of `hhvm`, it employs a few
caches to reduce the time required to patch files. Both caches are created by
default during the first invocation of `hussar`.

The first cache is the `.class-index`, which is a simple mapping from a fully
qualified class name to its corresponding file. This is encoded as a Bash
associative array and sourced before patching files. Without this cache, class
files must be "autoloaded" at patch time, which is very expensive because it
requires spinning up a `php` process for every file to be patched.

``` bash
# Make the .class-index file
$ /path/to/hussar/bin/main -c
```

The second cache is the patch cache, which is a full copy of the codebase after
having patched all files. This cache is built via `rsync` and cuts the time
required to patch files down to a few seconds by reducing the number of files to
be patched to just those changed or introduced in a revision or patchfile.

``` bash
# Fully patch the codebase and update the patch cache to match
$ /path/to/bin/main -F -u
```

#### Maintaining the workspace

As described in the *Usage* section above, `hussar` tries to be flexible with
what can be analyzed. The `bin/prepare-workspace` and `bin/patch-files` scripts
work together to ensure that the report is actually performed against the
specified revisions. Getting this right is tricky and requires keeping track of
the hashes for `origin/master`, the patch cache, the revision, the comparand,
and the patchfile (modulo which options are present).

The former script maintains the workspace as necessary by reseting HEAD and
pulling down the patch cache, then replacing the cached versions of files with
their equivalent version in the specified revision or patchfile. Patchfiles are
then committed so that `bin/make-report` can save the report under the new
revision.  If the patch cache is ahead of the specified revision, files *not*
present in the revision are removed. If the patch cache is behind, files *added*
in the revision are checked out to the index as new files. Upon completion, all
files not modified by the revision or patchfile are modified in the working tree
to their patch cache equivalents; that is, the workspace is identical to the
revision (or patchfile applied to the revision) but with patched versions of
nearly all files.

The latter script patches any files deemed necessary to patch (as described
above in *Limitations of HHVM static analysis*). Passing the `-F` option
specifies all files in the codebase. When a revision is specified, only the
files modified between `HEAD` and the revision are patched. This is precisely
where `bin/prepare-workspace` leaves off, so that by the end of these two
scripts all files in the codebase have been patched while otherwise reflecting
their contents at the specified revision (plus patchfile).


### License

`hussar` is distributed with an ISC License. See LICENSE for details.
