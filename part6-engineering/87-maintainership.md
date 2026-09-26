# Chapter 87 — Reviewing, Maintaining, and Subsystem Stewardship

> Chapter 86 was about getting your code in. This one is about being the person who decides
> whether other people's code goes in — which is a different job, with different skills, and
> it is the job that "senior" and "staff" actually describe. Most of what follows applies
> equally to an internal codebase; the kernel simply makes the dynamics unusually visible.

---

## Theory & First Principles

### T.0 — Start here: the job is saying no

A maintainer's inbox on a Monday:

```
  47 patches.  Of which, typically:
    ~12  correct and obviously fine                -> apply
    ~15  correct but the wrong shape               -> explain the right shape
     ~8  fix a real problem the wrong way          -> the hard conversation
     ~6  fix a problem that should not exist       -> push back on the design
     ~4  from someone who will never be seen again -> is it maintainable?
     ~2  break an ABI someone depends on           -> NAK, with reasons
```

**Only the first category is about code.** The other thirty-five are about *judgement*, and
that is the actual job.

**Because the asymmetry of the role is brutal, and worth stating numerically:**

```
   Accepting a patch:  5 minutes, now.
   Maintaining it:     10+ years, by YOU, for a contributor who may vanish
                       tomorrow, on hardware you do not own.
```

**A maintainer is not a gatekeeper who fears change — they are the person who will personally
be holding this code in 2035.** Every *why is this needed?* and *who will maintain this?* is
that asymmetry speaking. Once you see it, review feedback stops reading as obstruction.

**The three questions every maintainer is actually asking**, in this order — answer all three
in your cover letter before you are asked:

| Question | What a good submission does |
|---|---|
| **Is this the right problem?** | states the user-visible problem first, not the solution |
| **Is this the right shape?** | uses the subsystem's existing patterns; does not invent a parallel mechanism |
| **What is the long-term cost?** | no new ABI if avoidable; tests; documentation; a name in MAINTAINERS |

**Now the structural insight, which is why the kernel scales at all.** Maintainership is not a
committee — it is a **tree of delegated trust**:

```
                       Linus
                         |
        +----------------+----------------+
     net (Jakub)     mm (Andrew)     drivers/gpu (Dave)
        |                |                |
   +----+----+      +----+----+      +----+----+
  wifi  tcp  ...   slab  page  ...  i915  amd  ...
```

**Linus does not read 15,000 patches. He reads ~200 pull requests and trusts ~30 people, each
of whom trusts ~10 people.** Trust is earned by a public track record — which is exactly why
the mailing list is archived forever and why a `Reviewed-by:` tag carries weight. **Your
reputation is a permanent, queryable data structure.**

**And notice this is Ch. 00 §T.1's principle applied to humans:** Linus provides mechanism
(the merge window, the release cadence, the no-regressions rule); subsystem maintainers set
policy for their area. **Centralized invariants, decentralized decisions.**

**The one rule that is not delegated and that overrides every maintainer:**

> **We do not break userspace.**

A change that regresses a working userspace program is reverted — regardless of whether the
program relied on documented behaviour, regardless of whether the old behaviour was a bug,
and regardless of how elegant the fix is. **This single non-negotiable rule is why Linux
survived thirty years of refactoring without a userland schism**, and it is why Ch. 24 §T.0's
warning about UAPI matters so much: the kernel pays forever for interface mistakes, because
it has forbidden itself the usual escape.

**Practical advice for becoming trusted, in order of effectiveness:** review other people's
patches (this is scarce, and valued far above submitting your own); fix bugs reported by
others; take on the unglamorous maintenance; respond immediately to regressions you caused;
and add yourself to `MAINTAINERS` only when you mean it.

```bash
scripts/get_maintainer.pl -f mm/page_alloc.c
git shortlog -sn --since=1.year -- drivers/net/ | head
git log --format='%b' -200 | grep -c 'Reviewed-by'
sed -n '1,60p' Documentation/process/submitting-patches.rst
```

---

### T.1 — What a maintainer actually does

The naive model is "reviews patches." The real distribution of effort looks more like:

