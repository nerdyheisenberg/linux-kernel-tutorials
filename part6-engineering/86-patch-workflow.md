# Chapter 86 — The Patch Workflow: git, `b4`, Mailing Lists, Review

> **Part 6 — Upstream engineering & architecture.** Four chapters on the part of kernel work
> that is not code: getting change accepted, maintaining a subsystem, shipping to products,
> and exercising technical judgement at scale. This closes the last gap against Robert Love's
> *Linux Kernel Development* (his chapter 20), and it is the material that most distinguishes
> a senior engineer from a competent one.
>
> It is also directly interviewed. "Have you upstreamed anything?" is a real question, and
> "no, but here is exactly how I would" is a much weaker answer than "yes, here is the
> commit."

---

## Theory & First Principles

### T.0 — Start here: why your first patch will be rejected for its whitespace

You fix a real bug. You send it. The reply is:

> *Your patch is corrupted by your mail client. Also: the subject line is wrong, this should
> be three patches not one, there is no Fixes: tag, and you are missing a Signed-off-by.
> Please resend.*

Not one word about whether your fix is correct. **This feels like gatekeeping. It is not.**
Understanding why is the difference between a frustrating first year and a productive one.

**Start with the scale, because the process only makes sense against it:**

```
  ~4,000 developers per release
  ~15,000 commits per release   -> about 8 accepted commits PER HOUR, continuously
  ~70-day release cycle
  ~2,000 subsystems with distinct maintainers
  ~30,000,000 lines that must all keep working, on ~25 architectures
```

**At eight commits an hour, every second of a maintainer's attention is a contended
resource.** A malformed patch is not an aesthetic problem; it is a patch that cannot be
applied with `git am`, cannot be reviewed inline, and therefore costs a human ten minutes
instead of zero. Multiply by the queue.

**So every rule you will be told off for exists to make review cheap:**

| The rule | What it actually buys |
|---|---|
| One logical change per patch | a reviewer holds one idea; a bisect lands on one cause; a revert removes one thing |
| `subsystem: short imperative summary` | a maintainer triages 200 subjects without opening any |
| Explain **why**, not **what** | the diff already says what; the reason is what gets lost in three years |
| A `Fixes:` tag naming the broken commit | the **stable** trees find the backport target automatically (Ch. 88) |
| `Signed-off-by:` | the Developer Certificate of Origin — a legal chain of provenance |
| Plain-text email, no HTML | the patch must survive as text and be quotable inline |
| `checkpatch.pl` clean | machines do the mechanical review so humans do not |

**Take one-logical-change-per-patch seriously, because it is the rule people resist and the
one that matters most.** It is not a style preference — it is what makes `git bisect` a
precision instrument. A tree taking 15,000 commits per release is debuggable at all *only*
because bisect can point at one small change. A patch that fixes a bug, renames three
variables, and reformats a function destroys that property for everything downstream of it,
forever. **Reviewability and bisectability are properties of the *series*, not of any single
patch.**

**And the email question, which every newcomer asks.** Why not a web forge?

```
  What email gives you that a forge does not:
    * works offline, scriptable, greppable, archived forever
    * review happens INLINE, quoting the exact lines -- the highest-bandwidth
      code-review medium anyone has built
    * no central server that anyone must trust or fund
    * 2,000 maintainers each use their own tooling over one plain format
    * a thread is the unit, so a 30-patch series is one conversation
```

**The deeper point is not about email at all:** the workflow is optimized for **review
throughput and long-term archival**, not for contributor onboarding. Those are different
objectives and the kernel deliberately chose the first. `b4` and `patchwork` have since
removed most of the mechanical pain — use them from day one rather than fighting your mail
client.

**A final reframing that will change how you read feedback:** a terse review is not hostility,
it is compression. *NAK, this breaks the ABI* is five words that would take a paragraph to
soften, times two hundred patches a day. **Read for the technical content and discard the
tone**; the signal is almost always there.

```bash
scripts/checkpatch.pl --strict -g HEAD~1..HEAD
scripts/get_maintainer.pl -f drivers/foo/bar.c
b4 prep --new my-feature && b4 send --dry-run
git log --format='%s' -20 drivers/net/     # learn the subject-line conventions
```

---

### T.1 — Why email, and why it is not going to change

Engineers new to the kernel invariably ask why it does not use GitHub. The answer is not
nostalgia, and being able to give it properly is a small but real signal.

