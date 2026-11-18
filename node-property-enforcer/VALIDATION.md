# Validation record — 18 November 2026

Status: **development candidate; unchanged strict SonarQube quality gate PASSED, coverage 100%.** This supersedes the earlier 68.4% report. Coverage measures executed code, not proof of defect-free software or complete OWASP compliance.

## Scope and reproducibility

Only Node Property Enforcer source, test tooling, dependencies and plugin endpoints were assessed. Jenkins 2.555.1 / Java 21.0.13 were unchanged disposable test hosts. No Jenkins core patch, controller-wide vulnerability scan or organizational Jenkins access was performed. Existing local Maven, JDK, SonarQube and Gitleaks installations were reused.

The final full run contains **69 reported Java tests: 67 passed, two intentional skips, zero failures/errors** (`message-audit-cycle3-verify.log`). It passes `verify -Penable-jacoco`, including all coverage counters, SpotBugs and formatting. The skips are the opt-in manual browser fixture and one generated Jenkins harness check. The delivered JPI is this verified artifact.

Run `npm ci --ignore-scripts` followed by `npm run coverage` for the JavaScript suite. The final run contains **28 passing tests**, recorded in `message-audit-cycle3-js.log`. Test-only npm dependencies are not packaged in the JPI.

## Why coverage was 68.4%, and what changed

The earlier combined Sonar measure included browser scripts without an imported JavaScript coverage report and untested Java failure/lifecycle branches. Tests now execute those paths rather than suppressing or removing them. They exercise malformed inputs, failed writes, missing providers, permission boundaries, non-build executables, lifecycle cleanup, aborted/failed requests, bounded queues, nested restrictions and concurrent history updates.

| Measure | Final result |
| --- | --- |
| Sonar combined / line / branch coverage | 100.0% / 100.0% / 100.0% |
| Java lines / branches | 700/700 and 438/438 |
| Java instructions / methods / classes | 3,411/3,411; 207/207; 26/26 |
| JavaScript lines / branches | 377/377 and 340/340 |
| JavaScript statements / functions | 462/462 and 88/88 |
| Sonar bugs / vulnerabilities / code smells / hotspots | 0 / 0 / 0 / 0 |
| Sonar duplicated lines | 0.0% |
| SpotBugs | Zero findings and analysis errors |
| Job Restrictions 242.v6edda_c9e4ca_f | 16 restriction-content and queue tests passed against the final source |
| npm dependency audit | Current plugin test-tooling audit: zero reported vulnerabilities |
| Gitleaks source scan | Zero leaks in the complete staged source package |

No production code was excluded from reporting, no coverage counters were manufactured, and no rules or thresholds were relaxed. The inherited JaCoCo report exclusions were explicitly cleared. Instrumentation is limited to this plugin's package so it does not instrument Jenkins core. Reports inspect every compiled project class. All four production JavaScript files are instrumented; new source files are seeded with zero coverage before tests. JaCoCo requires 100% line, branch, instruction, method and class coverage; the JavaScript command requires 100% line, branch, statement and function coverage. Sonar imports both actual reports.

`target/message-audit-20261118/rules-comparison.json` verifies that strict gate conditions and assigned profile keys, rule counts and rule-update timestamps match the snapshots taken before analysis. Existing narrowly documented suppressions were not broadened. Historical resolved Sonar issues remain in the API export; the current scan has no open issues. The final unchanged gate passed (`message-audit-cycle3-sonar.log`).

## Jenkins resource-health checks

The review identified and fixed practical resource risks:

* Restriction traversal has a whole-tree budget of 1,024 restriction/row visits, in addition to the existing depth 32 and list size 100 limits. Tests cover a shared graph representing over a billion naive traversal paths, a cycle, and oversized nested submissions.
* Each build stores at most 256 distinct node names, with immutable snapshots and deduplication. History writes for one build do not hold a lock shared by another build. Concurrent updates, slow storage and failed writes are tested. No plugin-owned thread pool or background worker is created.
* Browser validation runs at most four requests concurrently, cancels superseded requests, bounds pending work, uses weakly held input caches and has ten-second request timeouts. Page exit cancels pending work, clears timers and disconnects the observer. Browser history restoration resumes validation.
* Warning polling allows one request at a time, pauses on hidden pages and stops for detached UI or page exit. Timeout, failure and restoration paths are tested.

The final sample of **2,000 enforcement queue checks** measured **40.211 ms wall time, 31.250 ms thread CPU and 10,219,768 bytes of temporary allocation**. The test requires completion within ten seconds and allocation below 128 MiB. Concurrent-history tests completed without detected deadlocks; the tests also check the history cap and absence of newly created plugin threads.

