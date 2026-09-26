# Chapter 88 — Stable, LTS, Backporting, and Product Kernels

> Mainline is where development happens. **Almost nobody runs mainline.** Your phone runs a
> five-year-old LTS with 100,000 lines of vendor delta; your server runs RHEL's 5.14 with a
> decade of backports; your embedded product runs whatever the SoC vendor shipped. This
> chapter is about that world — the one where kernel engineering actually meets a customer —
> and it is disproportionately what an embedded or platform architect is interviewed on.

---

## Theory & First Principles

### T.0 — Start here: your distro runs 5.15, and the fix landed in 6.12

```bash
uname -r        # 5.15.0-91-generic   -- "5.15" was released in 2021
```

The bug you are hitting was fixed in mainline 6.12. Your distro is not going to ship you a
new kernel: **the entire value of a long-term release is that it does not change.** And yet
the fix must reach you. **How?**

```
   mainline  ---o---o---o---o---o---o---o---o---o--->   ~15,000 commits / 70 days
                 \         \                   |
                  \         \                  | someone tags a commit:
                   \         \                 |   Cc: stable@vger.kernel.org
                    \         \                |   Fixes: <sha> (subject)
                     \         \               v
   6.12.y (LTS)       o---o---o---o---o--->   ONLY fixes, cherry-picked
   6.6.y  (LTS)       o---o---o--->
   5.15.y (LTS)       o---o--->
                      ^
                      each of these is maintained for 2-6 YEARS
```

**The rule that defines the whole system, and it is stricter than people expect:**

> **A stable tree accepts *fixes for real bugs that users hit*, and nothing else.** No
> features. No cleanups. No refactoring. No "this is obviously better." And critically: the
> fix must **already be in mainline** — stable never gets a patch first.

**That last clause is the load-bearing one**, and it generalizes to any maintenance branch you
will ever run:

> **Fix forward, then backport. Never the reverse.** If stable could take a fix that mainline
> does not have, then upgrading would *regress* — you would move to a newer kernel and lose
> your fix. The mainline-first rule is what guarantees that upgrading is always monotonic.

This is the same argument as "never fork your dependency" and as the rule that schema
migrations must be forward-only. **Divergence between a maintenance branch and the trunk is a
debt that compounds**, and mainline-first is the only discipline that prevents it.

**The constraints on a stable patch are mechanical enough to memorize** — they are what you
will be judged against:

| Requirement | Why |
|---|---|
| Already in Linus's tree | the monotonicity rule above |
| Fixes a **real** bug that users hit | not theoretical, not a code smell |
| Obviously correct, and tested | nobody wants to debug a stable tree |
| Small — ideally under ~100 lines | review cost, and risk |
| No new features, no ABI change | an LTS user must be able to upgrade blindly |
| Does not depend on unbackported changes | or you must backport those too, growing the risk |

**And now the genuinely hard case, which is the real content:** a fix that depends on a
refactor which is not in the old tree. You have three options, each bad in a different way —
backport the refactor (large, risky, changes behaviour in an LTS), write a *different* fix for
the old tree (divergence: the stable code now differs from mainline, and every future backport
to that area conflicts), or do not fix it (leave users exposed). **There is no good answer,
and recognizing that there is no good answer is the mature position.**

**Two operational facts worth carrying:**

1. **Most backporting is now automated.** The `Fixes:` tag, plus AUTOSEL (a machine-learning
   selector the stable maintainers run), picks up thousands of commits that nobody explicitly
   tagged. **Which is precisely why Ch. 86 §T.0 insists on the `Fixes:` tag** — it is not
   bookkeeping, it is the input to an automated pipeline.
2. **"Stable" is not the same as "secure."** Many security fixes are backported without ever
   being identified as security fixes, on the explicit view that any bug may be a security
   bug. This is a real and contested policy position — and it is why "we only apply CVE
   patches" is an ineffective patching strategy for Linux specifically.

```bash
git log --grep='Cc: stable' --oneline -20
git log --grep='^Fixes:' --oneline -20
git tag -l 'v6.6.*' | tail -5
cat Documentation/process/stable-kernel-rules.rst
```

---

### T.1 — The tree topology

