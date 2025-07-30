---
layout: distill
title: Learning Git
description: Getting over my fear of git
tags: code learning
giscus_comments: true
date: 2025-07-30
featured: true
mermaid:
  enabled: true
  zoomable: true
code_diff: true
map: true
chart:
  chartjs: true
  echarts: true
  vega_lite: true
tikzjax: true
typograms: true

# Optionally, you can add a table of contents to your post.
# NOTES:
#   - make sure that TOC names match the actual section names
#     for hyperlinks within the post to work correctly.
#   - we may want to automate TOC generation in the future using
#     jekyll-toc plugin (https://github.com/toshimaru/jekyll-toc).
toc:
  - name: How I Started
  - name: The Basics
  - name: Things to Know
  - name: Undoing Things
  - name: Branching
  - name: The Powerful Git Rebase
  - name: Flags I Like
  - name: Resources
  # if a section has subsections, you can add them as follows:
  # subsections:
  #   - name: Example Child Subsection 1
  #   - name: Example Child Subsection 2
---

## How I Started

I remember before starting college, `Git` was a scary word to me. Mainly because I didn't understand what it was and why it was important. All the `Git` commands seemed strange and obscure. I would be even more confused when people brought up Github. What's the difference between `Git` and Github? Aren't they the same thing? Why is one in the terminal and the other a website? I had lots of unanswered questions.

During a meeting with a University of Chicago grad who was working on Github developer docs, I told him how daunting `Git` felt to me. I asked him how he was able to learn `Git`, and whether there was a class out there or a specific resource to become familiar with it. He responded that I shouldn't worry too much, and that eventually I would pick it up as a student in college. I was a little skeptical.

But he turned out to be right.

---

## The Basics

One of the first things I learned through my intro CS class was how to use `Git` to clone, commit, and push my local work to a remote repository hosted on Github. This helped me understand the difference between `Git` and Github. `Git` is the essential version control system that lives on your computer, and Github is simply a service built around `Git` that hosts your code and helps you share your `Git` managed repositories. We could do without Github, but we cannot do without `Git` - that's the tool you should know how to use.

To get a local git repository started, you can clone (copy) an existing remote repository:

```shell
git clone https://github.com/davdma/davdma.github.io.git
```

You can see the modified states of your files using `git status`. Then you could commit changes by running:

1.  ```shell
    git add file.py
    ```
2.  ```shell
    git commit -m "my first commit"
    ```

To upload to the remote repository you just run:

```shell
git push
```

To pull in changes from the repo (when professors would add files we needed), run:

```shell
git pull
```

These commands were all that was needed for class.

Note: If you wanted to start a new `git` repository locally on your computer instead of cloning an existing one, you can just call `git init` in the project directory. You can always add the remote later using the `git remote` command.

---

## Things to Know

Some things that helped me better wrap my head around what was going on in `Git`:

- Files are either tracked or untracked. Tracked means that `Git` knows about it, so you will want to make sure your most important pieces of code are being tracked. Untracked files are just everything else in the directory (if you lose untracked files, `Git` cannot recover them for you). You can check the status of your files using `git status`.
- Terms you will hear a lot is **index** and **working tree**:
  - **Staging area (Index)** is the temporary area where you prepare changes to be included in the next commit. It is the bridge between the working tree and the repository's history. Each time you run `git add` it sends a snapshot of your working tree to put in the index. The purpose of the index is that it allows you to selectively stage changes for including in the next commit without doing it all at once.
  - The **working directory (Working Tree)** is the directory on your file system where the project files live. The working tree contains your tracked and untracked files and any modifications you make. The changes you make in your working directory or tree are not tracked until explicitly added to the index.
- The mental model I use is `working tree -> index -> git history`. The first two is bridged with `git add`, and the second two is bridged with `git commit`.
- When you are working on a remote repository, it is important to know that there is a **local representation** of the remote repository that lives on your computer (if your local branch is `main` then it is typically referenced by `origin/main`). Each time you `git pull` it actually runs two separate commands, `git fetch` and `git merge` behind the scenes. `git fetch` (kind of like a download) synchronizes your local remote representation with the source remote representation. To get the updates in your local files the `git merge` is then done between the new local remote branch and your local branch.

---

## Undoing Things

This is one of the things that stumped me for a while. How do I undo a particular commit? How do I revert a specific file rather than my entire tracked working tree?

While trying to revert just a single file to a previous state, I learned some intricacies about `git reset` as opposed to `git restore`. They sound similar, but are actually doing different things. I learned that `git reset` moves the `HEAD` while `git restore` does not, it only modifies your working directory. This is why `git restore` is a safer operation.

---

## Branching

Branching is the most powerful feature in `Git`, and I wish I learned about why sooner. During my software development class, I was working alongside multiple teams of students developing new features for the codebase. When you have many people all working on different versions of the code, things can get hairy quick. This is why it is essential to work with branching.

---

## The Powerful Git Rebase

During a talk by a software developer at Slack, he recommended that students learn to use the interactive git rebase `git rebase -i`. Apparently nobody knows how to use it, but it's a superpower. This piqued my interest, so I started learning more about rebasing.

---

## Flags I Like

- Typically for large repos with lots of files that I don't want to store on Github, I end up with many untracked files. This can clutter the `git status` output. Using the flag `git status -uno` however skips displaying untracked files, making it easier to read.
- Sometimes you want to just add and commit everything you modified. Save yourself some time with `git commit -a` flag.
- For cleaner commit history when incorporating remote changes, fetch and rebase instead of the default fetch and merge with `git pull --rebase` option.
- For reading the commit logs, show diffs with `git log -p`. Abbreviated stats (lines modified etc.) can be shown with `--stat` and the ASCII graph with `--graph`. Can also limit to specific files with `git log -- path/to/file`.
- For reading diffs with more context, use the `-U` flag, add the number of lines you want around the diffs with an integer after the flag, for example, `git diff -U8` for 8 lines of context instead of the default 3.

---

## Resources

Here are some useful resources I consulted (and I still often go back to) on my journey to learning git:

- [The UChicago Student Resource Guide](https://uchicago-cs.github.io/student-resource-guide/tutorials/git-local.html)
- [Interactive Git Tutorial](https://learngitbranching.js.org/)
- [Pro Git book](https://git-scm.com/book/en/v2)
