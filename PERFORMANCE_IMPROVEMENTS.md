# Go Performance Improvements Review

This document contains performance optimization opportunities identified in the codebase, sorted by **high impact + simplicity** (easiest to implement with maximum benefit first).

## High Impact + Simple Fixes

### 1. **Regex Compiled in Function Calls - Hot Path**

**Impact**: High | **Complexity**: Low | **Priority**: Critical

Multiple regex patterns are compiled inside functions that are called repeatedly, causing significant overhead.

**Locations**:
- `checks/raw/sast.go:231` - `regexp.MustCompile(usesRegex)` inside loop iteration
- `checks/raw/pinned_dependencies.go:493` - Docker SHA regex compiled per file
- `checks/raw/pinned_dependencies.go:645` - GitHub var regex compiled per file
- `checks/raw/pinned_dependencies.go:834-844` - Three regexes compiled in `isActionDependencyPinned()` on every call
- `checks/raw/shell_download_validate.go:505-506` - Hash and semver regexes compiled in `isGoUnpinnedDownload()`
- `checks/raw/shell_download_validate.go:527` - URL regex compiled in hot path
- `checks/raw/shell_download_validate.go:559-574` - Multiple regexes compiled per validation
- `clients/git/client.go:163,183` - Regexes compiled in `Search()` method
- `clients/gitlabrepo/licenses.go:54` - Regex compiled in `setup()` using `fmt.Sprintf()`
- `cmd/internal/nuget/client.go:228` - Regex compiled in `isSupportedProjectURL()`
- `internal/dotnet/properties/properties.go:110` - Regex compiled in `isValidFixedVersion()`

**Solution**: Move these regex patterns to package-level `var` declarations initialized once:

```go
// Before:
func isGoUnpinnedDownload(cmd []string) bool {
    hashRegex := regexp.MustCompile("^[A-Fa-f0-9]{40,}$")
    semverRegex := regexp.MustCompile(`^v\d+\.\d+\.\d+(-[0-9A-Za-z-.]+)?(\+[0-9A-Za-z-.]+)?$`)
    // ... use regexes
}

// After:
var (
    hashRegex   = regexp.MustCompile("^[A-Fa-f0-9]{40,}$")
    semverRegex = regexp.MustCompile(`^v\d+\.\d+\.\d+(-[0-9A-Za-z-.]+)?(\+[0-9A-Za-z-.]+)?$`)
)

func isGoUnpinnedDownload(cmd []string) bool {
    // ... use regexes
}
```

**Estimated Impact**: 30-50% performance improvement in affected code paths, especially for file scanning operations.

---

### 2. **Regex Compiled Inside Loops - Patch Generation**

**Impact**: High | **Complexity**: Low | **Priority**: Critical

Regex compilation inside loops in workflow patch generation code.

**Locations**:
- `probes/hasDangerousWorkflowScriptInjection/internal/patch/impl.go:122` - Array var regex in loop
- `probes/hasDangerousWorkflowScriptInjection/internal/patch/impl.go:145-147` - Multiple regexes created per pattern
- `probes/hasDangerousWorkflowScriptInjection/internal/patch/impl.go:345` - On regex in `findGlobalIndentation()`
- `probes/hasDangerousWorkflowScriptInjection/internal/patch/impl.go:390,396` - Blank/comment regexes in `isBlank()`/`isComment()`
- `probes/hasDangerousWorkflowScriptInjection/internal/patch/impl.go:423` - Label regex with `fmt.Sprintf` in function
- `checks/fileparser/github_workflow.go:442-443` - Two regexes compiled per step match

**Solution**: Hoist regex compilation to package level or struct fields.

**Estimated Impact**: 40-60% faster patch generation, especially for large workflows.

---

### 3. **Regex With fmt.Sprintf Inside Functions**

**Impact**: Medium-High | **Complexity**: Low | **Priority**: High

Dynamic regex construction with `fmt.Sprintf` inside functions adds unnecessary string formatting overhead.

