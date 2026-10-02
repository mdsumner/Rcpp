# Rcpp Modules namespace-load cost: findings and a patch

Date: 2026-09-10
Companion files (all against Rcpp tag 1.1.0):
- `rcpp-module-load-speedup.patch` - patch 1, R-only (`R/Module.R`)
- `rcpp-module-s4-metadata.diff` - patch 2, applies on top of patch 1
- `rcpp-module-load-combined.diff` - both together

## Summary

`loadNamespace("terra")` takes about 3.4 s on Linux. Almost none of that is
terra's own S4 layer. About 3.0 s is spent inside the `loadModule("spat", TRUE)`
load action, i.e. `Rcpp::Module()` rebuilding reference classes for terra's
15 exposed C++ classes (582 methods) from scratch on every namespace load.

Two of the steps in `Rcpp::Module()` are done twice per class. Doing each once
is an R-only change to `R/Module.R` (no C++, no API change) and removes 20-35%
of namespace-load time for every package that uses Rcpp Modules:

| Package   | classes / methods | Rcpp 1.1.0 | patched | change |
|-----------|-------------------|-----------:|--------:|-------:|
| RcppAnnoy | 5 / 85            | 0.550 s    | 0.370 s | -33%   |
| RcppBDT   | 6 / 112           | 0.838 s    | 0.664 s | -21%   |
| terra     | 15 / 582          | 3.206 s    | 2.157 s | -33%   |

(median of 7 fresh `Rscript -e 'system.time(loadNamespace(pkg))'` runs)

Tests: Rcpp `inst/tinytest/test_module.R` + `test_modref.R` 47/47 pass,
RcppAnnoy tinytest 87/87 pass, RcppBDT `tests/RcppBDT.R` runs clean, terra
spot checks (`rast`, `project`, `vect`, generators, `is(x@pntr, "Rcpp_SpatRaster")`)
behave identically.

## Environment