| Property | Email + `git` | Forge (GitHub/GitLab) |
|---|---|---|
| **Scale** | ~75,000 commits/year, ~2,000 developers/release, ~30 subsystem trees | Forges degrade badly at this rate |
| **Decentralization** | No single company controls the infrastructure | One vendor, one account system |
| **Archival** | `lore.kernel.org` is a complete, greppable, `git`-backed archive since 1995 | Data lives in a proprietary database |
| **Review granularity** | Inline reply, quoting exactly the hunk in question | Comment threads bound to line numbers that drift |
| **Offline** | Fully | No |
| **Cross-subsystem** | Cc: several lists, one thread | Cross-repo review is painful |
| **Tooling** | `git`, `b4`, `mutt`, `patchwork`, scripts | Whatever the forge offers |

The honest counterpoint: the barrier to entry is **much** higher, and that is a real cost in
contributor pipeline. `b4` exists specifically to lower it, and it works — a modern patch
submission is a handful of commands, not a fight with an email client.

**What actually matters:** the workflow is *patch-centric*, not branch-centric. The unit of
review is a self-contained, individually-correct commit. That shapes everything else.

### T.2 — The one rule that dominates: one logical change per patch

```
 BAD:  "fix driver"
       - new feature
       - unrelated cleanup
       - whitespace changes
       - a bug fix
       - a rename
       1,200 lines, 1 commit

 GOOD: 1/6 driver: rename foo_bar to foo_baz for clarity   (no behaviour change)
       2/6 driver: factor out foo_setup()                  (no behaviour change)
       3/6 driver: fix off-by-one in the ring index         (the actual fix)
       4/6 driver: add support for variant B                (the feature)
       5/6 driver: add tracepoints for submission
       6/6 dt-bindings: document vendor,variant-b
```

Four reasons, and you should be able to give all four:

1. **Reviewability.** A reviewer can hold one change in their head. A 1,200-line patch gets
   either a rubber stamp or no review at all, and both are failures.
2. **Bisectability.** `git bisect` (Ch. 06) requires every commit to build and work. A
   bisect landing in the middle of a broken series is a wasted day for whoever hits it.
3. **Revertability.** If patch 4 regresses, revert patch 4. If everything is one commit, you
   revert the fix too.
4. **Backportability.** Stable maintainers (Ch. 88) want the minimal fix. A fix tangled with
   a refactor will not be backported, so your users will not get it.

The corollary rule: **the patch must build and work at every point in the series.** Not just
at the end. This is what "each patch is a complete, correct change" means, and it is the
thing people most often get wrong.

### T.3 — Anatomy of a commit message

The commit message is the *durable* artifact. Code says what; the message says why, and in
five years the why is what nobody remembers.

```
subsystem: component: short imperative summary under ~70 chars

The problem, stated first. What is broken, or what is missing, and how
do you know. Include the symptom a user sees, and the conditions under
which it occurs.

Then the analysis: why does it happen? What is the mechanism? This is
where you explain the race, the ordering violation, the wrong bound.

Then the fix: what this patch does, and why this approach rather than
the obvious alternative. If you considered and rejected something, say
so -- it saves the reviewer from suggesting it.

Anything a reviewer or a future archaeologist needs: performance
numbers, the hardware it was tested on, what it does NOT fix.

Fixes: 0123456789ab ("subsystem: the commit that introduced the bug")
Cc: stable@vger.kernel.org # v6.1+
Reported-by: Someone <someone@example.com>
Closes: https://bugzilla.kernel.org/show_bug.cgi?id=12345
Signed-off-by: Your Name <you@example.com>
---
v3: addressed Reviewer's comment about the locking (Reviewer)
v2: split the cleanup into a separate patch
```

Rules, each with a reason:

| Rule | Why |
|---|---|
| **Imperative mood** ("fix", not "fixed"/"fixes") | Matches `git`'s own convention: the subject completes "if applied, this commit will…" |
| **Prefix with the subsystem** | Thousands of commits per release; the prefix is how people filter. Use `git log --oneline <file>` to find the local convention |
| **Explain *why*, not *what*** | The diff already says what |
| **No "this patch"** | Say what the change does, not what the patch does |
| **Wrap at 75 columns** | Email and `git log` readability |
| **`Fixes:` with 12-char SHA and subject** | Drives automated stable backporting |
| **Changelog below `---`** | `git am` strips it; it is for reviewers, not history |

The `Fixes:` tag is the single highest-leverage line you can add. It is machine-read by the
stable tooling, by distributions, and by CVE triage. Get the format exactly right:

```bash
git config --global alias.fixes "show -s --pretty=format:'Fixes: %h (\"%s\")'"
git fixes <sha>
# or:  git log -1 --abbrev=12 --format='Fixes: %h ("%s")' <sha>
```