| Activity | Share of time | What it really is |
|---|---|---|
| **Reviewing** | ~30% | Correctness, but also *fit* |
| **Saying no** | ~15% | The highest-leverage and least visible work |
| **Herding** | ~15% | Chasing stalled series, pinging reviewers, unblocking |
| **Integration** | ~10% | Trees, rebases, conflicts, pull requests, `linux-next` |
| **Firefighting** | ~10% | Regressions, bisects, stable backports, syzbot |
| **Design direction** | ~10% | Where is this subsystem going |
| **Community** | ~10% | Mentoring, disputes, growing the next maintainer |

The reframing that matters: **a maintainer's product is not code, it is the long-term health
of a subsystem.** Every decision is evaluated on a five-to-ten-year horizon, which is why
maintainers reject things that are individually fine. A patch that works today but makes the
next ten patches harder is a net negative.

### T.2 — The three questions of review

Order matters. Most reviewers only ask the first and wonder why their reviews are not valued.

**1. Is it correct?** Does it do what it says, on all paths, under concurrency, with hostile
input, on all architectures?

**2. Is it *right*?** Is this the correct approach, at the correct layer, with the correct
abstraction? Correct code solving the wrong problem is still wrong.

**3. Does it *fit*?** Does it match the subsystem's idioms, its direction, its existing
abstractions? Will it make the next change easier or harder? Can the person who inherits this
understand it?

Question 3 is the maintainer's distinctive contribution, and it is the one a drive-by
reviewer cannot answer. It is also why maintainers sometimes reject technically-perfect code:
*fit* is a property of the whole, not of the patch.

### T.3 — The review checklist, in priority order

Work top to bottom and stop when you have enough to send. There is no point nitpicking
whitespace in a patch whose locking is wrong.

**Tier 1 — blocking, always**
- [ ] **Does it break the userspace ABI?** The unbreakable rule (Ch. 24). Anything that
      changes observable behaviour for an existing program is rejected, full stop.
- [ ] **Memory safety.** Every allocation checked, every resource released on every path,
      refcounts balanced, no use-after-free, no double free.
- [ ] **Locking.** Correct lock held, consistent order, nothing sleeps under a spinlock, no
      lock taken from a context that forbids it.
- [ ] **Concurrency.** What happens if two CPUs enter here simultaneously? What if one is in
      IRQ context?
- [ ] **Input validation.** Anything from userspace or a device: bounds, signs, overflow,
      double-fetch (Ch. 24, Ch. 102).
- [ ] **Error handling.** Every error path unwinds correctly and leaves consistent state.

**Tier 2 — usually blocking**
- [ ] Integer overflow, truncation, and 32-bit correctness.
- [ ] Endianness, alignment, and cross-architecture assumptions.
- [ ] Does each patch build and work independently? (bisectability)
- [ ] Is the commit message adequate?
- [ ] `Fixes:` tag present and correct? Should it go to stable?
- [ ] Is there a test, and if not, why not?

**Tier 3 — fit and quality**
- [ ] Does it match subsystem idioms?
- [ ] Is there an existing helper it should use instead?
- [ ] Is the abstraction at the right level?
- [ ] Will this scale (core count, device count, request rate)?
- [ ] Is the naming consistent with the surrounding code?
- [ ] Documentation, for anything user-visible.

**Tier 4 — nits**
- [ ] Style, whitespace, line length, comment wording.
- [ ] **Mark these explicitly as nits** so the author knows what is blocking.

### T.4 — Saying no well

The most important maintainer skill, and the one most rarely taught.

Every "yes" is a permanent maintenance commitment — the code will be in the tree for a
decade, someone will have to fix it, someone will have to understand it. Every "no" costs a
contributor's goodwill and possibly their future contributions. Both are real costs.

**The structure of a good rejection:**

1. **Thank them and acknowledge the problem.** Never dispute the need; disputing the need
   makes it personal and usually you are wrong about it anyway.
2. **State the objection precisely, in their currency.** Not "this is complex" but "this adds
   a lock to the fast path that all 200 drivers pay for, to serve one device."
3. **Offer a direction**, if you have one. "Have you looked at how `bar` solves this?"
4. **Say what would change your mind.** "If three drivers need this, I withdraw the
   objection." This converts an argument into a test, and it is the move that most reliably
   ends a deadlock.
5. **Be clear about what kind of no it is.**

The four kinds of no, which must be distinguished:

| Kind | Say |
|---|---|
| **Not like this** | "The approach is right, the implementation is not. Here is what to change." |
| **Not here** | "This belongs in userspace / in the core / in your own tree." |
| **Not yet** | "Come back when you have a second user / the dependency lands." |
| **Not ever** | "This cannot go in, and here is the structural reason." |

Failing to distinguish these wastes months. A contributor who hears "not like this" when you
meant "not ever" will send v2 through v7.

### T.5 — Trees, branches, and the integration pipeline

```
 contributor
     │ patch to the list
     ▼
 SUBSYSTEM MAINTAINER
     │  applies to:
     ├─ for-next  ──────► linux-next ──► build+boot tested on ~30 configs,
     │                                    many architectures, every night
     ├─ fixes     ──────► sent to Linus during -rc
     │
     │ merge window: git request-pull
     ▼
 LINUS
     │
     ▼
 v6.N ──► stable maintainers ──► v6.N.y  (→ Ch. 88)
```

The branch discipline a maintainer keeps:

| Branch | Contains | Rebased? |
|---|---|---|
| `for-next` | Everything for the next merge window | **No** — `linux-next` depends on stability |
| `fixes` | Fixes for the current -rc | No |
| `for-6.N` | A tagged snapshot to send to Linus | No |
| `testing` | Not yet committed to | Yes, freely |

**The cardinal rule: once a branch is in `linux-next`, it must not be rebased.** Other
maintainers may have merged it; rebasing breaks their trees and destroys the testing that
`linux-next` provided. If you must remove a patch, revert it.

The pull request:

```bash
git tag -s -a foo-6.13 -m "foo subsystem updates for 6.13"
git push origin foo-6.13
git request-pull v6.12 https://git.kernel.org/pub/scm/linux/kernel/git/you/foo.git \
	foo-6.13 > /tmp/pr.txt
# Then edit /tmp/pr.txt to add a summary of what is in it, and email it
# to Linus with linux-kernel Cc'd.
```

`request-pull` verifies the tag is actually pushed and reachable — a check that exists because
people got it wrong. The summary you add is what appears in Linus's merge commit, so write it
for someone who has not been following your subsystem.

### T.6 — Handling regressions

The kernel's strongest cultural rule after "don't break userspace" is **regressions are the
highest priority**. A regression is any case where something that worked in release N does
not work in release N+1.

The protocol when one is reported:

1. **Believe the reporter.** The reflex is to assume misconfiguration. The reflex is usually
   wrong and always costly.
2. **Get a bisect.** Ask for one, or do it yourself. `git bisect` is the single most
   effective tool here (Ch. 06).
3. **Fix or revert, quickly.** A revert is not a failure; it is the correct action when a fix
   is not immediately obvious. The tree's health outranks the patch's author's feelings,
   including yours.
4. **Do not require the reporter to justify their configuration.** "Why would you do that?"
   is not a valid response to a regression. They did it; it worked; now it does not.
5. **`Fixes:` and `Cc: stable`** on the fix.

Linus has reverted merged features over regressions and will again. The `regressions@lists.linux.dev`
list and Thorsten Leemhuis's regression tracking exist to make sure none is lost.

### T.7 — Growing the next maintainer

Every maintainer eventually burns out, changes jobs, or dies. Subsystems that did not grow a
successor become orphaned, and orphaned subsystems rot. This is a real and recurring failure
mode in the kernel, and the same failure mode exists in every company.

What works:

- **Review generously but honestly.** Spend real time on a newcomer's first patches; explain
  the *reasoning*, not just the correction. The investment compounds.
- **Delegate visibly.** Ask someone to review a series and say publicly that you trust their
  judgement on it.
- **Add reviewers to `MAINTAINERS`** before they are maintainers. It signals trust and routes
  patches to them.
- **Let them make decisions you would have made differently**, when the cost is bounded. A
  deputy who is overruled every time does not become a maintainer.
- **Say "ask X, they know this area"** and mean it.

What does not work: waiting until you are burnt out to look for a successor, and gatekeeping
in the name of quality until nobody else knows the code.

### T.8 — Conflict, and the Code of Conduct

The kernel's reputation for hostility is substantially dated — the 2018 Code of Conduct and
Linus's public change in behaviour altered the register significantly — but conflict still
happens, and knowing how it is handled is part of operating here.

The escalation ladder:

1. **Technical disagreement** → argue it on the list, with evidence. Most disputes end here
   and should.
2. **Deadlock between maintainers** → the maintainer whose tree it goes through decides. If
   that is unclear, the next level up (the "tree" maintainer, or Linus).
3. **Repeated bad-faith behaviour** → the Code of Conduct committee
   (`conduct@kernel.org`). It is confidential and it does act.
4. **Maintainer is unresponsive or obstructive** → escalate on-list, involve the
   `MAINTAINERS` fallback or the TAB (Technical Advisory Board).

The rules of engagement that keep disputes productive:
- **Argue about the code, never the person.** "This patch has a race" not "you don't
  understand locking."
- **Concede when you are wrong**, promptly and clearly. It costs nothing and buys a great
  deal.
- **State what evidence would change your mind.** If nothing would, you are not having a
  technical argument.
- **Escalate late and rarely.** A maintainer who escalates every dispute stops being
  consulted.

### T.9 — The `MAINTAINERS` file

```
FOO SUBSYSTEM DRIVER
M:	Your Name <you@example.com>          # Maintainer
R:	Reviewer Name <rev@example.com>      # Designated reviewer
L:	linux-foo@vger.kernel.org            # Mailing list
S:	Maintained                           # Status
W:	https://foo.example.com              # Web page
T:	git git://git.kernel.org/.../foo.git # Tree
F:	drivers/foo/                         # Files
F:	include/linux/foo.h
F:	Documentation/devicetree/bindings/foo/
K:	\bfoo_register\b                     # Keyword regex
```

The `S:` field is a **commitment**, and people read it:

| Status | Means |
|---|---|
| `Supported` | Someone is **paid** to maintain this |
| `Maintained` | Someone maintains it, not necessarily as their job |
| `Odd Fixes` | Someone occasionally looks at it |
| `Orphan` | Nobody. Patches may sit forever |
| `Obsolete` | Being removed; do not build on it |

Declaring `Maintained` and then not responding for six months is worse than declaring
`Odd Fixes`, because contributors waste effort on the basis of your claim.

### T.10 — Scaling, and the limits of one person

At some point the volume exceeds one person. The options, roughly in order of preference:

1. **Delegate by area** — co-maintainers with clear boundaries, each with their own `F:`
   lines.
2. **Add reviewers (`R:`)** — they do not apply patches but their `Reviewed-by:` lets you
   apply faster.
3. **Automate** — bots for style, build, boot, and regression testing. Every check a bot
   performs is a review comment you never have to write.
4. **Raise the bar on new features** — if you cannot maintain more, accept less.
5. **Split the subsystem** — separate `MAINTAINERS` entries and separate trees.
6. **Step down, publicly and with a successor.** Doing this well is an act of stewardship,
   not failure.

The anti-patterns: reviewing less carefully to keep up (which converts a review bottleneck
into a bug bottleneck), applying patches without review, and silently becoming the
`Orphan` you have not declared.

---

## 1. Internals

### Documentation

| Path | Contents |
|---|---|
| `Documentation/maintainer/` | **The maintainer handbook** — read it all |
| `Documentation/maintainer/configure-git.rst` | Signing, `request-pull` setup |
| `Documentation/maintainer/pull-requests.rst` | How to send one correctly |
| `Documentation/maintainer/rebasing-and-merging.rst` | **When rebasing is and is not acceptable** |
| `Documentation/maintainer/maintainer-entry-profile.rst` | Documenting your own expectations |
| `Documentation/maintainer/modifying-patches.rst` | When and how you may edit someone's patch |
| `Documentation/process/management-style.rst` | Linus's essay; short, funny, and accurate |
| `Documentation/process/handling-regressions.rst` | The regression protocol |
| `Documentation/process/researcher-guidelines.rst` | After the UMN incident; how to engage as a researcher |
| `MAINTAINERS` | The file itself; read the header |

### The maintainer entry profile

A newer and genuinely useful convention: a per-subsystem document stating your expectations,
so contributors do not have to guess.

