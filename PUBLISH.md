# How to publish this repo to GitHub

## 1. Before you publish
- These cover **Starting Point** and **retired** machines only (allowed by HTB).
- **Never** add a writeup for an **active** machine.
- **Remove all flag values.** Keep proof-of-access (prompts, `id`, `whoami`), not flags.
- Add your own screenshots under `images/<box>/` (see `images/README.md` for the shot-list).

## 2. Create the repo
Website: github.com > New repository > `htb-writeups` > Public > Create (do not add a
README; this repo already has one).

Or with the GitHub CLI:
```bash
gh repo create htb-writeups --public --source=. --remote=origin
```

## 3. Push with ONE COMMIT PER BOX (greener contribution graph)
A single bulk commit looks thin. Committing box-by-box shows steady, self-driven work,
which recruiters do glance at.
```bash
cd htb-writeups
git init && git branch -M main
git add README.md WRITEUP_TEMPLATE.md PUBLISH.md .gitignore images/README.md
git commit -m "Portfolio: README, template, publish guide, image shot-list"
# then commit each box on its own:
for f in 01-starting-point/tier-0/*.md 01-starting-point/tier-1/*.md \
         01-starting-point/tier-2/*.md 02-retired/*.md; do
  git add "$f"; git commit -m "Add writeup: $(basename "$f" .md)"
done
git remote add origin https://github.com/<your-username>/htb-writeups.git
git push -u origin main
```

## 4. Put the link everywhere
- Resume: `GitHub: github.com/<your-username>/htb-writeups`
- LinkedIn featured section + HTB bio.
- When you post a new writeup, share the single box link (e.g. the Forest or Unified
  writeup) as a LinkedIn post. Visibility compounds.

## 5. Keep it alive
- Add screenshots over time (start with the forest/ shot-list).
- Add each new box as you root it, one commit each.
- Lead the README's flagship line with your strongest chains (Forest, Unified).