### T.4 — The trailer vocabulary

Each trailer has a precise meaning, and using them loosely is a common novice error.

| Trailer | Means | Who adds it |
|---|---|---|
| `Signed-off-by:` | **A legal statement** — see §T.5. Mandatory | author, and every maintainer in the chain |
| `Reviewed-by:` | "I read this carefully and believe it is correct" | reviewer |
| `Acked-by:` | "I am responsible for this area and I am fine with it going in" — weaker than Reviewed-by | maintainer of an affected area |
| `Tested-by:` | "I ran it and it works" | anyone |
| `Reported-by:` | who found the bug | author, with permission |
| `Suggested-by:` | who proposed the approach | author |
| `Co-developed-by:` | joint authorship; **must be paired with their `Signed-off-by:`** | author |
| `Fixes:` | which commit introduced the bug | author |
| `Closes:` | the bug-tracker URL this resolves | author |
| `Cc: stable@vger.kernel.org` | request backport; `# v6.1+` restricts the range | author or maintainer |
| `Link:` | to the mailing-list discussion | maintainer, automatically |

**You add tags people gave you; you do not add tags on their behalf.** Adding a
`Reviewed-by:` someone did not send is a serious breach. Conversely, when you repost a
series, you **must carry forward** the tags you received — dropping them silently wastes the
reviewers' time and they will notice.

The exception: if the patch changed substantively since the tag was given, drop the tag and
say why in the changelog. Carrying a `Reviewed-by:` across a rewrite is equally bad.

### T.5 — The Developer's Certificate of Origin

`Signed-off-by:` is not a formality. It is a specific legal assertion, reproduced in
`Documentation/process/submitting-patches.rst`:

> **(a)** The contribution was created in whole or in part by me and I have the right to
> submit it under the open source license indicated in the file; **or**
> **(b)** it is based upon previous work that, to the best of my knowledge, is covered under
> an appropriate open source license and I have the right under that license to submit it;
> **or (c)** it was provided directly to me by someone who certified (a), (b) or (c) and I
> have not modified it.
> **(d)** I understand and agree that this project and the contribution are public and that a
> record of the contribution is maintained indefinitely.

Consequences that matter in practice:

- **The name must be your real legal name**, matching how you would sign a document. Pseudonyms
  are not accepted.
- **If your employer owns the copyright**, you need their authorization — most large
  contributors have a blanket policy; check yours before your first patch.
- **The sign-off chain is a provenance record.** A maintainer applying your patch adds their
  own, asserting (c).
- **AI-generated code** is an open question the community is actively working through. The
  emerging position is that you must be able to certify (a) — i.e. you are responsible for
  the code regardless of how it was produced. Check the current
  `Documentation/process/` guidance before submitting anything you did not write line by line.

### T.6 — `b4`: the tool that makes this tractable

`b4` (Konstantin Ryabitsev) is the single biggest quality-of-life improvement in kernel
contribution, and using it marks you as someone who has done this before.

```bash
# ---- Contributor side ----
b4 prep -n my-feature -f v6.12              # start a tracked branch
# ... make commits ...
b4 prep --edit-cover                        # write the cover letter
b4 prep --auto-to-cc                        # run get_maintainer.pl, fill To:/Cc:
b4 send --dry-run                           # inspect exactly what will be sent
b4 send                                     # send it

# ... after review ...
b4 trailers -u                              # pull Reviewed-by/Tested-by from the list
                                            #   INTO your commits, automatically
b4 prep --edit-cover                        # add the v2 changelog
b4 send                                     # sends v2, with In-Reply-To set correctly

# ---- Reviewer / maintainer side ----
b4 mbox <message-id>                        # fetch the whole thread
b4 am -o /tmp <message-id>                  # fetch the series as an mbox, with
                                            #   all trailers already applied
b4 shazam <message-id>                      # fetch AND apply to the current branch
b4 diff <message-id>                        # diff v2 against v1 -- invaluable
b4 ty                                       # generate thank-you/applied replies
```

`b4 trailers -u` deserves emphasis: it reads the list archive, finds every `Reviewed-by:`
and `Tested-by:` on your series, and inserts them into the right commits. Doing this by hand
across a 12-patch series is where people make mistakes.

`b4 diff` is the reviewer's equivalent: "show me what changed between v2 and v3," which is
what a reviewer actually wants and what email alone does not give you.

### T.7 — Finding the right recipients

