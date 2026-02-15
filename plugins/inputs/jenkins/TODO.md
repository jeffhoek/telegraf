# PR Review TODO

Fixes for `fix: inputs.jenkins: Report all concurrent builds` (commit 8d89a81)

## 1. Bound build fetching with `tree` query parameter

The loop in `getJobDetail` iterates over `js.Builds` and makes a `getBuild()` HTTP call per entry. Jenkins returns all historical builds by default, so for jobs with thousands of builds this fires unbounded HTTP requests until one exceeds `MaxBuildAge`. A hardcoded cap of ~20 via the `tree` parameter is sufficient since `MaxBuildAge` remains the user-facing control.

- [ ] Write a test with a job that has more than 20 builds to verify only the capped number of builds are fetched
- [ ] Add `tree=builds[number]{0,20}` to the `getJobs` API call so Jenkins limits the `builds` array server-side
- [ ] Run tests and confirm the build count is bounded

## 2. Improve error resilience in build fetch loop

In `jenkins.go:309-311`, a single `getBuild()` failure aborts the entire job and no metrics are emitted for any builds. This is inconsistent with sub-job error handling at line 282 which uses `acc.AddError`.

- [ ] Write a test where one `getBuild()` call returns an error but other builds succeed, and assert metrics are still emitted for the successful builds
- [ ] Replace `return err` with `acc.AddError(err)` and `continue` so remaining builds are still processed
- [ ] Run tests and confirm partial failures no longer drop all metrics for the job

## 3. Validate newest-first ordering assumption

`jenkins.go:319` uses `break` assuming builds are newest-first, meaning a single out-of-order build would silently drop all subsequent builds. The invariant is not enforced in code.

- [ ] Write a test with builds in non-descending order and assert all valid builds within `MaxBuildAge` are reported
- [ ] Sort the builds slice by number descending before iterating to guarantee the newest-first invariant
- [ ] Run tests and confirm out-of-order builds are handled correctly

## 4. Add test coverage for edge cases

`TestGatherJobsMultipleBuilds` only covers the happy path of multiple completed builds. Several important code paths are untested.

- [x] Add test cases: a running build (should be skipped), a build older than `MaxBuildAge` (should stop iteration), empty `Builds` with valid `LastBuild` (fallback path), and empty `Builds` with `LastBuild.Number < 1` (no builds)
- [ ] Verify existing code handles each case correctly; fix any issues found
- [ ] Run tests and confirm all edge cases pass
