# EMF JUnit report producer correction — status note (2026-11-25)

## Problem

EMF CI runs the strict shared report gate, SWT
[`readiness.py`](https://github.com/hsoliwal/eclipse.platform.swt/blob/1202ab304c3b81b73b628d201b987941d16c94d2/tools/release/readiness.py)
at `1202ab304c3b81b73b628d201b987941d16c94d2`. It requires every declared
`<testsuite>` counter to equal the per-`<testcase>` count. Master run
[44326241353](https://github.com/hsoliwal/org.eclipse.emf/actions/runs/44326241353)
built and tested successfully, then the gate refused
`XcoreScopingTest.xml`: `declared=6 observed=7`. The report holds seven
`<testcase>` elements for six names. `scopeAllElements - Annotations.xcore`
appears once skipped and once passed, and the header says `tests=6
skipped=0`.

The cause is the report producer, not EMF. Stock Apache Surefire 3.5.6
`StatelessXmlReporter`, which Tycho 5.0.4 embeds in its OSGi booter, groups
the runs of a report by source name and method name and declares one result
per group, but writes one `<testcase>` per run. GEF hit the same defect with
inherited parameterized tests; its record
([GEF-JUNIT-DIRECT-CASE-EVIDENCE-20261118](https://github.com/hsoliwal/gef-classic/blob/b559af293c0903b22c76e418a9346e65b99a887c/docs/quality/GEF-JUNIT-DIRECT-CASE-EVIDENCE-20261118.md))
says to correct the producer and keep the strict gate.

EMF had no record or directive of its own for this. It takes the same
producer route, so both repositories keep one strict gate.

## What changed

The build change is in `.github/workflows/verify.yml` only. No EMF source,
test, POM, Maven command or verifier changes.

1. A new `report-producer` job builds the producer from source. It reads the
   two sealed patches from
   [hsoliwal/gef-classic `tests/m3/surefire356-authentic`](https://github.com/hsoliwal/gef-classic/tree/b559af293c0903b22c76e418a9346e65b99a887c/tests/m3/surefire356-authentic)
   at `b559af293c0903b22c76e418a9346e65b99a887c` and requires their Git
   blob IDs. It applies them to Apache Surefire
   `25ea054860a5c1e5932b360d8aa0a31944c2b841` (3.5.6), runs the complete
   `StatelessXmlReporterTest` class (12 cases, including the regression
   `testCountsRepeatedMethodNameInvocationsInXml`), and rebuilds the
   unchanged Tycho booter at `c4349ee27dfde2817dfc7121b5f84bfb9eeb0195`
   (5.0.4, the version in the EMF parent POM). It requires the identities
   qualified in GEF run
   [45154851672](https://github.com/hsoliwal/gef-classic/actions/runs/45154851672):
   booter JAR `73c9822161a262d9f8fa3169219f15be316a73b6f9df8ea94210eb22939a42cc`
   and embedded reporter class
   `cdd12416d7e9a0e18795f95747f28b945b7ee0f542f6f8c1d86f1ab86d97152a`.
2. Each `verify` job checks the downloaded hashes, installs the booter at
   its stock coordinates in the job's Maven local repository, runs the
   unchanged build and the unchanged strict gate, then requires that every
   test bundle's OSGi runtime configuration names that booter. It records
   the producer identities in `emf-source.txt` and removes the booter from
   `~/.m2` before the shared Maven cache is saved.

With the corrected reporter and no reruns, each report entry is declared
with its own outcome. The Xcore report is then expected to declare
`tests=7 skipped=1`, matching its seven testcases, one of them skipped.
Totals and failure handling in the gate are unchanged.

## Verification and status

- Locally (2026-11-25): the original reporter fails the regression test
  (`expected:<[2]> but was:<[1]>`); the corrected one passes all 12
  reporter tests. Driving the real stock and corrected reporters with an
  Xcore-shaped skipped-then-passed run, the unchanged strict gate refused
  the stock report and accepted the corrected one. The full EMF reactor was
  not run locally.
- CI: the pull request run is the first EMF run with this producer. Its
  result is recorded on the pull request.
- The same producer change for GEF is hsoliwal/gef-classic#47.
- This route does not depend on hsoliwal/eclipse.platform.swt#597 or
  hsoliwal/org.eclipse.emf#9.
- Rights: Apache Surefire (Apache-2.0) with the sealed patch and Tycho
  (EPL-2.0, rebuilt unchanged) are CI test infrastructure only. They are not
  redistributed with EMF.