```bash
# THE tool. Run it on your patch, not on the files.
./scripts/get_maintainer.pl 0001-my-patch.patch

# What it outputs and what to do with it:
#   "maintainer"     -> To:
#   "reviewer"       -> Cc:
#   "supporter"      -> Cc:
#   list addresses   -> Cc:
#   linux-kernel@vger.kernel.org -> ALWAYS Cc:, always

# Useful variants:
./scripts/get_maintainer.pl --no-git-fallback 0001-*.patch  # skip the
    # "recently touched this file" heuristic, which adds noise
./scripts/get_maintainer.pl -f drivers/foo/bar.c            # by file
```

The rules:
- **Always Cc `linux-kernel@vger.kernel.org`.** It is the archive of record.
- **To: the maintainers, Cc: everyone else.** Maintainers are the people who will apply it.
- Cc the **subsystem list**, and Cc `linux-kernel`.
- For device tree changes, Cc `devicetree@vger.kernel.org` and the DT maintainers.
- Cc anyone whose `Reported-by:` or `Suggested-by:` you used.
- **Do not** Cc `stable@vger.kernel.org` directly on the submission; put the `Cc:` line in
  the commit message and let the tooling handle it after it is merged. Sending an unmerged
  patch to stable is a common novice error.

### T.8 — Review etiquette, both directions

**Receiving review.** The dominant failure mode is taking it personally. Kernel review is
terse, direct, and about the code. It is not about you.

| Situation | Response |
|---|---|
| A reviewer is right | "Good catch, fixed in v2." Fix it. Do not argue |
| A reviewer is wrong | Explain, with evidence. "The lock is already held here — see foo() at line 214." Politely, once |
| A reviewer is rude | Answer the technical content, ignore the tone. If it is genuinely abusive, the Code of Conduct committee exists |
| You disagree on design | Present the alternative and its cost. If they still disagree and they are the maintainer, their call |
| **Silence** | See §T.9 |

**Giving review.** You will be asked to do this, and doing it well is how you become known.

- **Review the patch that was sent**, not the one you would have written.
- **Be specific.** "This is wrong" is useless; "this dereferences `dev` before the NULL check
  three lines down" is actionable.
- **Distinguish blocking from advisory.** "This is a bug" versus "nit: I would name this
  differently." Say which.
- **Quote only the relevant hunk.** Trim aggressively. A reply quoting 400 lines to comment
  on one is a failure of consideration.
- **Give the tag you mean.** `Reviewed-by:` is a real statement; do not give it if you only
  skimmed.
- **Reply inline**, below the quoted text, never top-posting.

### T.9 — What to do about silence

The most common experience of a first-time contributor, and the thing nobody explains.

Maintainers are overloaded. A patch that gets no reply usually means "not seen," not
"rejected." The protocol:

1. **Wait a week.** Merge windows, conferences, and holidays are real.
2. **Resend as a ping**, replying to your own patch with a one-line "Gentle ping — any
   thoughts on this?" Do not resend the patch itself yet.
3. **Wait another week**, then resend the series with `RESEND` in the subject:
   `[PATCH RESEND v2 0/3] ...`
4. **Widen the Cc** — the subsystem list, other reviewers who touched the file.
5. **Check you sent it correctly.** Look at your patch on `lore.kernel.org`. Is it
   whitespace-damaged? Did it thread properly? Did it actually arrive?
6. **If a maintainer is genuinely unresponsive for months**, `MAINTAINERS` may list a
   fallback, and `linux-kernel` plus the relevant `TREE` maintainer is the escalation.

**Patience is a skill here.** The median time from first post to merge, for a
non-trivial series, is weeks to months. That is normal, not a problem with you.

### T.10 — The release cycle, and when to send

```
 v6.N released
   │
   ├── MERGE WINDOW (2 weeks) ── maintainers send pull requests to Linus
   │                             NO new patches accepted by subsystem
   │                             maintainers during this period
   ├── -rc1
   ├── -rc2 ... -rc7            ── FIXES ONLY. ~1 week each
   │                             This is when you send bug fixes
   └── v6.(N+1) released
```

The implications for you:

- **New features:** send during -rc. They go into `linux-next`, get tested, and are merged in
  the *next* merge window. Sending a feature during the merge window guarantees it is
  ignored.
- **Bug fixes:** any time, but especially during -rc. Late -rc fixes should be small and
  obviously correct.
- **`linux-next`** is the integration tree. If your patch is applied to a maintainer tree, it
  appears in `linux-next` within a day or two and gets build-tested on many architectures.
  **Watch for the build-bot emails.**
