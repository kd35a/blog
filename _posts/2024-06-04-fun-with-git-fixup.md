---
title: "Reblog: Fun with Git: fixup"
tags: ["git", "fixup"]
---
_I originally wrote this blog post for my employer AFRYs blog (dead link:
https://buitsyd.com/blog/2023/02/27/fun-with-git-fixup). This blog has been
removed, so I decided to reblog my old post here._

# Fun with Git: fixup
Git is a tool full of small features that can be more or less hard to find, and
one of these are `fixup`. Let us first quickly check how to use it and what it
is for, before we dive down to see _why and how it works_.

## Usage

`fixup` is a special kind of commit that tells git you want to _fix up_ an older
commit where you missed something. `git rebase` can then automatically handle
these _fix ups_ if instructed to.

Say you have a commit tree that looks like this
```
* b5cc884 - (master) Add foobar to hello script (34 hours ago) <Fredrik Strandin>
| * e639e91 - (HEAD -> master.add_calc) Add multiplication to calc script (34 hours ago) <Fredrik Strandin>
| * 2497c9f - Add calc script (34 hours ago) <Fredrik Strandin>
|/
* 05230cf - Initial hello script (34 hours ago) <Fredrik Strandin>
```

_Tip_: To view your commit tree like above, add an alias like [`glola`](https://github.com/ohmyzsh/ohmyzsh/blob/a7d910c3a61d8599f748a8ddae59ecdd9424022a/plugins/git/git.plugin.zsh#L223).

But now you realise you made a typo in `2497c9f` - hate when it happens! Let's
first correct the mistake, and now we want to commit this patch to git. First,
we commit it as a `fixup` commit:
```
$ git add calc.py
$ git commit --fixup=2497c9f
$ glola
[..]
* 41e5ebb - (HEAD -> master.add_calc) fixup! Add calc script (2 minutes ago) <Fredrik Strandin>
[..]
```

At this point we can continue our work, and do some more commits. When we feel
ready to push, we first do an _interactive autosquash_ rebase:
```
$ git rebase --interactive --autosquash master
```
which opens your editor with the following:
```
pick 2497c9f Add calc script
fixup 41e5ebb fixup! Add calc script
pick e639e91 Add multiplication to calc script
```

Note how `git rebase` has re-ordered your commits, and put a `fixup` command in
front of `41e5ebb` instead of the regular `pick` command. Lets save this and
quit the editor and see what happens:
```
* d929c35 - (HEAD -> master.add_calc) Add multiplication to calc script (2 days ago) <Fredrik Strandin>
* 2c32671 - Add calc script (2 days ago) <Fredrik Strandin>
* b5cc884 - (master) Add foobar to hello script (2 days ago) <Fredrik Strandin>
* 05230cf - Initial hello script (2 days ago) <Fredrik Strandin>
```
Well look at that! If you now do `git show 2c32671` you will see that the commit
now contains what you initially wanted without your typo.

Now that we know what `fixup` is and how to use it, lets look a little closer at
how it works.

## How does it work?

### Naive idea
An initial idea would be that `git` adds metadata to the commit object, to link
the fixup commit with the commit it's intended to fix. Let's take a look inside
the git object of the commit:
```
$ git cat-file commit 41e5ebb
tree 2d992cbdf3a99033a69af510d51378120a98d68d
parent e639e91d72f34cd3054e1cf7b4b76f6688f067e3
author Fredrik Strandin <fredrik@strandin.name> 1666290763 +0200
committer Fredrik Strandin <fredrik@strandin.name> 1666290763 +0200

fixup! Add calc script
```

Well, that wasn't much? It points to a file/folder `tree`, and to its `parent`
commit, the usual author and commit info. And lastly we have the commit message.

_Why wouldn't it work having a header, like `fixup 123abc4`?_ Commits can be
rebased and rearranged, or in other ways get their hashes changed. Adding a
header would add a lot of bookkeeping to every command that modifies the commit
tree. So, this would be a bit tedious to implement in reality.

### Read the source, Luke

_Note: I will refer to `git` version `2.39.2` source code in this post, which is the
latest release when writing this._

In the source code for `git`, the builtin commands have their implementations
in the `builtin/` directory, so lets open [`builtin/rebase.c`](https://github.com/git/git/blob/v2.39.2/builtin/rebase.c).

After a bit of argument parsing etc., we end up in the [`do_interactive_rebase`](https://github.com/git/git/blob/v2.39.2/builtin/rebase.c#L257-L314)
function. Here is the listing of this function, abbreviated for emphasis on
important parts, and with some comments added by me:
```c
// builtin/rebase.c
static int do_interactive_rebase(struct rebase_options *opts, unsigned flags)
{
        // ...
        // This creates an empty string buffer
        struct todo_list todo_list = TODO_LIST_INIT;
        // ...

        // generate a "sequencer script" in the string buffer
        ret = sequencer_make_script(the_repository, &todo_list.buf,
                                    make_script_args.nr, make_script_args.v,
                                    flags);

        if (ret)
                error(_("could not generate todo list"));
        else {
                // ...
                // and finally, execute the "sequencer script"
                ret = complete_action(the_repository, &replay, flags,
                        shortrevisions, opts->onto_name, opts->onto,
                        &opts->orig_head->object.oid, &commands,
                        opts->autosquash, opts->update_refs, &todo_list);
                // ...
        }

        todo_list_release(&todo_list);
}
```

Now we will end up in internal implementation parts of git, let's look at
[`sequencer_make_script` in `sequencer.c`](https://github.com/git/git/blob/v2.39.2/sequencer.c#L5649-L5725).

```c
// sequencer.c
int sequencer_make_script(struct repository *r, struct strbuf *out, int argc,
                          const char **argv, unsigned flags)
{
        // ...
        while ((commit = get_revision(&revs))) {
                // ...
                strbuf_addf(out, "%s %s ", insn,
                            oid_to_hex(&commit->object.oid));
                pretty_print_commit(&pp, commit, out);
                if (is_empty)
                        strbuf_addf(out, " %c empty", comment_line_char);
                strbuf_addch(out, '\n');
        }
        // ...
}
```

This populates a simple set of commands/todo list, where we just pick every
commit. The above example would look something like:
```
pick 2497c9f Add calc script
pick e639e91 Add multiplication to calc script
pick 41e5ebb fixup! Add calc script
```

After this, we want to execute the generated commands, which takes us to
[`complete_action` in `sequencer.c`](https://github.com/git/git/blob/v2.39.2/sequencer.c#L6047-L6143).

```c
// sequencer.c
int complete_action(struct repository *r, struct replay_opts *opts, unsigned flags,
                    const char *shortrevisions, const char *onto_name,
                    struct commit *onto, const struct object_id *orig_head,
                    struct string_list *commands, unsigned autosquash,
                    unsigned update_refs,
                    struct todo_list *todo_list)
{
        // ...

        // Lets rearrange our todo list
        if (autosquash && todo_list_rearrange_squash(todo_list))
                return -1;

        // ...

        // This will allow us to edit the command list, as show under Usage
        res = edit_todo_list(r, todo_list, &new_todo, shortrevisions,
                             shortonto, flags);
        // ...
}
```

And now, we're up for the actual rearrangement of the todo list, in
[`todo_list_rearrange_squash` in `sequencer.c`](https://github.com/git/git/blob/v2.39.2/sequencer.c#L6172-L6327)
```c
// sequencer.c
int todo_list_rearrange_squash(struct todo_list *todo_list)
{
        struct hashmap subject2item;
        // ...
        /*
         * The hashmap maps onelines to the respective todo list index.
         *
         * If any items need to be rearranged, the next[i] value will indicate
         * which item was moved directly after the i'th.
         *
         * In that case, last[i] will indicate the index of the latest item to
         * be moved to appear after the i'th.
         */
        hashmap_init(&subject2item, subject2item_cmp, NULL, todo_list->nr);
        // ...
        for (i = 0; i < todo_list->nr; i++) {
                // ...
                // Remove the "fixup!/squash!/amend!" prefix
                if (skip_fixupish(subject, &p)) {
                        // ...

                        // and find out what entry the commit message is refering to:
                        entry = hashmap_get_entry_from_hash(&subject2item,
                                                strhash(p), p,
                                                struct subject2item_entry,
                                                entry);
                        if (entry)
                                /* found by title */
                                i2 = entry->i;
                        else if (!strchr(p, ' ') &&
                                 (commit2 =
                                  lookup_commit_reference_by_name(p)) &&
                                 *commit_todo_item_at(&commit_todo, commit2))
                                /* found by commit name */
                                i2 = *commit_todo_item_at(&commit_todo, commit2)
                                        - todo_list->items;
                        else {
                                /* copy can be a prefix of the commit subject */
                                for (i2 = 0; i2 < i; i2++)
                                        if (subjects[i2] &&
                                            starts_with(subjects[i2], p))
                                                break;
                                if (i2 == i)
                                        i2 = -1;
                        }
                }
                // Since we found a reference, lets rearrange:
                if (i2 >= 0) {
                        rearranged = 1;
                        // what kind of fixup command is it?
                        if (starts_with(subject, "fixup!")) {
                                todo_list->items[i].command = TODO_FIXUP;
                        } else if (starts_with(subject, "amend!")) {
                                todo_list->items[i].command = TODO_FIXUP;
                                todo_list->items[i].flags = TODO_REPLACE_FIXUP_MSG;
                        } else {
                                todo_list->items[i].command = TODO_SQUASH;
                        }
                        // rearrange the todo list:
                        if (tail[i2] < 0) {
                                next[i] = next[i2];
                                next[i2] = i;
                        } else {
                                next[i] = next[tail[i2]];
                                next[tail[i2]] = i;
                        }
                        tail[i2] = i;
                }
        // ...
}
```

After this, we obviously need to actually _do_ the squashing/fixups, according
to the todo list we've created. But that is left as an exercise to the reader 😉

### Drawbacks

If you have multiple commits with the same message (but you don't, you obviously
write descriptive and well-thought-out messages, right?), there is a risk that
the sequencer will rearrange the fixup incorrectly. This will result in merge
conflicts when executing the rebase:
```
Auto-merging calc.py
CONFLICT (content): Merge conflict in calc.py
error: could not apply faecb7d... fixup! Add some calc
```