```rst
.. SPDX-License-Identifier: GPL-2.0

Foo Subsystem
=============

Overview
--------
The foo subsystem handles ...

Submit Checklist Addendum
-------------------------
- Patches must be tested on at least one real device; state which.
- New device support requires a devicetree binding patch in the same
  series.
- Performance-affecting changes require before/after numbers from
  ``tools/testing/foo/bench``.

Key Cycle Dates
---------------
New feature submissions are accepted up to -rc6. After that only fixes.

Review Cadence
--------------
Expect an initial response within one week. If you have not heard
anything after two weeks, please ping -- it means the patch was missed,
not rejected.
```

Writing one for any codebase you own is cheap and eliminates most process friction. It is a
good thing to point at in an interview when asked how you manage a team's code.

### Tooling for maintainers

```bash
# --- Intake ---
b4 mbox -o /tmp <msgid>            # fetch a thread
b4 am -o /tmp <msgid>              # fetch as an applyable mbox, trailers applied
b4 shazam <msgid>                  # fetch and apply in one step
b4 diff <msgid>                    # v2 vs v3 -- what actually changed
b4 ty -S                           # generate "thanks, applied" replies

# --- Verification before applying ---
./scripts/checkpatch.pl --strict -g <range>
git rebase <base> --exec 'make -j$(nproc)'          # every commit builds
make W=1 C=1 -j$(nproc)
make allmodconfig && make -j$(nproc)

# --- Provenance ---
./scripts/get_maintainer.pl --self-test
git log --format='%an <%ae>' <range> | sort -u      # who wrote it
./scripts/checkpatch.pl --strict <patch> | grep -i 'signed-off'

# --- Integration ---
git request-pull <base> <url> <tag>
git tag -s -a foo-6.13 -m "..."

# --- Finding stalled work ---
# patchwork: https://patchwork.kernel.org/project/<your-project>/list/
# Filter by state=new, sort by age.
```

---

## 2. Practice

### Lab 87.1 — Review a real series, properly

```bash
# 1. Pick a series from lore in an area you know.
b4 am -o /tmp <message-id>
git checkout -b review v6.12
git am /tmp/*.mbx

# 2. Verify the mechanics BEFORE reading the code.
git rebase v6.12 --exec 'make -j$(nproc) >/dev/null' \
	|| echo "FAILS TO BUILD AT SOME COMMIT"
./scripts/checkpatch.pl --strict -g v6.12..HEAD

# 3. Read each patch with the Tier 1 checklist in hand.
git log -p --reverse v6.12..HEAD

# 4. Build and boot it.
make -j$(nproc)
qemu-system-x86_64 -kernel arch/x86/boot/bzImage ... # Ch. 04

# 5. Run the relevant tests.
make -C tools/testing/selftests TARGETS=<subsystem> run_tests
```

Then write the review. **Structure it like this:**

```
On Mon, Jan 06, 2025 at 10:23:45AM +0000, Author wrote:
> +	spin_lock(&dev->lock);
> +	entry = kmalloc(sizeof(*entry), GFP_KERNEL);

GFP_KERNEL under a spinlock -- this can sleep. Either move the
allocation before the lock, or use GFP_ATOMIC (and handle the much
higher failure rate).

> +	list_add(&entry->node, &dev->entries);
> +	spin_unlock(&dev->lock);

Also, `entry` is not checked for NULL before dereferencing it above.

> +static void foo_cleanup(struct foo_dev *dev)

nit: the rest of this file uses `foo_dev_*` as the prefix for functions
taking a struct foo_dev.

Otherwise this looks good to me. With the allocation issue fixed:

Reviewed-by: Your Name <you@example.com>
```

Note the shape: **blocking issues first, specifically, with the fix implied; nits explicitly
marked as nits; and a conditional tag so the author knows what "done" looks like.**

### Lab 87.2 — Practise the four kinds of no

Write a rejection for each of these, using the §T.4 structure. Then have someone critique
your tone.

**Scenario A.** A contributor sends a patch adding a module parameter to work around a bug in
their board's firmware. It works. It is 12 lines.
> *(The right answer is probably "not like this" — a quirk keyed off the DT compatible or
> DMI, not a global module parameter. Explain why a module parameter is the wrong shape: it
> is a UAPI you can never remove, it requires the user to know about it, and it does not
> scale to the second affected board.)*

**Scenario B.** A contributor proposes a new syscall to expose a statistic their monitoring
system wants.
> *("Not here." A syscall is permanent; a trace event or a `/sys` file costs nothing and can
> evolve. Offer the alternative.)*

