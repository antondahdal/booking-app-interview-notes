# Git for this plan

Three GitHub repos. Each is its **own** project. Do not mix commits.

| Repo | What | Local folder (typical) |
|---|---|---|
| [LC-Practice](https://github.com/antondahdal/LC-Practice) | Java LC solutions, JUnit, coach rules (`AI-TEACHING.md`) | `Downloads/LeetCode` |
| [booking-app-interview-notes](https://github.com/antondahdal/booking-app-interview-notes) (this repo) | Calendar + mentor notes | `Downloads/booking-app-interview-notes` |
| [SpringBoot-bookingApp](https://github.com/antondahdal/SpringBoot-bookingApp) | The app (Part 2 / Part 3 code) + `AI-TEACHING.md` | the Spring workspace |

## What the words mean

- **Working tree** — files on disk right now.
- **`git status`** — what changed vs the last commit.
- **Stage (`git add`)** — pick files for the next commit.
- **Commit** — a snapshot with a message. Lives only on **your** machine until you push.
- **`origin`** — GitHub.
- **Push** — send your new commits to GitHub so another machine / Cursor can pull them.
- **Pull** — bring GitHub commits onto this machine (`git pull --ff-only` when you have no conflicting local commits).

You always run git **inside that repo’s root** (the folder that contains `.git`). LC commands do nothing to notes, and the other way around.

## When we push

**After Part 3 that weekday**, Anton says push. Then:

1. **LC-Practice** — his solutions + tests + extra coach notes.
2. **This notes repo** — that day’s block in `week-NN.md` + maps.
3. **Spring** — only if Part 2/3 actually changed app code.

Do **not** push from a coaching session until he says so. Do **not** `--force` on `master`/`main`.

## Commands (PowerShell)

From the repo you mean:

```powershell
git status
git add -A
git commit -m "Week 5 Day 1: #19, #206, Ch 4 cache."
git pull --ff-only
git push -u origin HEAD
```

First clone (other PC):

```powershell
git clone https://github.com/antondahdal/LC-Practice.git
git clone https://github.com/antondahdal/booking-app-interview-notes.git
```

If GitHub is ahead and you have local commits, pull **before** push. If it refuses, stop — do not rebase unless Anton asks.

## Do not

- Commit `.env`, passwords, JWT secrets.
- `git push --force` to `master` / `main`.
- Commit `target/` (LC `.gitignore` already skips it).
- Put Spring code in the notes repo or notes in the Spring repo.

## Coach rule

End of **Part 1**: update LC-Practice files locally. End of **Part 3**: Anton says push → LC-Practice **and** this notes repo together. Spring too if app code changed.

**Part 2/3 how to coach:** [ai-spring.md](ai-spring.md) (same as Spring `AI-TEACHING.md`). Notes first. He types. Finish one topic before the next. Part 3 start: list today’s topics, skip Done. Human notes, one closed block per topic.