```
                    mainline (Linus)
                    v6.N released every ~9-10 weeks
                          │
          ┌───────────────┼──────────────────────┐
          │               │                      │
    stable v6.N.y     stable v6.(N-1).y      LTS v6.M.y
    ~3 months          ~3 months             2-6 years
    (until N+1)        (until N+1)                │
                                                  │
                          ┌───────────────────────┼─────────────┐
                          │                       │             │
                 Android Common Kernel      Distro kernels   Vendor BSP
                 (ACK, per-LTS)             (RHEL/SUSE/      (SoC vendor,
                          │                  Debian/Ubuntu)   + huge delta)
                          │                                       │
                 OEM/device kernel                         product kernel
```

| Tree | Lifetime | Who runs it | Delta from mainline |
|---|---|---|---|
| **mainline** | 9–10 weeks | kernel developers, the brave | 0 |
| **stable** (`v6.N.y`) | until `v6.(N+1)` | some distros, some users | fixes only |
| **LTS** (`v6.M.y`) | 2 years (extendable to 6) | Android, embedded, enterprise base | fixes only |
| **Distro** (RHEL, SUSE) | 10+ years | enterprise | **enormous** — thousands of backports |
| **Android Common** | per-LTS | every Android device | large, but upstreamed aggressively |
| **Vendor BSP** | per-SoC | embedded products | often 1–3 million lines |

**The number to internalize:** a typical Android device shipped in the 2010s ran a kernel
with **1–3 million lines** of out-of-tree code. Google's Generic Kernel Image (GKI) effort
exists specifically to attack that number, and it has substantially succeeded — which is one
of the best real-world demonstrations of the cost of an out-of-tree delta.

### T.2 — The stable rules

`Documentation/process/stable-kernel-rules.rst`, which is short and should be read verbatim.
The criteria for a stable backport:

- **It must fix a real bug that bothers people.** Not a theoretical one.
- **It must already be in Linus's tree.** Stable never gets a patch mainline does not have.
  This is the *upstream-first* principle and it is absolute.
- **It must be obviously correct and tested.**
- **No bigger than ~100 lines**, with context.
- **It must fix one thing.** No "while we're here" cleanups.
- **No new features. No new APIs. No refactoring.**
- It may be a build fix, a security fix, an oops/hang/data-corruption/real-security-issue
  fix, or a "real bug that bothers people" fix.

The two mechanisms for requesting one:

```
 # In the commit message, before Signed-off-by:
 Cc: stable@vger.kernel.org
 Cc: stable@vger.kernel.org # v6.1+
 Cc: stable@vger.kernel.org # 6.1.x: abcdef123456: prerequisite patch subject
```

```
 # After it is merged to mainline, email stable@vger.kernel.org with the
 # mainline SHA and which trees it should go to.
```

**And the automatic mechanism, which now dominates:** the stable team runs an AI/heuristic
classifier over every mainline commit looking for fixes that were not tagged. A `Fixes:` tag
massively increases the chance of being picked up. **This is why `Fixes:` is the single
highest-leverage line in a commit message** (Ch. 86 §T.3).

### T.3 — Upstream-first, and why it is not negotiable

> **Every fix goes to mainline first. Always. Without exception.**

If a patch goes into a product kernel but not mainline:
1. The next kernel you rebase onto does not have it. You carry it forever.
2. Nobody else benefits, and nobody else reviews it.
3. It never gets tested by the community's infrastructure.
4. When the code it touches changes upstream, your patch conflicts and someone has to
   re-derive its intent from a diff.

The compounding cost is the point. A vendor with a 2-million-line delta is paying a rebase
tax on every one of those lines, every kernel uprev, forever. The GKI programme's entire
value proposition is converting that recurring cost into a one-time upstreaming cost.

**The sequence for a product fix under time pressure:**

```
 1. Fix it locally so the product ships.          (day 0)
 2. IMMEDIATELY prepare the mainline version.     (day 0-2)
 3. Submit upstream.                              (day 2)
 4. When it lands, tag it Cc: stable.             (week 2-6)
 5. When it appears in the LTS, DROP your local   (week 6-10)
    patch and take the upstream one.
```

Step 5 is the one organizations skip, and skipping it is how a 200-patch delta becomes a
2,000-patch delta. **Make dropping local patches an explicit, tracked activity.**