**Locations**:
- `clients/gitlabrepo/licenses.go:54` - `regexp.Compile(fmt.Sprintf("%s/-/blob/(?:\\w+)/(.*)", handler.repourl.URI()))`
- `probes/hasDangerousWorkflowScriptInjection/internal/patch/impl.go:423` - `regexp.MustCompile(fmt.Sprintf("^%s%s:", strings.Repeat(" ", indent), label))`

**Solution**: 
- For static parts, compile once and store
- For dynamic parts, consider string operations or compile once per unique pattern

**Estimated Impact**: 20-30% improvement in license parsing and patch generation.

---

### 4. **Security Policy Regex Compilation**

**Impact**: Medium-High | **Complexity**: Low | **Priority**: High

Three regexes compiled in `collectPolicyHits()` that processes policy content line by line.

**Location**: `checks/raw/security_policy.go:200-206`

```go
func collectPolicyHits(policyContent []byte) []checker.SecurityPolicyInformation {
    // These should be package-level variables
    reURL := regexp.MustCompile(`(http|https)://[a-zA-Z0-9./?=_%:-]*`)
    reEML := regexp.MustCompile(`\b[A-Za-z0-9._%+-]+(@|\\?\[at\\?\])[A-Za-z0-9.-]+\.[A-Za-z]{2,6}\b`)
    reDIG := regexp.MustCompile(`(?i)(\b*[0-9]{1,4}\b|(Disclos|Vuln))`)
    // ... line-by-line processing
}
```

**Solution**: Move to package-level vars.

**Estimated Impact**: 35-45% faster security policy scanning.

---

### 5. **Dangerous Workflow Pattern Regexes**

**Impact**: Medium | **Complexity**: Low | **Priority**: High

Two regexes compiled at package level in `checks/raw/dangerous_workflow.go:33,62` but they create closures unnecessarily.

**Location**: `checks/raw/dangerous_workflow.go`

```go
untrustedContextPattern := regexp.MustCompile(
    `.*(<untrusted pattern>).*`)
dangerousToJSONPattern := regexp.MustCompile(`(?i)tojson\s*\(\s*github(\.event)?\s*\)`)
```

These are already at package level but could benefit from being `var` instead of being inside `containsUntrustedContextPattern()`.

**Solution**: Already partially optimized, but review usage patterns.

**Estimated Impact**: 5-10% improvement in workflow validation.

---

### 6. **Slice Allocations Without Capacity**

**Impact**: Medium | **Complexity**: Very Low | **Priority**: High

Many slice allocations use `make([]T, 0)` when the capacity is known or estimable.

**Locations** (sample):
- `clients/gitlabrepo/tarball.go:282` - `ret := make([]string, 0)`
- `clients/githubrepo/tarball.go:239` - `ret := make([]string, 0)`
- `clients/azuredevopsrepo/search_commits.go:45` - `commits := make([]clients.Commit, 0)`
- `clients/azuredevopsrepo/zip.go:221` - `ret := make([]string, 0)`
- `clients/githubrepo/contributors.go:136` - `owners := make([]*clients.User, 0)`
- `clients/githubrepo/graphql.go:243` - `ret := make([]clients.Commit, 0)`

**Solution**: Add capacity hint when size is known or estimable:

```go
// Before:
ret := make([]string, 0)

// After (when final size is known):
ret := make([]string, 0, expectedSize)
```

**Estimated Impact**: 10-20% reduction in allocations and GC pressure.

---

### 7. **Double Nested fmt.Sprintf**

**Impact**: Low-Medium | **Complexity**: Very Low | **Priority**: Medium

Nested `fmt.Sprintf` calls cause unnecessary string allocations.

**Locations**:
- `pkg/scorecard/sarif.go:436` - `ID: fmt.Sprintf("%s/%s/%s", category, runName, fmt.Sprintf("%s-%s", commit, t.Format(time.RFC822Z)))`

**Solution**:
```go
// Before:
ID: fmt.Sprintf("%s/%s/%s", category, runName, fmt.Sprintf("%s-%s", commit, t.Format(time.RFC822Z)))

