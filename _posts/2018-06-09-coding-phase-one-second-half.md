---
layout: post
title: GSoC' 2018 Coding Phase 1 Week 3 & 4 report
categories: gsoc
description: This is biweekly report of week 3 & 4 in coding phase 1.
keywords: gsoc, report
---

The end of coding phase 1 is coming. Like the previous post, I will briefly explain
what I've done in the past two weeks and lessons I learned.

### Set up meta_review app

The [Pull Request](https://github.com/coala/community/pull/134)
was created at the end of week 2. I got it merged in week 3. My mentor told
me to prettify it a bit, as it currently has no 'UI'. The only problem is that
the website itself has no UI at all. Another GSoC student is going to develop
a bit UI for that, and he has began. To make the style consistent, this part
will be saved for the third coding phase.

### Implement meta-review scoring & ranking system

The [Pull Request](https://github.com/coala/community/pull/143)
is still pending review. Hope I could get it merged before coding phase
two starts. This is the largest pull request during the past two weeks,
approx. 700 lines. The basic idea is to fetch `issues.json` from gh-board
repo, process it and store relevant information in `communnity` database.
Some information will be reused by another GSoC project,
[Newcomer metrics and Gamification](https://github.com/coala/cEPs/blob/master/cEP-0020.md).
Also, a ranking list will be displayed on community website. Although this
PR hasn't been merged yet, you could take a look at
[preview](https://deploy-preview-143--coala-community.netlify.com/meta-review/).

I finished it at the end of the third week. Then I thought a week would be
undoubtedly sufficient to get proper reviews and get it merged. Who knows,
I spent several days struggling with CI.

At first, my PR could not pass coala. coala always got stuck somewhere.
I ran it locally, the same thing happened. Later I found out coala got
stuck on a specific file: `meta_review.json`, which is where I dump Django
database. Now things became clear - `AnnotationBear` always gets stuck on
large files. I filed an issue
[AnnotationBear gets stuck on large files](https://github.com/coala/coala-bears/issues/2522)
to report that. The (temporary) solution is extremely easy: just run `coala`
before my script runs. Although this is an adhoc solution, it makes sense:
we shouldn't check files generated by custom scripts, as they might introduce
resources out of our control.

Thank god, I made Travis CI green. Netlify, however, still failed. It always
got stuck at some point, like this:

```
6:51:34 PM: + python manage.py run_meta_review_system
7:20:47 PM: Build exceeded maximum allowed runtime
```

This is quite weird, as Travis always finishes build in 10 minutes, and
they are running the same code & build script! I added a lot of logs, but
nothing was printed out on time. It looked like Netlify just got stuck at
some point. I decided to print out time in the debug messages. Then the
strange thing happened:

```
10:49:16 AM: before some heavy functionality 2018-06-08 10:28:06.843622+08:00 (Note the TIME)
10:49:16 AM: after some heavy functionality 2018-06-08 10:47:29.461540+08:00
```

Take a look at the above log. The time on the left: `10:49:16 AM` is printed
out by Netlify deploy process. The time on the right: `10:28:06` is printed out
by my program. Interesting, huh? The first line wasn't printed out on time - it
was delayed for 20 minutes! Well, it seemed that stdout was buffered, probably
by Netlify. Then I use `flush=True` attribute in print statement. No surprise,
log is printed on time.

But what's the real reason for that? I asked Netlify community, but got no
response. My assumption is that Netlify redirects stdout to somewhere else,
and buffers it. Since Netlify is not open source, cannot do further
investigation on it. Anyway, this is not related to my work, just an
interesting phenomenon ;)

Then debugging became smoother - at least I could use log to debug. I found out
the bottleneck: either IO or CPU speed on Netlify was sooooooo slow, compared to
that on Travis. My program did a lot of database operations, which took
Travis 1 minute and Netlify more than 10 minutes. So my focus shifted to how
to improve speed of Django database operations.

I was using the naive way to create Django objects:

```python
for key, comment in self.comments.items():
    c, created = Comment.objects.get_or_create(
        id=comment['id']
    )
```

This is quite a common approach when we want to create if not existed and
get if existed. This approach, however, is quite slow. Django translates it
to independent SQL queries. When there are thousands of `comments`, there
are thousands of SQL queries. This is a waste of resources.

Luckily, since version 1.4, Django supports
[bulk_create](https://docs.djangoproject.com/en/2.0/ref/models/querysets/#bulk-create).
The above code becomes as follows:

```python
old_comments = Comment.objects.all()
old_commments_set = set()
for old_comment in old_comments:
    old_commments_set.add(old_comment.id)

new_comments = []
for key, comment in self.comments.items():
    # if it is an old comment, we skip it
    if not comment['id'] in old_commments_set:
        new_comments.append(
            Comment(id=comment['id'])
        )

Comment.objects.bulk_create(new_comments)
```

Also, I was using the naive ways to dump objects into database:

```python
for key, participant in self.participants.items():
    participant.save()
```

There is no `bulk_save` method, but we could use bulk delete and
then bulk create again:

```python
Comment.objects.all().delete()
Comment.objects.bulk_create(comments)
```

Done! Now the whole process only takes one minute to run, even on Netlify.
Finally I get all CI pass. In all, the task is more complicated then I
expected. The good news is I learned new things about Django ;)

### Get test suite working

I get the [Pull Request](https://github.com/coala/gh-board/pull/42) merged,
which makes test suite working again on coala/gh-board repo. It took me some
time to investigate what caused failure of CI. Luckily, I managed to solve it.
I also improved the test process a bit. The original author uses a magic number
42 as exit code to judge whether test passes. Although 42 is an awesome number
for sure, it is not elegant at all. I abandoned the use of magic code and made
test process more standard. As discussed
[here](https://github.com/coala/gh-board/pull/42#discussion_r193952422),
a pull request to upstream repos will be created during coding phase two.

### Documentation

Luckily I still have two days for documentation. I will create a pull request
shortly after.

### Others

During the past two weeks, I also reviewed some pull requests.
I also merged a pull request
[package.json: Add linting on script & test](https://github.com/coala/gh-board/pull/46)

### Conclusion

From my own and other GSoC students' experience, I would say the biggest
challenge for many of us is not the task itself, but the lack of ability to
estimate tasks. This includes estimation of difficulty and time. It is often
the case that we are too optimistic. It is also due to lack of experience.
Novice programmers often feel that they don't know their capacities. They
don't know how many lines of code they could write every day, and how much
time they will spend on debugging and refactoring. Even I have done an
internship before, I have no idea what my capacity is. A good programmer
must be a good project manager at the same time, this is what I've learned
from GSoC experience.

There is one lucky thing: coala has strict requirements on code review and
continuous integration, which helps us make sure the quality of our project
doesn't become out of control. To write readable code with high quality may
take a bit more time at the beginning, but saves many efforts in the
future.