- The bots that will review you before a human does: `kernel test robot` (0-day) for build
  and boot on many configs, `syzbot` for fuzzing, and various static-analysis bots. Fixing
  their complaints before a human looks is free credibility.

---

## 1. Internals

### The documentation you must read

| Path | Contents |
|---|---|
| `Documentation/process/submitting-patches.rst` | **The authoritative guide.** Read it end to end, once, properly |
| `Documentation/process/submit-checklist.rst` | The pre-flight checklist |
| `Documentation/process/coding-style.rst` | Style. Not optional |
| `Documentation/process/development-process.rst` | The release cycle, the trees, the culture |
| `Documentation/process/email-clients.rst` | How to configure yours not to corrupt patches |
| `Documentation/process/5.Posting.rst` | The posting mechanics |
| `Documentation/process/maintainer-*.rst` | Per-subsystem conventions (netdev, tip, KVM, …) |
| `Documentation/process/stable-kernel-rules.rst` | The stable rules (→ Ch. 88) |
| `Documentation/process/code-of-conduct.rst` | And its interpretation document |
| `MAINTAINERS` | Who owns what. `get_maintainer.pl` reads this |

**`Documentation/process/maintainer-netdev.rst` deserves special mention**: netdev has its own
rules (patchwork-driven, 24-hour rule, `net` versus `net-next` targeting) and violating them
is the fastest way to get ignored. Several subsystems have such documents; **check before you
send**.

### Tooling

```bash
# --- Before sending, always ---
./scripts/checkpatch.pl --strict --codespell 0001-*.patch
./scripts/checkpatch.pl -g HEAD~3..HEAD          # check commits directly

# Build tests. Do all of these.
make allmodconfig && make -j$(nproc)             # everything as a module
make allyesconfig && make -j$(nproc)             # everything built in
make W=1 -j$(nproc)                              # extra warnings
make C=1 -j$(nproc)                              # sparse
make CHECK="smatch -p=kernel" C=1                # smatch
make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- defconfig all   # cross-build
make LLVM=1 -j$(nproc)                           # clang

# Documentation build if you touched it:
make htmldocs 2>&1 | grep -i warn

# Devicetree binding check:
make dt_binding_check DT_SCHEMA_FILES=vendor,device.yaml
make dtbs_check

# Coccinelle:
make coccicheck MODE=report M=drivers/foo/
```

**`checkpatch.pl` is advisory, not law.** It has false positives, and blindly obeying it
produces worse code — maintainers will tell you so. But an unclean checkpatch on a first
patch reads as carelessness, so run it and fix what is genuinely wrong.

### Configuring git and your mailer

```bash
git config --global user.name "Your Real Name"
git config --global user.email "you@example.com"
git config --global format.signOff true
git config --global sendemail.smtpServer smtp.example.com
git config --global sendemail.smtpUser you@example.com
git config --global sendemail.smtpEncryption tls
git config --global sendemail.smtpServerPort 587
git config --global sendemail.confirm always
git config --global core.abbrev 12                # kernel convention

# Useful aliases
git config --global alias.fixes "log -1 --abbrev=12 --format='Fixes: %h (\"%s\")'"
git config --global alias.oneline "log --oneline --no-merges"
```

**The single most common mechanical failure is an email client that mangles patches** — line
wrapping, HTML, tab-to-space conversion, base64 encoding. Use `git send-email` or `b4 send`
and the problem does not arise. If you must use a GUI client, read
`Documentation/process/email-clients.rst` and **test by sending a patch to yourself and
applying it with `git am`.**

---

## 2. Practice

### Lab 86.1 — Full setup