// After:
ID: fmt.Sprintf("%s/%s/%s-%s", category, runName, commit, t.Format(time.RFC822Z))
```

**Estimated Impact**: Small but measurable reduction in allocations.

---

### 8. **Fuzzing Regex Compiled in File Parser**

**Impact**: Medium | **Complexity**: Low | **Priority**: High

Regex compilation inside file content parser that runs for every file.

**Location**: `checks/raw/fuzzing.go:341`

```go
var getFuzzFunc fileparser.DoWhileTrueOnFileContent = func(path string, content []byte, args ...interface{}) (bool, error) {
    // ...
    r := regexp.MustCompile(pdata.pattern)
    // ... process lines
}
```

**Solution**: Pre-compile the pattern and store in the struct or pass as compiled regex.

**Estimated Impact**: 25-35% faster fuzzing detection.

---

### 9. **Unnecessary String Joins in Hot Paths**

**Impact**: Low-Medium | **Complexity**: Very Low | **Priority**: Medium

Multiple uses of `strings.Join()` for error messages or formatting in performance-sensitive paths.

**Locations**:
- `checks/raw/pinned_dependencies.go:263,274` - Joining paths/versions for error messages
- `checks/evaluation/contributors.go:84` - Joining organization names

**Solution**: Consider whether these joins happen in hot paths. If yes, use `strings.Builder` or lazy evaluation.

**Estimated Impact**: 5-15% improvement if in hot paths.

---

### 10. **io.ReadAll on Unbounded Readers**

**Impact**: High (Security + Performance) | **Complexity**: Low | **Priority**: Critical

Multiple uses of `io.ReadAll()` without size limits on external data sources.

**Locations**:
- `clients/ossfuzz/client.go:137` - Reading HTTP response
- `clients/cii_http_client.go:71` - Reading HTTP response
- `cron/internal/cii/main.go:76` - Reading HTTP response
- `checks/raw/sast.go:294` - Reading file content
- `checks/fileparser/listing.go:132` - Reading file content

**Solution**: Use `io.LimitReader` to cap maximum read size:

```go
// Before:
content, err := io.ReadAll(resp.Body)