### T.4 — Backporting mechanics

```bash
# The happy path: it cherry-picks cleanly.
git checkout linux-6.6.y
git cherry-pick -x <mainline-sha>
#   -x adds "(cherry picked from commit <sha>)", which is REQUIRED for
#   stable and is how everyone traces provenance.
```

When it conflicts, which is most of the time on an older tree:

```bash
git cherry-pick -x <sha>
# CONFLICT

# 1. Understand WHAT the patch does. Read the mainline commit message
#    and the surrounding code in BOTH trees.
git show <sha>
git log --oneline -20 <sha> -- <file>       # what else changed nearby?

# 2. Are there prerequisites? Usually yes.
git log --oneline v6.6..<sha> -- <file>
#    If the fix depends on a refactor, you have three options:
#      a) backport the prerequisite too (if it is safe)
#      b) adapt the fix to the old code structure
#      c) decline

# 3. Resolve, preserving INTENT, not text.
git mergetool
git cherry-pick --continue

# 4. DOCUMENT the adaptation. This is mandatory.
git commit --amend
```

The commit message for an adapted backport:

```
foo: fix use-after-free in the teardown path

commit 0123456789abcdef0123456789abcdef01234567 upstream.

[ Upstream used the new foo_put() helper introduced in commit
  fedcba987654 ("foo: add refcounting helpers"), which is not present
  in 6.6. Open-coded the equivalent refcount_dec_and_test() +
  kfree() sequence instead. No functional difference. ]

<original commit message>

Signed-off-by: Original Author <author@example.com>
Signed-off-by: Backporter <you@example.com>
```

The `[ ... ]` note is not optional. Six months later, when someone bisects a problem to your
backport, that note is the only thing that tells them what you changed and why.

### T.5 — When not to backport

Equally important, and a genuine judgement call.

**Do not backport when:**
- The fix requires a large prerequisite series. The prerequisites carry more risk than the
  bug.
- The code has been substantially rewritten upstream. Your adaptation is effectively new,
  unreviewed code.
- The bug is not reachable in your configuration.
- The "fix" is really a design change.
- You cannot test it on the affected hardware.

**The asymmetry to hold onto:** a regression in a stable kernel is worse than the bug it was
meant to fix, because users trust stable kernels not to change behaviour. The stable team's
own regression rate from backports is a recurring topic; they take it seriously and so should
you.

When you decline, **record the decision** — bug ID, CVE, why not, and what the mitigation is.
"We chose not to backport CVE-X because the vulnerable code path requires
`CONFIG_FOO` which we do not enable" is a defensible answer to an auditor; "we did not notice"
is not.

### T.6 — CVEs, and the 2024 change

In February 2024 the kernel became a **CVE Numbering Authority (CNA)**, and the effect was
dramatic and deliberate.

**The old world:** third parties assigned kernel CVEs sporadically, often years late, often
for bugs that were not exploitable, often missing the ones that were. The CVE list was close
to useless as a security signal.

**The new world:** the kernel CNA assigns a CVE to **any commit that fixes a
potentially-security-relevant bug** — which, given that almost any kernel bug is potentially
security-relevant, means **thousands of CVEs per year**.

The stated rationale, which is worth being able to reproduce because it is a genuinely
interesting security-architecture position:

> Any bug can be a security bug. There is no reliable way to know in advance. Therefore the
> only defensible security posture is **to take all the stable fixes**, and the CVE list's
> job is to document what a given stable release fixed — not to give you a list to
> cherry-pick from.

The consequences for anyone shipping a product:

1. **"Fix the CVEs" is no longer a viable strategy.** There are too many, and the list is
   explicitly not a prioritized one.
2. **The supported strategy is: take the whole stable tree, continuously.** Rebase onto
   `v6.M.y` regularly; do not pick.
3. **A CVE is assigned only where a fix exists in a stable tree.** Running an unsupported
   kernel means you have no CVE data at all — which is worse, not better.
4. Compliance frameworks that require CVE-by-CVE triage are now in direct tension with the
   kernel's guidance, and resolving that tension is a real architect problem.