```bash
#!/bin/bash
# upstream_setup.sh — everything needed to send your first patch.
set -e

echo "=== 1. Tools ==="
# Debian/Ubuntu
sudo apt install -y git git-email b4 mutt codespell \
	sparse smatch coccinelle python3-ply python3-git \
	gcc-aarch64-linux-gnu clang llvm lld

echo "=== 2. git identity (MUST be your real name) ==="
git config --global user.name "Your Real Name"
git config --global user.email "you@example.com"
git config --global format.signOff true
git config --global core.abbrev 12

echo "=== 3. send-email (example: Gmail with an app password) ==="
git config --global sendemail.smtpServer smtp.gmail.com
git config --global sendemail.smtpServerPort 587
git config --global sendemail.smtpEncryption tls
git config --global sendemail.smtpUser you@gmail.com
git config --global sendemail.confirm always
# Do NOT put the password in the config; use a credential helper or
# ~/.authinfo.gpg. git send-email will prompt otherwise.

echo "=== 4. Clone the trees you need ==="
mkdir -p ~/src && cd ~/src
git clone https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git
cd linux
# The subsystem tree you are targeting -- find it in MAINTAINERS:
git remote add next https://git.kernel.org/pub/scm/linux/kernel/git/next/linux-next.git
git remote add stable https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux-stable.git
git fetch --all

echo "=== 5. Subscribe to the lists you will post to ==="
echo "  https://subspace.kernel.org/vger.kernel.org.html"
echo "  Always at minimum: linux-kernel@vger.kernel.org (high volume --"
echo "  consider reading via lore.kernel.org instead of subscribing)"

echo "=== 6. Verify the pipeline works: send yourself a test patch ==="
cat <<'EOF'
  cd ~/src/linux
  # make a trivial change, commit it, then:
  git format-patch -1 -o /tmp/
  git send-email --to=you@example.com /tmp/0001-*.patch
  # Save the received mail, then:
  git am /tmp/received.mbox
  # If that applies cleanly, your mailer is not corrupting patches.
EOF
```

**Do step 6.** It takes ten minutes and it prevents the most common and most embarrassing
failure mode.

### Lab 86.2 — Find and fix a real bug

```bash
cd ~/src/linux

# --- Sources of real, mergeable work, easiest first ---

# 1. Static-analysis findings nobody has fixed:
make C=1 M=drivers/staging/ 2>&1 | grep -E 'warning|error' | head -40
make W=1 M=drivers/foo/ 2>&1 | grep warning | head

# 2. Coccinelle findings:
make coccicheck MODE=report M=drivers/ 2>&1 | head -40

# 3. syzbot's open bugs, filterable by subsystem:
#    https://syzkaller.appspot.com/upstream
#    Look for ones with a reproducer and no assigned fix.

# 4. The kernel test robot's reports on lore.

# 5. Documentation and comment fixes -- real, welcome, and a good first
#    patch to learn the mechanics with:
codespell --skip='*.patch,.git' Documentation/ drivers/foo/

# 6. TODO files in drivers/staging/ -- explicitly a list of wanted work.
cat drivers/staging/*/TODO
```

Pick one. **Small and real beats large and speculative for a first patch** — the goal is to
learn the mechanics with low stakes, and a genuine one-line fix with a perfect commit
message teaches you everything.

### Lab 86.3 — Produce a clean series

```bash
cd ~/src/linux

# 1. Base on the right tree. For most drivers, Linus's master or the
#    subsystem's -next branch. Check MAINTAINERS for the tree.
git checkout -b my-feature v6.12
# or, for a subsystem:
#   git fetch subsys && git checkout -b my-feature subsys/for-next

# 2. Make commits, one logical change each.
git add -p                      # stage hunks selectively; this is how
                                # you split a change into logical commits
git commit -s                   # -s adds Signed-off-by

# 3. Reshape the history until each commit is right.
git rebase -i v6.12
#   reword  -- fix a commit message
#   edit    -- amend a commit
#   squash  -- combine (for fixups)
#   split   -- `edit`, then `git reset HEAD^`, then commit in pieces

# 4. VERIFY EVERY COMMIT BUILDS. This is not optional.
git rebase v6.12 --exec 'make -j$(nproc) 2>&1 | grep -E "^(ERROR|.*error:)" && exit 1 || true'
#    Or more thoroughly:
git rebase v6.12 --exec 'make -j$(nproc) && ./scripts/checkpatch.pl -g HEAD'

# 5. Check the whole series.
./scripts/checkpatch.pl --strict -g v6.12..HEAD

# 6. Generate and READ the patches. Read them as a reviewer would.
git format-patch --cover-letter -v1 -o /tmp/series/ v6.12..HEAD
less /tmp/series/*.patch
```

Step 4 is the one people skip and it is the one that causes real pain for others. **Make it
a habit.**

### Lab 86.4 — Write a cover letter that earns review

For any series of two or more patches:

```
Subject: [PATCH v2 0/4] drivers/foo: add support for the FooBar-3000

Hi all,

This series adds support for the FooBar-3000 variant to the foo driver.
The hardware is mostly compatible with the existing FooBar-2000 but adds
a second DMA channel and moves the status register.

The first two patches are preparatory refactoring with no functional
change: patch 1 extracts the register layout into a per-variant struct,
and patch 2 factors the channel setup into a helper. Patch 3 fixes a
pre-existing off-by-one in the ring index that I hit while testing;
it is independent and could be applied on its own. Patch 4 adds the
new variant.

I considered handling the register-offset difference with a runtime
check rather than a per-variant struct, but that adds a branch to the
submission fast path and does not extend to the next variant, which
reportedly moves two more registers.

Tested on a FooBar-2000 (no regression) and a FooBar-3000 engineering
sample, both on arm64. Throughput on the -3000 is 1.8 GB/s with both
DMA channels versus 0.95 GB/s with one.

Patch 3 is a fix and is tagged for stable.

v2:
 - split the refactoring out of the feature patch (Alice)
 - fix the locking around the channel allocation (Bob)
 - drop the unrelated whitespace cleanup
 - add the Fixes: tag to patch 3

v1: https://lore.kernel.org/r/20250101000000.12345-1-you@example.com

Your Name (4):
  drivers/foo: extract the register layout per variant
  drivers/foo: factor out channel setup
  drivers/foo: fix off-by-one in the ring index
  drivers/foo: add FooBar-3000 support

 drivers/foo/foo.c | 142 ++++++++++++++++++++++++++++-----------------
 drivers/foo/foo.h |  28 +++++++++
 2 files changed, 118 insertions(+), 52 deletions(-)
```

What makes this one work, and what to copy:

- **The problem first**, then the structure of the series.
- **The rationale for the split** — the reviewer now knows which patches are mechanical and
  which need attention.
- **A rejected alternative, with its cost.** This pre-empts the obvious suggestion.
- **Concrete test evidence**, including "no regression on the old hardware."
- **Numbers.**
- **A flag that patch 3 is independently applicable** — a maintainer can take it immediately.
- **A per-item changelog crediting the reviewers.** People notice.
- **A link to v1.**

### Lab 86.5 — Send it

```bash
cd ~/src/linux

# --- The b4 way (recommended) ---
b4 prep -n foobar3000 -f v6.12
# ... commits ...
b4 prep --edit-cover
b4 prep --auto-to-cc
b4 send --dry-run          # READ THE OUTPUT. Check To:, Cc:, and threading.
b4 send

# --- The git send-email way ---
git format-patch --cover-letter -v2 -o /tmp/series/ v6.12..HEAD
$EDITOR /tmp/series/v2-0000-cover-letter.patch
./scripts/get_maintainer.pl /tmp/series/*.patch

git send-email \
	--to="Maintainer Name <maint@example.com>" \
	--cc="reviewer@example.com" \
	--cc="linux-foo@vger.kernel.org" \
	--cc="linux-kernel@vger.kernel.org" \
	--dry-run \
	/tmp/series/*.patch
# Inspect, then drop --dry-run.

# --- After sending ---
# 1. Find your series on lore and CHECK IT RENDERED CORRECTLY.
#    https://lore.kernel.org/linux-kernel/
# 2. Watch for the kernel test robot. Fix what it finds, immediately.
# 3. When review comes in, collect trailers:
b4 trailers -u
```

### Lab 86.6 — Handle review, and produce v2

```bash
# 1. Reply to every comment. Even "will fix" -- silence looks like you
#    ignored it. Reply inline, below the quote, trimmed.

# 2. Make the changes.
git rebase -i v6.12

# 3. Collect the tags people gave you. Do NOT hand-edit these.
b4 trailers -u

# 4. Update the cover letter changelog, crediting each reviewer.
b4 prep --edit-cover

# 5. Send v2 -- b4 sets In-Reply-To so it threads under v1.
b4 send

# If you are NOT using b4, the threading matters:
git send-email --in-reply-to='<message-id-of-v1-cover>' ...
```

**Rules for v2 and beyond:**
- Change the version: `[PATCH v2]`. Always.
- Carry forward every tag you received, on the patches that did not change.
- **Drop tags from patches that changed substantively**, and say so in the changelog.
- Put the changelog below `---` in the cover letter (and per-patch if a patch has its own).
- Link to the previous version.
- If you disagreed with a comment and did not change something, **say so explicitly** in the
  changelog. Silently ignoring a review comment is the fastest way to lose a reviewer.

### Lab 86.7 — Review somebody else's patch

The other half of the workflow, and the faster path to being trusted.

```bash
# 1. Pick a series in an area you know.
b4 mbox -o /tmp <message-id>
b4 am -o /tmp <message-id>
git checkout -b review-series v6.12
git am /tmp/*.mbx

# 2. Actually build and test it.
make -j$(nproc)
# Boot it in QEMU (Ch. 04). Run the relevant selftests.

# 3. Review each patch:
git log -p v6.12..HEAD
```