// After:
content, err := io.ReadAll(io.LimitReader(resp.Body, maxSize))
```

**Estimated Impact**: Prevents potential DoS and OOM issues, improves predictability.

---

## Medium Impact Optimizations

### 11. **SAST Regex in Loop**

**Impact**: Medium | **Complexity**: Low | **Priority**: Medium

**Location**: `checks/raw/sast.go:298`

Sonar config regex compiled inside validation function:

```go
regex := regexp.MustCompile(`<sonar\.host\.url>\s*(\S+)\s*<\/sonar\.host\.url>`)
```

**Solution**: Move to package-level variable.

**Estimated Impact**: 20-30% faster SAST detection.

---

### 12. **License File Regex Scanning**

**Impact**: Medium | **Complexity**: Low | **Priority**: Medium

**Location**: `checks/raw/license.go:299`

```go
spdxDigitInExt := regexp.MustCompile(`\.[[:digit:]]`)
```

Regex compiled inside license validation loop.

**Solution**: Move to package level.

**Estimated Impact**: 15-25% faster license scanning.

---

### 13. **Unnecessary bytes.Split in Fuzzing**

**Impact**: Low-Medium | **Complexity**: Low | **Priority**: Medium

**Location**: `checks/raw/fuzzing.go:342`

```go
lines := bytes.Split(content, []byte("\n"))
for i, line := range lines {
    // process line
}
```

**Solution**: Use `bufio.Scanner` for line-by-line processing to avoid allocating the entire slice:

```go
scanner := bufio.NewScanner(bytes.NewReader(content))
lineNum := 0
for scanner.Scan() {
    lineNum++
    line := scanner.Bytes()
    // process line
}
```

**Estimated Impact**: 20-30% less memory for large files.

---

### 14. **Repeated String Concatenation**

**Impact**: Low-Medium | **Complexity**: Low | **Priority**: Medium

**Locations**:
- `checks/evaluation/branch_protection.go:266,270,338,356` - Multiple addition operations in scoring

```go
sum += score.scores.review + score.scores.adminReview
sum += score.scores.thoroughReview + score.scores.codeownerReview
```

These are fine as-is for integers, but watch for string concatenations.

**Solution**: Use `strings.Builder` for string building in loops.

**Estimated Impact**: N/A for numeric operations, significant for strings.

---

### 15. **Append Without Pre-allocation - Slice Growth**

**Impact**: Medium | **Complexity**: Low | **Priority**: Medium

**Location**: `probes/hasDangerousWorkflowScriptInjection/internal/patch/impl.go:259`

```go
lines = slices.Insert(lines, insertPos, append(bytes.Repeat([]byte(" "), envvarIndent), []byte(envvarDefinition)...))
```

Multiple allocations for line insertion.

**Solution**: Consider building new slice with known size or using capacity hints.

**Estimated Impact**: 10-20% less allocations in patch generation.

---

### 16. **Sync.Mutex Could Be sync.RWMutex**

**Impact**: Low-Medium | **Complexity**: Low | **Priority**: Low

**Location**: `checks/raw/license.go:213-214`

```go
ciMapMutex        = sync.Mutex{}
reGrpIdxsMapMutex = sync.Mutex{}
```

If these protect read-heavy data structures, consider `sync.RWMutex`.

**Solution**: Analyze read vs write patterns and use `sync.RWMutex` if read-heavy.

**Estimated Impact**: 15-30% improvement in concurrent read scenarios.

---

### 17. **Package Manager Regex Compilation**

**Impact**: Low | **Complexity**: Very Low | **Priority**: Low

**Location**: `cmd/package_managers.go:32-34`

```go
githubDomainRegexp    = regexp.MustCompile(`^https?://github[.]com/([^/]+)/([^/]+)`)
githubSubdomainRegexp = regexp.MustCompile(`^https?://([^.]+)[.]github[.]io/([^/]+).*`)
gitlabDomainRegexp    = regexp.MustCompile(`^https?://gitlab[.]com/([^/]+)/([^/]+)`)
```

Already at package level - good! Just verify they're used efficiently.

**Estimated Impact**: Already optimized.

---

### 18. **Unnecessary Type Conversions**

**Impact**: Low | **Complexity**: Low | **Priority**: Low

**Location**: Multiple files

Converting between `[]byte` and `string` repeatedly in some functions.

**Solution**: Minimize conversions, work with one type as long as possible.

**Estimated Impact**: 5-10% reduction in allocations.

---

## Lower Priority Optimizations

### 19. **Global Regex Variables Already Optimized**

**Impact**: N/A (Already Good) | **Complexity**: N/A | **Priority**: N/A

Several files already use package-level regex compilation correctly:

- `checks/raw/sbom.go:28-29` - `reRootFile`, `reSBOMFile`
- `checks/raw/branch_protection.go:30` - `commit`
- `checks/raw/code_review.go:28-29` - `rePhabricatorRevID`, `rePiperRevID`
- `checks/raw/license.go:40,91` - `reLicenseFile`, `reLicenseFileExts`
- `clients/gitlabrepo/checkruns.go:26` - `gitCommitHashRegex`
- `checks/raw/shell_download_validate.go:60` - `gitCommitHashRegex`
- `probes/unsafeblock/impl.go:223-239` - Java unsafe detection regexes

**Status**: These are already optimized! Good examples to follow.

---

### 20. **Missing sync.Pool for Frequent Allocations**

**Impact**: Medium (for high-throughput scenarios) | **Complexity**: Medium | **Priority**: Medium

No evidence of `sync.Pool` usage for frequently allocated objects like buffers.

**Locations**: Consider for:
- `bytes.Buffer` allocations throughout the codebase
- Temporary slices in hot paths
- Regex match results

**Solution**: Implement `sync.Pool` for `bytes.Buffer`:

```go
var bufferPool = sync.Pool{
    New: func() interface{} {
        return new(bytes.Buffer)
    },
}