```bash
# The CVE data, machine-readable:
git clone https://git.kernel.org/pub/scm/linux/security/vulns.git
cd vulns
ls cve/published/6.6/
# Each CVE maps to: the introducing commit, the fixing commit, and the
# affected version range.

# Which CVEs does my kernel version have unfixed?
./scripts/cvetool ...    # or the published JSON
```

### T.7 — Distribution kernels

RHEL's "5.14" kernel is not 5.14. It contains thousands of backports from 6.x, features
backported wholesale, and a stable kABI — and understanding why is useful.

| Property | Why |
|---|---|
| **10+ year support** | Enterprise purchasing cycles |
| **Frozen version number** | A marketing and certification artifact, not a technical one |
| **kABI stability** | Third-party modules (storage, network, security vendors) must keep loading across updates |
| **Massive backport volume** | Features must be added without changing the version |
| **Certification** | Hardware and software vendors certify against a specific kernel |

**kABI** deserves attention because it is a genuinely interesting engineering problem.
Upstream Linux deliberately has **no** stable in-kernel ABI
(`Documentation/process/stable-api-nonsense.rst`). Distributions synthesize one:

- A **whitelist** of symbols whose signature must not change.
- **Symbol checksums** (`genksyms`/`modversions`) to detect accidental changes.
- **Padding fields** in structures, reserved for future expansion without changing size.
- **Compatibility shims** where a change was unavoidable.

The cost is that the distribution cannot take upstream changes that alter those interfaces,
which means diverging further, which increases the backport cost — a ratchet. Being able to
explain that ratchet is a good answer to "why does enterprise Linux cost money?"

### T.8 — Android, GKI, and the modularization story

The best-documented case study in out-of-tree delta management, and one you should know.

**The problem, circa 2018:** every Android device ran a unique kernel — an LTS base plus the
SoC vendor's BSP plus the OEM's changes. 1–3 million lines of delta. Consequences: devices
could not receive kernel security updates (the delta made rebasing impossible), fragmentation
was total, and the ecosystem's median kernel age was measured in years.

**GKI (Generic Kernel Image), from Android 11:**

```
 ┌──────────────────────────────────────────────────┐
 │  Vendor modules (.ko)   -- SoC and OEM specific   │
 ├──────────────────────────────────────────────────┤
 │  KMI: a STABLE module interface, per Android      │
 │       release. Symbol list + ABI checking (STG)   │
 ├──────────────────────────────────────────────────┤
 │  GKI: ONE kernel binary, built by Google from     │
 │       the Android Common Kernel, for all devices  │
 └──────────────────────────────────────────────────┘
```

The mechanisms:
- **Everything vendor-specific becomes a module.** Nothing SoC-specific in the core image.
- **A stable KMI** within an Android release, enforced by ABI-comparison tooling (`libabigail`,
  then `STG`) in CI. Breaking it fails the build.
- **Aggressive upstreaming**, because the only way to shrink the delta permanently is to
  eliminate it.
- **Google ships the GKI binary**, so a security update does not require the SoC vendor.

**The results**, which are the interesting part: the ACK delta over LTS dropped very
substantially, devices became able to take kernel updates independently of the SoC vendor,
and the median shipped-kernel age fell.

**The transferable lessons**, which apply to any embedded product:
1. **Modularize aggressively.** Every vendor-specific thing that is a module is a thing that
   does not block a kernel update.
2. **Define and enforce an interface.** Even a locally-stable one. Enforce it in CI.
3. **Upstream continuously.** The delta is a recurring cost; upstreaming converts it to a
   one-time cost.
4. **Decouple the update path from the vendor.** If your security update requires a
   third party to act, you do not control your security posture.

### T.9 — Choosing a kernel for a product

The architect decision, with the real criteria.

| Criterion | Question |
|---|---|
| **Support lifetime** | Does the LTS outlive the product? Check `kernel.org/category/releases.html`; LTS is 2 years by default, extended to 6 only when someone is testing it |
| **Vendor support** | Does the SoC vendor's BSP target this version? In practice this usually decides it |
| **Feature requirements** | What do you need that is not in the older LTS? Can it be backported? |
| **Delta size** | How much out-of-tree code, and what is the rebase cost? |
| **Certification** | Do you need a certified kernel? (automotive, medical, avionics) |
| **Security posture** | Can you take stable updates continuously? If not, why not, and can that be fixed? |