The new read-only queue diagnostic bounds requests to eight IDs, matching nodes to 32 and name-rule traversal to 128 visits/depth 32/list size 100. Unknown restriction classes and subclasses are not evaluated. Tests cover these bounds, provider matching, anonymous access, job visibility, hidden pattern details, offline/eligible node fallback, browser request timeouts, widget replacement and navigation cleanup. These limits do not impose a runtime timeout on the provider's Java regular-expression matching.

These are bounded regression checks on this laptop, not a heap-retention soak test, production capacity benchmark or proof against every OOM, CPU spike, JVM failure or thread issue. Queue cost still depends on policy size, node count, request concurrency and external provider behavior. Arbitrary Java regular expressions are checked for syntax/completeness, not runtime complexity. A syntactically valid expensive expression can still burden the Job Restrictions provider when it evaluates jobs. Validate intended policies and soak-test representative workloads before production rollout.

## Functional behavior retained

Required Job Restrictions reject the built-in No restriction choice at every nested level. And/Or require both branches; Not requires its child; multi-entry lists need complete entries. User/group IDs and class rows cannot be blank. Expressions must be nonblank and valid Java syntax. Registered third-party restriction types retain their own validation; completeness checks do not prove policy effectiveness.

Required Matrix Auth needs explicit node permissions. Authenticated creators receive enabled node-scoped permissions on new/copied nodes only; existing edits do not infer or restore creator permissions. Automated tests cover local, email-style SSO and domain-qualified LDAP-style IDs through Jenkins APIs, without contacting external identity services.

Native Save/Apply/Enter checks keep incomplete forms open with errors beside the affected fields. Direct submissions have server-side enforcement. Existing agents cannot receive new queued work when build enforcement is enabled; running builds continue. Shared labels and compound label expressions are covered with two real online agents. The label fixture now saves label changes through Jenkins.updateNode before scheduling, so Jenkins' assigned-label cache represents the saved configuration.

## Earlier browser review

The earlier 17 November Chrome review used a disposable JenkinsRule preview with two online agents. Its first short-lived preview expired during review; that unfinished build is not counted as successful completion. A replacement preview extended only the manual fixture lifetime to 30 minutes to allow the requested 60-second inspection pauses. Production code and automated test thresholds were unchanged.

That earlier review confirmed field-local errors for missing properties and nested No restriction choices, same-page Save/Apply blocking, warning-only running/completed console messages, a shared-label build selecting the compliant agent, invalid-expression blocking, and a corrected expression persisting after Apply/reload. A pending build started when its assigned node was corrected, and saving the node while that build was running kept it online. The resumed build completed SUCCESS with no setup warning. These observations supplement the automated JavaScript/HtmlUnit checks; they are not a claim that every possible browser, node type or provider combination has been tested.

That preview review is historical. On 18 November, the updated JPI was installed through Jenkins' plugin upload endpoint and the user-authorized localhost:19085 controller was restarted with the same home, core WAR and startup arguments. All three requested existing agents reconnected. SHA-256 comparison confirms the installed JPI is identical to the delivered artifact. The pre-install and post-restart comparisons found all 29 existing node, job, user and controller configuration files unchanged. `queue.xml` is live queue state, not a configuration file, and changes when test builds are queued.

The earlier queue-message walkthrough and evidence were recorded under `target/queue-message-review/` and packaged in `dist/node-property-enforcer-validation.zip`. Queue explanations stay beside pending builds, identify the node/job/pattern, and link directly to that node's configuration. A blocked label pool lists separate reasons for its nodes. `test #3` finished SUCCESS on `Restricted`; `approved-npe-review-20261117-label-pool #2` and the compound-label job's build #2 finished SUCCESS on `restriction-validation-164439`. Running and completed consoles showed no false setup warning. Save on `noRestriction` stayed on the original form, focused its missing option, and displayed red field-local messages. The updated setting's help clearly explains checked and unchecked behavior.

Review also found that the dashboard's native hover text retained the incomplete reason. The final frontend correction keeps that plain-text tooltip consistent with the visible explanation, restores the original tooltip on fallback, and never enables HTML rendering. It was verified in Chrome after installing the final JPI; the JavaScript suite and unchanged Sonar gate were rerun successfully. The new queue explanation is a classic UI enhancement: Jenkins' native REST `why` is deliberately unchanged, and unsupported cases retain its native explanation. The previous initial wait for asynchronous queue widgets was corrected in the current review: new visible rows now request their first result promptly while the 30-second background refresh setting remains unchanged. Important views were held for at least 60 seconds.