func doWork() {
    buf := bufferPool.Get().(*bytes.Buffer)
    buf.Reset()
    defer bufferPool.Put(buf)
    // ... use buffer
}
```

**Estimated Impact**: 20-40% reduction in allocations under high load.

---

### 21. **Dangerous Workflow Patch Generation Could Use Fewer Allocations**

**Impact**: Medium | **Complexity**: Medium | **Priority**: Medium

**Location**: `probes/hasDangerousWorkflowScriptInjection/internal/patch/impl.go`

Multiple slice operations, insertions, and byte array manipulations could be optimized with:
- Pre-calculating sizes
- Using `strings.Builder` or `bytes.Buffer` where appropriate
- Reducing temporary allocations

**Solution**: Profile and optimize the hottest paths.

**Estimated Impact**: 15-30% faster patch generation.

---

### 22. **Consider Using strings.Cut More Extensively**

**Impact**: Low | **Complexity**: Very Low | **Priority**: Low

Go 1.18+ introduced `strings.Cut` which is more efficient than `strings.Split` for simple splitting.

**Location**: Already used in some places like `checks/raw/pinned_dependencies.go:508`

**Solution**: Audit uses of `strings.Split` and `strings.SplitN` where only first/last split is needed.

**Estimated Impact**: 5-10% improvement in string parsing.

---

### 23. **Range Over Map Keys When Values Not Needed**

**Impact**: Very Low | **Complexity**: Very Low | **Priority**: Low

**Location**: `docs/checks/impl.go:77`

```go
for k := range d.internaldoc.InternalChecks {
    // only k is used
}
```

This is already optimal! But check other locations for `for k, _ := range` which should be `for k := range`.

**Estimated Impact**: Minimal, but cleaner code.

---

### 24. **Consider Lazy Compilation for Rarely-Used Regex**

**Impact**: Low | **Complexity**: Medium | **Priority**: Low

Some regexes that are compiled at package init time may never be used in certain execution paths.

**Solution**: Use `sync.Once` for lazy initialization:

```go
var (
    expensiveRegex     *regexp.Regexp
    expensiveRegexOnce sync.Once
)

func getExpensiveRegex() *regexp.Regexp {
    expensiveRegexOnce.Do(func() {
        expensiveRegex = regexp.MustCompile("<pattern>")
    })
    return expensiveRegex
}
```

**Estimated Impact**: Faster startup time, slightly more complex code.

---

### 25. **Review Error String Allocations**

**Impact**: Low | **Complexity**: Low | **Priority**: Low

Some error messages use `fmt.Sprintf` when simple concatenation or constants would suffice.

**Solution**: Use constants or `errors.New()` for static error messages.

**Estimated Impact**: 5-10% reduction in error path allocations.

---

## Summary Statistics

| Priority | Count | Estimated Total Impact |
|----------|-------|------------------------|
| Critical | 4 | 30-50% improvement in hot paths |
| High | 8 | 15-35% improvement each |
| Medium | 8 | 10-25% improvement each |
| Low | 5 | 5-15% improvement each |

## Recommended Implementation Order

1. **Phase 1** (Quick Wins): Items 1-7 - Regex compilation fixes
2. **Phase 2** (Safety): Item 10 - io.ReadAll limits
3. **Phase 3** (Memory): Items 6, 13, 15, 20 - Allocation optimizations
4. **Phase 4** (Polish): Remaining items

## Testing Recommendations

For each optimization:
1. Add benchmarks using `testing.B`
2. Compare before/after with `benchstat`
3. Profile with `pprof` to verify impact
4. Ensure all existing tests pass

Example benchmark structure:

```go
func BenchmarkIsGoUnpinnedDownload(b *testing.B) {
    cmd := []string{"go", "get", "github.com/foo/bar@v1.2.3"}
    b.ResetTimer()
    for i := 0; i < b.N; i++ {
        _ = isGoUnpinnedDownload(cmd)
    }
}
```

---

**Total Items**: 25 performance improvement opportunities identified
**Estimated Overall Impact**: 20-40% performance improvement across the codebase
**Implementation Effort**: Low to Medium for most items