**The decision rule, in one sentence:** *pick the newest LTS your SoC vendor supports, budget
explicitly for continuous stable updates, and treat every out-of-tree patch as a recurring
liability with a named owner and an upstreaming plan.*

The failure mode to name in an interview: **picking an old LTS because the vendor BSP targets
it, then discovering at end-of-life that you cannot move**, because the delta has grown to
the point where rebasing is a multi-quarter project. That is the single most common embedded
Linux failure, and the fix is entirely a matter of discipline applied from day one.

### T.10 — Managing a delta

If you must carry out-of-tree patches — and you must — manage them as a first-class artifact.

**The discipline:**

1. **Every patch has an owner and a category.**

   | Category | Plan |
   |---|---|
   | Upstreamable | submit it, now; drop when it lands |
   | Not-yet-upstreamable | what is blocking? who is unblocking it? |
   | Never-upstreamable (product-specific) | document why; minimize it |
   | Backport from newer upstream | drop at the next uprev |

2. **Keep it as a patch series, not a fork.** `git rebase` onto each stable release, not
   `git merge`. The series is reviewable; a merged fork is not.

3. **Automate the rebase.** Every stable release, rebase and run CI. If you do it weekly it
   is an hour; if you do it yearly it is a quarter.

4. **Track the number and report it.** Patch count and line count, on a chart, in front of
   management. It is the only way the upstreaming budget survives contact with a deadline.

5. **Require justification for additions.** A new out-of-tree patch should need the same
   approval as a new dependency.

Tools that exist for this: `quilt` (the classic), `git-series`, `b4`, Yocto's
`devtool`/`kernel-yocto` (→ Ch. 98), and the Android `repo`/ACK workflow.

---

## 1. Internals

### Documentation

| Path | Contents |
|---|---|
| `Documentation/process/stable-kernel-rules.rst` | **The rules.** Short; read it verbatim |
| `Documentation/process/stable-api-nonsense.rst` | Why there is no in-kernel ABI |
| `Documentation/process/security-bugs.rst` | How to report a security bug |
| `Documentation/process/cve.rst` | The CNA process and its rationale |
| `Documentation/process/handling-regressions.rst` | |
| `Documentation/admin-guide/reporting-issues.rst` | What to ask a reporter for |
| `Documentation/process/backporting.rst` | Backporting guidance |

### The trees

```bash
# stable and LTS
git clone https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux-stable.git
git branch -a | grep linux-6                # all the stable branches

# the stable queues -- what is PENDING for the next stable release
git clone https://git.kernel.org/pub/scm/linux/kernel/git/stable/stable-queue.git
ls stable-queue/queue-6.6/

# the CVE database
git clone https://git.kernel.org/pub/scm/linux/security/vulns.git

# Android Common Kernel
repo init -u https://android.googlesource.com/kernel/manifest -b common-android14-6.1

# linux-next
git clone https://git.kernel.org/pub/scm/linux/kernel/git/next/linux-next.git
```

### Useful queries

```bash
# Is a given mainline commit in a stable tree?
git log --oneline --grep="$(git log -1 --format=%s <mainline-sha>)" v6.6..linux-6.6.y

# What fixes went into a stable release?
git log --oneline v6.6.50..v6.6.51

# How many commits are in the stable branch beyond the .0 release?
git rev-list --count v6.6..linux-6.6.y

# Which commits in my tree are NOT upstream? (the delta)
git log --oneline --no-merges v6.6..my-product-branch

# Which of my patches have an upstream equivalent? (cherry-pick detection)
git cherry -v v6.6 my-product-branch
#   "-" means upstream already has an equivalent -> DROP IT
#   "+" means it is genuinely local

# What does a Fixes: tag point to, and is that commit in my tree?
git log -1 --format='%h %s' $(git log -1 --format=%B <sha> | grep -oP 'Fixes: \K[0-9a-f]+')
git merge-base --is-ancestor <that-sha> HEAD && echo "affected" || echo "not affected"
```

`git cherry` is the underused one. **Run it on every uprev**; it finds the patches you can
drop, and dropping patches is the only thing that makes the delta smaller.

---

## 2. Practice

### Lab 88.1 — Measure a real delta