That earlier configuration comparison showed **29 unchanged files**. Only that review's queued items 9, 10 and 11 were cancelled; completed build history remains available. `noRestriction`, `Restricted` and `restriction-validation-164439` are online, and Chrome was left on Nodes. No existing node, job or authorization settings were edited during the walkthrough.

## Current Chrome message audit

The 18 November review uses the final installed artifact, SHA-256 `0fd2289f8782d02a67d53f57b7a5b2106db840f4303b074c024a09ff9feacd41`. Detailed cases and screenshots are in the validation archive's `chrome-review` folder. Important views were left visible for at least 60 seconds. Chrome interaction initially failed because another application covered its window; restoring Chrome visibility resolved that obstruction.

The review found and corrected three plugin issues before continuing: queue rendering could overwrite the Cancel tooltip, optional-provider linkage could prevent queue guidance from loading without Job Restrictions, and an oversized class name showed a syntax message instead of its 1024-character limit. Each correction was built, installed in the review environment and retested. The final full verification and unchanged Sonar gate passed after those corrections.

The recorded checks include:

* Both missing properties, security alone missing, and restriction alone missing: dashboard, job-history and node messages name exactly the incomplete settings for the relevant node. Valid properties and unrelated nodes are not added to those explanations.
* Same-page Save/Apply/Enter blocking, empty permission matrices, every built-in restriction choice, nested unrestricted children, empty lists, blank user/group/class rows, invalid expression syntax and length boundaries. Corrected input clears its field-local error. The class-limit correction was specifically retested after deployment.
* No optional providers installed: the plugin and queue component load. Selected unavailable requirements have local red provider/version guidance. Empty rows, available core selections and disabled requirements have no unrelated error.
* All behavior help links; independent warning and existing-edit switches; strict enforcement restored after each optional-mode exercise. With enforcement on, three new builds remain queued for one or both missing requirements. Hiding warnings does not permit those builds to run.
* New-node creation adds the authenticated creator's five Agent permissions, blocks default No restriction, and retains the permissions after a valid save and reload. Existing-node edits do not automatically grant permissions.
* A deliberately warning-only preview build continued when enforcement was enabled during execution and finished SUCCESS. Its running and completed console explanations changed appropriately, listed only its actual node, and left the log/result unchanged. A later build stayed queued until both required properties were completed; saving the complete node released it, and it finished SUCCESS. Current warnings on the earlier completed console then cleared.

All setting changes above were confined to disposable preview controllers. The existing localhost:19085 node, job, user and controller configurations were compared against the 29-file baseline. A finite scenario checklist and 100% measured code coverage do not establish exhaustive UI coverage across every browser, provider, node type or Jenkins version.

The final deployed-controller reruns produced these results without editing their configurations:

| Review job | Build | Result / selected node |
| --- | --- | --- |
| `test` | 4 | SUCCESS on `Restricted` |
| `approved-npe-review-20261117-label-pool` | 3 | SUCCESS on `restriction-validation-164439` |
| `approved-npe-review-20261117-label-compound` | 3 | SUCCESS on `restriction-validation-164439` |
| `approved-npe-review-20261117-direct` | 2 | SUCCESS on `restriction-validation-164439` |
| `npe-review-20261117-noRestriction` | Original queue item 15 | Still waiting; both required properties missing |
| Restricted-name and blocked-label review jobs | Queue items 17 and 16 | Stayed queued with the correct node-specific explanations; cancelled only these newly created review entries after validation |

The running and completed direct/shared-label consoles had no false warning about unused nodes. All three requested nodes remained online. The final comparison again found zero changes among the 29 existing configuration files; the original queued build was retained. Source, installed JPI and test-stage production resources were compared, and the installed and delivered JPI hashes match.

## Scan limitations and release status

The fresh OWASP Dependency-Check update **failed** because no usable NVD API key/data was available (`coverage-dependencies.log`). This is not a passing current dependency scan. The prior cached scan dated 10 November is historical and is not presented as fresh evidence. The existing expiring matrix-auth/matrix-project false-positive suppression was not changed. Xray has no configured service credentials. No new broad DAST result is claimed; direct plugin HTTP tests cover permissions, POST and CSRF boundaries.

This is an evaluation build, not an approved Jenkins community release or a guarantee of future LTS/weekly compatibility. Jenkins hosting metadata/maintainer setup, current dependency vulnerability data and representative production soak testing remain release tasks. The package includes source, test reports, coverage reports, scanner evidence, resource measurements and SHA-256 checksums.



