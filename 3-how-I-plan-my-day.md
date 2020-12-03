# How I plan my day

Organising your day is one of the things can have a high impact on personal productivity.
I don't want to go too much into motivational aspects or _pro tips_ for
achieving hyper-focus or getting into _the flow_ - there are countless opinions about this already.

What I want to share is a practical, methodological approach that I follow in order to help me
plan my work day and avoid forgetting stuff.
It is not a fancy system, quite the opposite - it is intentionally simple so that it can be
done on a computer, phone, paper and maybe even on a smart-watch.
There are just enough rules to make it effective, but not too many so it doesn't get in my way.

I plan my day using a simple list, with four sections:

1. a title, mentioning the date and tag
    this allows me to filter and sort them on my notes app
1. what was done previously, one per line
    _previously_ means since the last time a note was written, usually the previous day
1. what is to be done today, one per line
1. unstructured "bellow the fold" area for anything worth mentioning
    aka, dumping ground

Here is an example:

```
2020-11-28 #worklog

Previously
[x] define api schema for big-feature-x
[-] implement new api
[ ] deploy to prod

Today
[ ] implement new api
[ ] deploy to prod
[ ] integrate new api with service Alpha30

---

Talk to @jsmith to get credentials for integration with api
```

And these are the rules to make this system worthwhile for me:
- `[x]` for things that are complete
- `[-]` for things that were started but not complete. There might be a reason for this if so, it will be noted "below the fold"
- `[ ]` for things to be done
- At the start of each day:
    1. duplicate the note from the previous day (most note taking apps will have this shortcut on the UI, if not just copy-paste into a new note)
    2. update the date in the `title`
    3. copy everything in the `today` section into `previously`
    4. in the `today` section, remove all completed tasks and write new ones
    5. clean up the `below the fold` section as appropriate

Here we can see what it might look like when we apply these rules to the previous example.
Let's assume we finished the previous day with just one task not done:

```
2020-11-29 #worklog

Previously
[x] implement new api
[x] deploy to prod
[ ] integrate new api with service Alpha30

Today
[ ] integrate new api with service Alpha30
[ ] Celebrate new service being live!

---

```

This helps me keep my day focused.
With a quick glance at this I can recall my thoughts and to continue where I left off.
Just by looking at the shape of a note, I can tell if it was a quiet or a chaotic day or whether
I was able to reach my goals or not.

Following this process, the "worklog" will be like a linked-list of notes - each note mentions
what was planned for the previous day.
I find this to be a good format to look at when doing retrospectives or sharing thoughts with
peers when asked __how's work going?__

The fact that there are no mentions to meeting slots is intentional.
I only write them down when I need preparations for them, otherwise it suffices to have a
notification from the calendar when it's the time.

It's hard to get the right level of detail on each entry.
Too broad and generic and it probably won't be marked as done (`[x]`) for a long time.
If it's too narrow and specific then there will be a plethora of entries, making it hard to grasp
just how much of **meaningful** work there is.

Over time I started getting better at planing my day, summarising and estimating each goal so
that I could reach the end of the day with everything marked as done.
If I carry over more than one entry, then I have failed to understand the complexity of the work
or I was over-confident.
When I mark everything as done and still have plenty of sunlight ahead, then I am most likely lacking
the focus or drive, ultimately being inefficient with my time and energy.