**The review checklist:**

- [ ] Does each patch do one thing?
- [ ] Does each patch build? (`git rebase --exec`)
- [ ] Is the commit message explaining *why*?
- [ ] Is there a `Fixes:` tag if it is a fix? Is the SHA right?
- [ ] Should it go to stable?
- [ ] **Error paths**: is every allocation checked? Is every resource released on every path?
- [ ] **Locking**: is the lock held where it should be? Is the order consistent with the rest
      of the subsystem? Can anything sleep under a spinlock?
- [ ] **Concurrency**: what happens if two CPUs enter here?
- [ ] **Integer safety**: overflow, sign, truncation on 32-bit?
- [ ] **User input**: validated? Double-fetch? `__user` handled correctly?
- [ ] **Lifetime**: refcounts balanced? Can the object be freed under us?
- [ ] Does it match the subsystem's existing style and idioms?
- [ ] Is there a test? Should there be?
- [ ] Does it need documentation?

Then reply — inline, trimmed, specific — and give the tag you actually mean.

---

## 3. Mastery drills

1. Get one patch merged into mainline. Any patch. The mechanics are the lesson, and having
   done it once removes all the mystery.

2. Send a five-patch series with a cover letter, respond to review, and get to v3. Document
   every mechanical mistake you made along the way.

3. Review ten patches from a subsystem you know and send substantive comments. Track how many
   of your comments the maintainer agreed with.

4. Take a large, tangled change (yours or someone else's) and split it into a clean series
   where every commit builds. Use `git add -p` and `git rebase -i --edit`. Time yourself.

5. Write a script that verifies every commit in a range builds, passes checkpatch, and boots
   in QEMU. Make it your pre-send gate.

6. Find a bug with `syzkaller`, produce a minimal reproducer, write the fix, and submit it
   with a proper `Fixes:` and `Reported-by:`. This is the complete loop.

7. Read `Documentation/process/submitting-patches.rst` and the three
   `maintainer-*.rst` files for subsystems you might touch. Write the one-page diff between
   the general rules and each subsystem's local rules.

8. Trace a patch from posting to release: find one on lore, follow the thread, find when it
   landed in a maintainer tree, when it hit `linux-next`, and which release contains it.
   Measure the elapsed time at each stage.

9. Deliberately send yourself a patch through a GUI mail client and try to `git am` it.
   Document exactly how it gets corrupted. Then fix the client configuration.

10. Study three patch series that were rejected. Determine why — technical, process, or
    scope — and write what you would have done differently.

11. Write and submit a `MAINTAINERS` entry for something unmaintained that you use. Then
    actually maintain it for three months (→ Ch. 87).

12. Set up a local CI that mirrors the kernel test robot: `allmodconfig`, `allyesconfig`,
    `W=1`, sparse, smatch, and cross-builds for arm64 and riscv, on every commit of your
    branch. Run it before every send.

---

## 4. Further reading

**Mandatory**
- `Documentation/process/submitting-patches.rst` — **read it completely, once, carefully**
- `Documentation/process/submit-checklist.rst`
- `Documentation/process/coding-style.rst`
- `Documentation/process/development-process.rst` — the 8-part guide; the best single
  explanation of how the kernel actually operates
- `Documentation/process/email-clients.rst`
- `Documentation/process/maintainer-*.rst` for your target subsystem

**Tools**
- `b4` documentation: `b4.docs.kernel.org` — read the `prep`/`send`/`trailers` sections
- `lore.kernel.org` — the archive. Learn its search syntax; it is how you find prior art,
  precedent, and whether your idea was already rejected
- `patchwork.kernel.org` — how many maintainers track patch state
- `git send-email` documentation
- `scripts/get_maintainer.pl --help`
- `syzkaller` dashboard: `syzkaller.appspot.com`

**Books and guides**
- Greg Kroah-Hartman's "HOWTO do Linux kernel development" (in `Documentation/process/`)
- Jonathan Corbet's LWN "Development process" articles and the annual development statistics
- *Linux Kernel Development* (Love), ch. 20 — dated on mechanics, sound on culture
- The Linux Foundation's "A Beginner's Guide to Linux Kernel Development" (LFD103) — free

**Culture**
- `Documentation/process/code-of-conduct.rst` and its interpretation
- The kernel maintainer summit reports on LWN, annually — the best window into how decisions
  are actually made
- Read a few weeks of `linux-kernel` on lore before posting. Absorbing the register of the
  conversation is worth more than any guide

→ Next: [87-maintainership.md](87-maintainership.md)