**Scenario C.** A contributor sends a well-written generic framework with exactly one user —
their driver.
> *("Not yet." Ask for the third user. Explain that a one-user abstraction encodes that
> user's assumptions and is usually wrong in ways that only show up with the second one.)*

**Scenario D.** A contributor proposes a change that alters an error code returned to
userspace, because the current one is wrong.
> *("Not ever," probably. Hyrum's Law: something depends on the current errno. If it is
> genuinely unreachable or provably unused, that argument has to be made explicitly.
> Otherwise the wrong errno is now permanent.)*

### Lab 87.3 — Maintain something for three months

The only way to learn this.

```bash
# 1. Find something unmaintained that you use and understand.
grep -B5 'S:.*Orphan' MAINTAINERS | head -40
grep -B5 'S:.*Odd Fixes' MAINTAINERS | head -40
# Or: a driver for hardware you own with no listed maintainer.

# 2. Demonstrate competence first: fix three real bugs in it and get
#    them merged. Do NOT open by claiming maintainership.

# 3. Then propose yourself.
#    A patch adding your MAINTAINERS entry, Cc'ing the previous
#    maintainer (if reachable) and the subsystem list, with a short
#    note explaining what you intend to do.

# 4. Set up your workflow: a tree, a for-next branch, patchwork or a
#    b4-based intake, and a testing setup.

# 5. Actually do it for three months. Respond within a week. Every time.
```

Even if you do not become a kernel maintainer, **doing this inside your company for an
internal component teaches the same lessons** — the dynamics of intake, backlog, saying no,
and growing reviewers are identical.

### Lab 87.4 — Run the integration pipeline

```bash
# Simulate a maintainer's release cycle on a scratch tree.

# 1. Set up the branches.
git checkout -b for-next v6.12
git checkout -b fixes v6.12

# 2. Apply incoming patches.
b4 shazam <msgid-of-a-feature>     # -> for-next
git checkout fixes
b4 shazam <msgid-of-a-fix>         # -> fixes

# 3. Verify every commit.
git rebase v6.12 --exec './scripts/checkpatch.pl -g HEAD && make -j$(nproc)'

# 4. Test the merge of both against upstream -- find conflicts EARLY,
#    not at merge-window time.
git checkout -b integration v6.12
git merge for-next fixes
make -j$(nproc)

# 5. Produce a pull request.
git tag -s -a foo-6.13 -m "foo updates for 6.13"
git request-pull v6.12 <your-url> foo-6.13

# 6. Write the summary that goes above the diffstat. Write it for
#    someone who has not read your list.
```

### Lab 87.5 — Handle a regression, start to finish

```bash
# A user reports: "my device stopped working between 6.11 and 6.12."

# 1. Believe them. Gather: exact versions, .config diff, dmesg from both,
#    hardware details.

# 2. Reproduce, or get them to bisect.
git bisect start v6.12 v6.11
git bisect run ./test_script.sh     # automate it if at all possible
# Give the reporter a script if they are doing the bisect.

# 3. Confirm the bisect result by reverting on top of v6.12.
git revert <bad-commit>
make && test

# 4. Decide: fix or revert?
#    Fix if it is obvious and small and you are confident.
#    REVERT if it is not. Revert now, fix properly later. The tree's
#    health outranks the feature.

# 5. The fix (or revert) commit message:
cat <<'EOF'
foo: fix device detection on boards without the bar property

Commit 0123456789ab ("foo: use the bar property for detection")
assumed every board provides the "vendor,bar" property. Boards using
the older binding do not, and now fail to probe with -ENODEV.

Fall back to the legacy detection path when the property is absent.

Fixes: 0123456789ab ("foo: use the bar property for detection")
Reported-by: Reporter Name <reporter@example.com>
Closes: https://lore.kernel.org/r/...
Tested-by: Reporter Name <reporter@example.com>
Cc: stable@vger.kernel.org # v6.12
Signed-off-by: Your Name <you@example.com>
EOF

# 6. Tell the reporter, and thank them. Publicly.
```

### Lab 87.6 — Write a maintainer entry profile

For a subsystem you maintain, or a component you own at work. It should answer, in one page:

- What is in scope and what is not.
- What a submission must include (tests? numbers? hardware statement? bindings?).
- What the review cadence is, and when to ping.
- What the branch and release model is.
- What you will reject on sight, and why.
- Who to escalate to.

**Then measure whether it worked**: count the process-related back-and-forth on submissions
before and after. This is one of the highest-return documents you can write for any team.

### Lab 87.7 — Study a rejection

```bash
# Find a significant rejected proposal on lore and read the entire thread.
# Good candidates: any large new subsystem that did not merge, a
# contentious API proposal, a rejected syscall.
```

Write up:
1. What was proposed, and what problem it solved.
2. What the objections were — and separate the *stated* objection from the *underlying*
   concern, which is usually maintenance burden or ABI permanence.
3. Which of the four kinds of no it was.
4. Whether the decision was right, in hindsight.
5. What the proposer could have done differently.

**Do this for three different threads.** Patterns emerge quickly, and they are the same
patterns that govern design review at any company.

---

## 3. Mastery drills

1. Review twenty patches in a subsystem you know, over a month. Track: how many of your
   comments the maintainer agreed with, how many the author acted on, and how many were nits.
   Adjust your ratio.

2. Get listed as `R:` (reviewer) in `MAINTAINERS` for something. Then actually review the
   patches that get routed to you, for six months.

3. Maintain something — upstream or internal — for a full release cycle. Run the intake, the
   integration, the pull request, and the post-release fixes.

4. Write rejections for the four scenarios in Lab 87.2 and have three colleagues rate them on
   clarity, tone, and whether they would send v2 after receiving it.

5. Take a subsystem and produce a written five-year direction: what should be deleted, what
   should be abstracted, what technical debt is load-bearing and what is not. Present it.

6. Handle a real regression end to end, including the bisect, the decision to fix or revert,
   the stable tag, and the communication with the reporter.

7. Build the automated gate you would want as a maintainer: on every incoming series,
   checkpatch, per-commit build, boot test, and the subsystem's selftests, with a report
   posted back. Measure how many review comments it makes unnecessary.

8. Study three subsystems with very different maintainer styles (netdev, tip, drm, for
   instance). Characterize each, and explain what about the subsystem drives the style.

9. Identify an orphaned subsystem, assess what it would take to revive it, and write the
   proposal.

10. Mentor someone through their first upstream patch, from finding the bug to seeing it in a
    release. Write down every piece of tacit knowledge you had to transfer — that list is the
    real curriculum.

11. Read `Documentation/process/management-style.rst` and write your own, honestly, for the
    codebase you own.

12. Write the succession plan for a component you maintain: who could take it, what they
    would need to learn, and what you would have to document. Then start executing it.

---

## 4. Further reading

**Mandatory**
- `Documentation/maintainer/` — the whole directory
- `Documentation/process/management-style.rst` — Linus on maintainership; short, funny,
  and more serious than it appears
- `Documentation/process/handling-regressions.rst`
- `Documentation/maintainer/rebasing-and-merging.rst` — when rebasing is destructive
- `Documentation/maintainer/maintainer-entry-profile.rst`
- `Documentation/process/code-of-conduct-interpretation.rst`

**Reading**
- The annual **Maintainers Summit** reports on LWN — the best available record of how the
  kernel's governance actually works, and of the problems maintainers are currently worried
  about (almost always: burnout, succession, and review capacity)
- Jonathan Corbet's LWN development-statistics articles, per release
- Greg Kroah-Hartman's talks on maintainership and on stable
- Thorsten Leemhuis's regression-tracking reports
- The UMN "hypocrite commits" incident and the resulting
  `Documentation/process/researcher-guidelines.rst` — a case study in trust, and in how a
  community responds to bad-faith contribution

**Books**
- Fogel, *Producing Open Source Software* — free online; the best general book on running a
  project, and much of it maps directly
- Raymond, *The Cathedral and the Bazaar* — dated, still worth reading for the framing
- Nadia Eghbal, *Working in Public* — the modern analysis of maintainer burnout and the
  economics of open source. Read it if you plan to maintain anything

**Practice**
- `lore.kernel.org` — read review threads in a subsystem you care about, weekly. Absorbing
  how experienced maintainers phrase objections is worth more than any written advice
- `patchwork.kernel.org` — see the actual backlog a maintainer faces
- The `linux-next` daily reports — integration problems, surfaced early

→ Next: [88-stable-backporting.md](88-stable-backporting.md)