- Ubuntu 24.04, R 4.3.3, GDAL 3.8.4, GEOS/PROJ from apt
- Rcpp 1.1.0 built from the GitHub tag (Ubuntu's r-cran-rcpp 1.0.12 is too
  old for terra's variadic constructors)
- terra from GitHub main (1.9.51), RcppAnnoy and RcppBDT from GitHub main
- All timings on the same machine, same session, back to back

## Where the 3.4 s goes (unpatched)

Profiled with `Rprof(interval = 0.01)` around `loadNamespace("terra")` and
attributed by frame name.

| Stage                                                                  | Time     |
|------------------------------------------------------------------------|---------:|
| `loadModule("spat", TRUE)` load action, total                          | 2.97 s   |
| - `Module__classes_info` (C++ builds one `C++Class` per class, each holding one `C++OverloadedMethods` RefClass object per method, created via R-level `new()`) | 0.46 s |
| - `cpp_fields()` / `cpp_refMethods()`                                  | 0.12 s   |
| - `methods::setRefClass()` x 15 (codetools `findGlobals` over every method closure, field checks, class representation) | 1.07 s |
| - `generator$methods(initialize = ...)` x 15 (a *second* full `refClassInformation()` pass over all methods) | 0.68 s |
| - `.get_Module_Class()` -> `Module__get_class` (rebuilds the identical `C++Class` object that `classes_info` just built, including all 582 method objects again) | 0.50 s |
| terra's own S4: 382 `setGeneric` / 872 `setMethod` through `cacheMetaData` | ~0.6 s |
| `dyn.load`, lazy-load DB fetches, `.gdinit()`                          | < 0.1 s  |

Per-class cost (isolated re-run of the `Module()` loop, elapsed seconds):

| class                | methods | setRefClass | $methods(initialize) |
|----------------------|--------:|------------:|---------------------:|
| Rcpp_SpatRaster      | 283     | 0.293       | 0.271                |
| Rcpp_SpatVector      | 150     | 0.170       | 0.149                |
| Rcpp_SpatRasterStack | 31      | 0.044       | 0.030                |
| Rcpp_SpatNetwork     | 27      | 0.048       | 0.026                |
| (11 small classes)   | 1-24    | 0.014-0.036 | 0.003-0.023          |

Cost is close to linear in the number of exposed methods.

`Module::get_class()` and `Module::classes_info()` in
`inst/include/Rcpp/api/meat/module/Module.h` construct the same
`CppClass(this, cl, buffer)` object, so the second call is pure repetition.

## The patch

Two changes in `R/Module.R`, inside `Module()`:

1. Put the `initialize` method into the `methods` list that is passed to
   `methods::setRefClass()`, instead of adding it afterwards with
   `generator$methods(initialize = ...)`. `$methods()` on a generator re-runs
   `refClassInformation()` over every method of the class, so this halves the
   reference-class analysis. Behaviour is the same: if a C++ class exposed a
   method literally named `initialize`, the old code overwrote it in the
   second call and the new code overwrites it in the list.

2. In the second loop, reuse the `C++Class` objects already returned by
   `Module__classes_info` (`CLASS@generator <- generators[[clname]]`) instead
   of calling `.get_Module_Class()`, which calls `Module__get_class` and
   reconstructs the object and all of its method/field sub-objects.
   `.get_Module_Class()` has no other callers in Rcpp.

Terra effect: 3.41 / 3.40 / 3.45 s -> 2.15 / 2.08 / 2.40 s here;
2.33 / 2.59 / 2.42 s on Michael's machine.

Dirk will want a `ChangeLog` entry and an `inst/NEWS.Rd` line with the PR.
Suggested order: patch 1 as its own PR (R-only, trivially reviewable),
patch 2 as a second PR once the first is in.

## Patch 2: plain S4 objects for module metadata

After patch 1, `Module__classes_info` still cost ~0.45-0.68 s for terra.
`C++OverloadedMethods`, `C++Field` and `C++Constructor` were *reference
classes*, and C++ created one object per exposed method / field /
constructor with `Rcpp::Reference("...")`, which evaluates R-level `new()`.
`new()` on a reference class with eight fields costs ~0.4-0.5 ms (creates
the environment, `.self`, `.refClassDef`, an `uninitializedField` object per
field, `is()` checks on every `$<-`); on an equivalent S4 class it is ~10 us,
and `Rcpp::S4("...")` bypasses R-level `new()` altogether via
`R_do_new_object`.

Microbenchmark, 582 objects: RefClass `new()` 0.286 s, S4 `new()` 0.005 s.
`Module__classes_info` on terra: 0.68 s -> 0.006 s.

The change:

- `inst/include/Rcpp/Module.h`: `S4_CppOverloadedMethods`, `S4_field` and
  `S4_CppConstructor` derive from `Rcpp::S4` instead of `Rcpp::Reference`
  and populate slots (`slot("x") = ...`, i.e. `R_do_slot_assign`) instead
  of fields (`field("x") = ...`, i.e. an R-level `$<-` call).
- `R/00_classes.R`: the three classes become `setClass()` with the same
  slot names. The `info()` reference method of `C++OverloadedMethods`
  becomes an internal function `.cpp_methods_info(x, prefix)`.
- `R/Module.R`, `R/01_show.R`: the only consumers (`method_wrapper`,
  `binding_maker`, `show` for `C++Class`) use `@` instead of `$`.
- Compatibility: `$` and `$<-` methods are defined for the three classes,
  mapping to slot access; `x$info(prefix)` still works. The `$<-` method
  is what keeps *binaries compiled against older Rcpp headers* loadable:
  their `FieldProxy::set` evaluates `` `$<-`(obj, name, value) `` and stores
  the result, which now lands in the slot. Verified by loading an RcppAnnoy
  built against unpatched headers into the patched Rcpp: loads, 87/87
  tests pass. Without the `$<-` method the old binary fails at load with
  "no method for assigning subsets of this S4 class".
- The three Rd files updated (Slots instead of Fields, the `$` methods,
  a note about the change).

`R CMD check` on the patched package (no tests/vignettes here): clean
apart from environment noise (locale, unavailable Suggests).

## Results with both patches

Same machine, one run, medians of 7 fresh `Rscript` processes. (The VM was
slower in this run than in the earlier table; compare within a row.)

| Package   | Rcpp 1.1.0 | patch 1 | patch 1+2 | total change |
|-----------|-----------:|--------:|----------:|-------------:|
| RcppAnnoy | 0.690 s    | 0.475 s | 0.359 s   | -48%         |
| RcppBDT   | 1.012 s    | 0.901 s | 0.770 s   | -24%         |
| terra     | 4.089 s    | 2.784 s | 2.064 s   | -50%         |

Tests on patch 1+2: Rcpp module tinytests 47/47, RcppAnnoy 87/87 (both
fresh build and old-header build), RcppBDT test script clean, terra spot
checks identical, `show(terra:::SpatExtent)` and `$`-style access on the
metadata objects work.

## What is left after both patches

terra at ~2.05 s, of which:

- ~1.0 s `methods::setRefClass()` doing codetools analysis of 582 method
  closures (`refClassInformation` -> `insertClassMethods` ->
  `makeClassMethod` -> `codetools::findGlobals`). This is the `methods`
  package's cost. Only reducible by exposing fewer methods or by not
  generating reference classes at load time (install-time caching with
  pointer re-patching, a redesign).
- ~0.6 s terra's own S4 metadata caching. Ordinary for that many generics
  and methods; group generics (`Arith`, `Compare`, ...) show up in
  `.checkGroupSigLength` but nothing there looks worth changing.
- ~0.15 s `cpp_refMethods`/`method_wrapper` building 582 closures with
  `substitute`, dyn.load, lazy-load fetches.

## terra-side options considered

- Exposing fewer methods in `src/RcppModule.cpp` is the only lever terra has;
  the cost is roughly 2.5 ms per method after the patch. Probably not worth
  the churn.
- Lazy module loading (defer `loadModule` to first use) is not viable:
  `Module()` falls back to defining the reference classes in `.GlobalEnv`
  when `where` is a locked namespace.
- Install-time caching of the reference classes would need every method and
  class external pointer re-patched at load. That is an Rcpp redesign, not a
  terra change.
- Once the Rcpp change ships, terra only needs `Rcpp (>= <that version>)` in
  DESCRIPTION.

## Reproducing

```sh
# build Rcpp from tag, apply the patches, install to a separate library
git clone --depth 1 --branch 1.1.0 https://github.com/RcppCore/Rcpp.git
cd Rcpp && git apply ../rcpp-module-load-combined.diff   # or patch 1 alone
R CMD INSTALL --library=../rlib-patched .

# then install terra / RcppAnnoy / RcppBDT into that library and time.
# Patch 2 changes headers: rebuild the module packages with --preclean,
# stale .o files compiled against the old headers will otherwise be linked
# (they still load, via the compat $<- method, but you will not see the gain).
R_LIBS=../rlib-patched R CMD INSTALL --preclean --library=../rlib-patched terra
R_LIBS=../rlib-patched Rscript -e 'system.time(loadNamespace("terra"))'
```

Profiling attribution used:

```r
Rprof("prof.out", interval = 0.01); loadNamespace("terra"); Rprof(NULL)
l <- readLines("prof.out")[-1]
has <- function(p) sum(grepl(p, l, fixed = TRUE)) / 100
sapply(c('"loadModule"', '"setRefClass"', '"generator$methods"',
         '".get_Module_Class"', '"codetools::findGlobals"'), has)
```

Rcpp module tests against the patched build:

```r
library(tinytest); library(Rcpp); Sys.setenv(RunAllRcppTests = "yes")
run_test_dir("inst/tinytest", pattern = "test_mod.*\\.R$")
```

## Other Rcpp Modules packages in Dirk's stable

RcppAnnoy (5 classes / 85 methods), RcppBDT (6 / 112), RcppRedis (1 / 53,
needs Rcpp >= 1.1.1 and RApiSerialize, not measured). RcppExamples,
RcppCNPy, RcppSimdJson, RcppParallel, RcppQuantuccia do not define modules.
RcppAnnoy is the useful small example: loaded transitively (uwot etc.) and a
third of its startup was Rcpp doing the reference-class work twice.