```bash
#!/bin/bash
# delta_report.sh — characterize an out-of-tree delta.
# Usage: ./delta_report.sh <upstream-base-tag> <product-branch>
set -e
BASE=${1:-v6.6}
BRANCH=${2:-HEAD}

echo "=== Delta size ==="
echo "commits: $(git rev-list --count $BASE..$BRANCH)"
git diff --shortstat $BASE..$BRANCH

echo
echo "=== By subsystem (top 20) ==="
git diff --numstat $BASE..$BRANCH \
	| awk '{split($3,a,"/"); print a[1]"/"a[2]}' \
	| sort | uniq -c | sort -rn | head -20

echo
echo "=== Patches that already have an upstream equivalent (DROP THESE) ==="
git cherry -v $BASE $BRANCH | grep '^-' | head -40
echo "count: $(git cherry $BASE $BRANCH | grep -c '^-' || true)"

echo
echo "=== Genuinely local patches ==="
echo "count: $(git cherry $BASE $BRANCH | grep -c '^+' || true)"

echo
echo "=== Patches with no Signed-off-by (provenance problem) ==="
git log --format='%h %s' $BASE..$BRANCH | while read sha subj; do
	git log -1 --format=%B "$sha" | grep -q 'Signed-off-by:' || echo "$sha $subj"
done

echo
echo "=== Largest individual patches (review risk) ==="
git log --format='%h %s' $BASE..$BRANCH | while read sha subj; do
	n=$(git show --numstat --format= "$sha" | awk '{s+=$1+$2} END {print s+0}')
	echo "$n $sha $subj"
done | sort -rn | head -10
```

Run it on a real vendor BSP. **The output is almost always sobering**, and it is the artifact
that starts the conversation about upstreaming budget.

### Lab 88.2 — Backport a real fix

```bash
cd ~/src/linux-stable

# 1. Find a fix in mainline that is not in an older LTS.
git log --oneline --grep='Fixes:' v6.11..v6.12 -- drivers/foo/ | head

# 2. Try the clean cherry-pick.
git checkout -b backport linux-6.6.y
git cherry-pick -x <sha>

# 3. It conflicts. Now do the work:
#    a. Read the mainline commit fully.
git show <sha>
#    b. Read the mainline code around it.
git show <sha>:drivers/foo/foo.c | sed -n '100,200p'
#    c. Read YOUR tree's version of the same code.
sed -n '100,200p' drivers/foo/foo.c
#    d. What changed between them, and why?
git log --oneline v6.6..<sha> -- drivers/foo/foo.c

# 4. Decide: backport prerequisites, adapt, or decline. Write down why.

# 5. If adapting, resolve by INTENT.
git mergetool
git cherry-pick --continue

# 6. Document the adaptation in the [ ] note.
git commit --amend

# 7. TEST IT. On the affected hardware if at all possible, or with a
#    reproducer, or at minimum with the relevant selftests.
make -j$(nproc) && ./run_test.sh

# 8. Send it to stable@vger.kernel.org with the mainline SHA.
```

Do this for five real fixes of increasing difficulty. **The skill being built is judgement
about when to decline**, and you only develop it by doing the hard ones.

### Lab 88.3 — Build the continuous-update pipeline

```bash
#!/bin/bash
# stable_uprev.sh — rebase a product branch onto a new stable release.
set -e
OLD=${1:?old tag, e.g. v6.6.50}
NEW=${2:?new tag, e.g. v6.6.51}
BRANCH=${3:-product}

echo "=== What is in $NEW that was not in $OLD? ==="
git log --oneline "$OLD..$NEW" | tee /tmp/new_commits.txt
echo "count: $(wc -l < /tmp/new_commits.txt)"

echo
echo "=== Do any of them touch files we have patched? ==="
git diff --name-only "$OLD..$BRANCH" > /tmp/our_files.txt
git diff --name-only "$OLD..$NEW" > /tmp/their_files.txt
comm -12 <(sort -u /tmp/our_files.txt) <(sort -u /tmp/their_files.txt) \
	| tee /tmp/conflict_risk.txt
echo "at-risk files: $(wc -l < /tmp/conflict_risk.txt)"

echo
echo "=== Rebase ==="
git checkout "$BRANCH"
git rebase "$NEW" || {
	echo "CONFLICTS. Resolve, then: git rebase --continue"
	exit 1
}

echo
echo "=== Which of our patches are now redundant? ==="
git cherry -v "$NEW" | grep '^-' || echo "  none"

echo
echo "=== Build + boot ==="
make olddefconfig && make -j"$(nproc)"
./boot_test.sh

echo
echo "=== Which CVEs did this fix? ==="
for sha in $(git log --format=%H "$OLD..$NEW"); do
	grep -rl "$sha" ~/src/vulns/cve/published/ 2>/dev/null | xargs -r basename -s .json
done | sort -u
```

