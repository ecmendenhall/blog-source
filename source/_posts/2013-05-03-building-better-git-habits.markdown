---
layout: post
title: "Building better git habits"
date: 2013-05-03 01:00
comments: true
categories: git 
---


Like many young whippersnapper kids these days, I started using GitHub before I really understood git. The benefits of version control were obvious right away, but I stuck to the very simplest git commands for a long time, and many of them turned into habits.

This week, I've been working on making cleaner commits and writing better messages. Composing [good commit messages](http://tbaggery.com/2008/04/19/a-note-about-git-commit-messages.html) is the best git habit of all, and I've found two simple substitutions that nudge me towards more thoughtful commits.

If you're used to `git add -u` or `git add -a`, try:

    git add -p

Instead of committing everything at once, this will walk you through the results of `git diff`, offering the option to stage each chunk of changed code:

    @@ -34,5 +35,45 @@ public class GameTree {
    
    +
    +        public List<Node> getLeaves() {
    +            List<Node> leaves = new ArrayList<Node>();
    +
    +            for (Node child : children) {
    +                if (child.children.isEmpty()) {
    +                    leaves.add(child);
    +                } else {
    +                    leaves.addAll(child.getLeaves());
    +                }
    +            }
    +            return leaves;
    +        }

    Stage this hunk [y,n,q,a,d,/,K,g,e,?]?

(You can find the alphabet soup of staging options [here](http://git-scm.com/docs/git-add)). This is easier to read than a full diff, requires a careful review of each change in context, and makes composing small atomic commits much easier.

Then, if you're used to `git commit -m <message>`, try:

    git commit -F <filename>

Instead of pulling a commit message from the command line or opening an editor, this uses an existing file as your commit message. But the real trick here is to keep an editor open as you page through the previous command, and compose your commit message chunk by chunk.

There are [many](http://www-cs-students.stanford.edu/~blynn/gitmagic/), [many](http://sethrobertson.github.io/GitBestPractices/) [more](http://reinh.com/blog/2009/03/02/a-git-workflow-for-agile-teams.html) git workflows, including ways to commit with reckless abandon and clean up later with `git rebase`, but these two simple changes have helped me make much better commits just by switching a few command line flags.