**Run it weekly.** The entire argument of this chapter is that continuous is cheap and
periodic is expensive, and the only way to believe it is to feel the difference.

### Lab 88.4 — CVE triage, the modern way

```bash
cd ~/src/vulns

# 1. Which CVEs affect my version?
MYVER=6.6.50
for f in cve/published/6.6/*.json; do
	python3 - "$f" "$MYVER" <<'EOF'
import json, sys
d = json.load(open(sys.argv[1]))
# Parse the affected version ranges; report if MYVER is in range.
# (The real script is more careful; this is the shape.)
print(d['cveMetadata']['cveId'], d['containers']['cna'].get('title',''))
EOF
done | head -20

# 2. The important realization: run the count.
ls cve/published/6.6/*.json | wc -l
#    -> hundreds to thousands. This is the point of §T.6.

# 3. The correct response is NOT to triage them individually. It is:
git log --oneline v6.6.50..v6.6.75 | wc -l
#    "We move to 6.6.75, which contains all of them."
```

Then write the one-page memo for a compliance-minded stakeholder explaining why per-CVE
triage is not the right strategy for the Linux kernel, and what the alternative control is
(continuous stable adoption + a tested update mechanism + a documented exception process).
**This memo is a real deliverable in every regulated-industry embedded project**, and being
able to write it is a distinguishing skill.

### Lab 88.5 — Shrink a delta

Take a real out-of-tree patch set and reduce it.

```bash
# 1. Categorize every patch.
git log --format='%h %s' v6.6..product | while read sha subj; do
	echo "$sha | $subj"
done > /tmp/delta.txt
# Then, by hand or with a script, tag each:
#   UPSTREAM-READY | BLOCKED:<reason> | PRODUCT-SPECIFIC | REDUNDANT

# 2. Drop the redundant ones first -- free wins.
git cherry -v v6.6 product | grep '^-'

# 3. For UPSTREAM-READY: clean up and submit (Ch. 86).
#    Most out-of-tree patches need work before they are submittable:
#    - no commit message, or a useless one
#    - debug hacks mixed with the real change
#    - hardcoded values that should be DT properties
#    - #ifdef CONFIG_VENDOR_HACK around everything

# 4. For BLOCKED: write down what is blocking and who owns unblocking it.

# 5. For PRODUCT-SPECIFIC: can it become a module? A DT property? A
#    Kconfig option? Every one you convert is one you stop carrying.

# 6. Re-measure and report.
./delta_report.sh v6.6 product
```

**Target a measurable reduction and report it.** "We reduced the delta from 412 patches to
280 and upstreamed 94" is a result; "we tried to upstream some things" is not.

### Lab 88.6 — Evaluate a vendor BSP

The skill an embedded architect is hired for.

```bash
# You are given a vendor BSP. Assess it before committing to it.

# 1. What is the base, really?
make kernelversion
git describe --tags 2>/dev/null
grep -r 'EXTRAVERSION\|VERSION =' Makefile | head -3

# 2. Is that base still supported?
#    Check kernel.org/category/releases.html for the EOL date.
#    Compare with your product's support commitment. If the kernel EOLs
#    first, that is your problem to solve, not the vendor's.

# 3. How big is the delta?
git log --oneline v<base>..HEAD | wc -l
git diff --shortstat v<base>..HEAD

# 4. What is the quality of the delta?
#    - Do commits have messages?
#    - Signed-off-by?
#    - Are there squashed mega-commits?
git log --format='%h %s' v<base>..HEAD | grep -icE 'wip|temp|fix|hack|xxx'

# 5. Is any of it upstream already?
git cherry v<base> HEAD | grep -c '^-'

# 6. How current is it against stable?
#    Does it have the latest v<base>.y fixes, or is it stuck at .0?
git log --oneline | grep -m1 "Linux <base>"

# 7. What is the vendor's update commitment? ASK, IN WRITING:
#    - How often do you rebase onto stable?
#    - How long will you support this version?
#    - What is your process for security fixes?
#    - Will you upstream, and on what timeline?
```

**Write the assessment memo**, with a recommendation and the conditions. This is directly a
staff-level deliverable, and steps 6 and 7 are the ones that most often reveal that a BSP is
not actually supported in any meaningful sense.

---

## 3. Mastery drills

1. Backport ten fixes of increasing difficulty to an LTS. Document each: clean pick, adapted
   (how), needed prerequisites (which), or declined (why).

2. Build and run a continuous stable-uprev pipeline for a product branch for three months.
   Measure the per-uprev cost and compare with an estimate of doing it once annually.

3. Take a vendor BSP and produce the full assessment of Lab 88.6, with a
   recommendation and a risk register.

4. Reduce a real out-of-tree delta by 25%, and document exactly how each patch was
   eliminated.

5. Get five previously out-of-tree patches merged upstream. Then verify they appear in the
   next LTS and drop your local copies.

6. Write the CVE-strategy memo of Lab 88.4 and present it to someone in a compliance role.
   Iterate until they accept it.

7. Study Android's GKI programme: read the KMI documentation, the ABI-checking tooling, and
   the delta-reduction reporting. Write up what is transferable to a non-Android product.

8. Compare three distributions' kernels (RHEL, SUSE, Ubuntu). Characterize each one's
   backport policy, kABI approach, and support model. Explain the differences in terms of
   their customers.

9. Take a `Fixes:` tag and trace its full lifecycle: the bug's introduction, the fix, the
   stable backport, and its appearance in a distribution kernel. Measure the elapsed time at
   each stage.

10. Design the kernel strategy for a hypothetical 10-year-lifetime industrial product: which
    LTS, what update mechanism, what delta policy, what security process, and what you will
    do when the LTS EOLs before the product does.

11. Build a tool that, given a product kernel and the `vulns` database, reports which known
    CVEs are unfixed and which are unreachable in your configuration. Be honest about the
    tool's limitations.

12. Write the out-of-tree patch policy for an engineering organization: who may add one, what
    justification is required, who owns it, and what the exit plan must be.

---

## 4. Further reading

**Mandatory**
- `Documentation/process/stable-kernel-rules.rst`
- `Documentation/process/cve.rst` — the CNA rationale; short and important
- `Documentation/process/stable-api-nonsense.rst` — Greg KH's essay; the definitive statement
  of why there is no in-kernel ABI
- `Documentation/process/security-bugs.rst`

**Reading**
- Greg Kroah-Hartman's talks on stable and LTS, and his "Why you should be using a supported
  kernel" talk in particular — the argument for continuous stable adoption, made by the
  person who runs it
- The kernel CVE announcement (Feb 2024) and the LWN coverage of the reaction
- Android GKI documentation: `source.android.com/docs/core/architecture/kernel/generic-kernel-image`
  and the KMI/ABI-monitoring pages
- LWN: "The kernel's CVE process," "Stable kernel maintenance," and the annual LTS coverage
- Konstantin Ryabitsev on kernel.org infrastructure and the `vulns` repository

**Data and tools**
- `kernel.org/category/releases.html` — which versions are supported and until when.
  **Bookmark it**; it is the input to every product-kernel decision
- `git.kernel.org/pub/scm/linux/security/vulns.git` — the CVE database
- `git.kernel.org/pub/scm/linux/kernel/git/stable/stable-queue.git` — what is pending
- `linuxkernelcves.com` (third-party, useful cross-reference)
- `git cherry`, `git rebase --onto`, `quilt`, `b4`

**Cross-references**
- Ch. 86 — the `Fixes:` tag and upstream submission
- Ch. 87 — regression handling and maintainer responsibility
- Ch. 98–99 — Yocto's kernel recipe and patch management, where this becomes concrete
- Ch. 102 — the security architecture these updates are defending

→ Next: [89-architect-playbook.md](89-architect-playbook.md)
