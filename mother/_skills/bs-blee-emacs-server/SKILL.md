---
name: bs-blee-emacs-server
description: Use when Claude needs the user's running Blee/Emacs --- to read its state (a variable, whether a hook or a reloaded function is in place, *Messages*), to check a Blee/magit setting, or to regenerate COMEEGA dblocks that only Blee can expand (`b:py3:cs:cmnd/classHead`, `b:py3:cs:framework/csuListImportPlus`, `setup.py`'s `b:py3:pypi:setup/*`) with `/bisos/blee/bin/bx-dblock`. Also when `emacsclient` says "can't find socket", or `bx-dblock` reports success but changes nothing. Covers getting the socket path from the user, `emacsclient -s`, read-only versus state-changing use, running `bx-dblock` with `G_emacsServerPid`, its side effects, and the pitfalls (the global dblock regexp, stale sockets, several Emacs instances, reload without `config|main`), and authoring new Blee elisp as COMEEGA `b:elisp:defs/*` dblocks.
---

# Using the User's Running Blee (emacsclient)

Claude can reach the user's running Blee through its server socket with `emacsclient`. That gives
two things: *reading* Emacs state instead of asking the user to type `M-:` checks, and
*regenerating dblocks* that the pure-Python `py-dblock.cs` cannot expand (see `dblock-authoring`).
It is the user's live editor: read freely, change only with agreement.

## 1. Get the socket path --- ask the user

Most reliable: ask the user to evaluate, in the Emacs they mean (`M-:`),

```elisp
(expand-file-name server-name server-socket-dir)
```

and give you the result, e.g. `/tmp/emacs1000/blee-3.1-31.1-1522377`. Blee names its server
`blee-<bleeVer>-<emacsVer>-<pid>`. In a privileged container `/run/user/<uid>` may not work and
the sockets are under `/tmp/emacs<uid>/`. If `(server-running-p)` is nil there, the user starts it
with `M-x server-start`.

Guessing from `ls /tmp/emacs<uid>/ /run/user/<uid>/emacs/` is a fallback only: there may be several
live Emacs instances (not all with the latest reloads) and dead sockets left by exited ones.

## 2. Read state

Always with a timeout (a dead socket hangs) and small printed values:

```bash
S=/tmp/emacs1000/blee-3.1-31.1-1522377
timeout 10 emacsclient -s $S --eval '(emacs-pid)'
timeout 10 emacsclient -s $S --eval "(memq 'some-fn some-hook)"
timeout 10 emacsclient -s $S --eval '(with-current-buffer "*Messages*"
  (buffer-substring-no-properties (max (point-min) (- (point-max) 2000)) (point-max)))'
```

Where a function came from: `(find-lisp-object-file-name 'fn 'defun)`. Truncate `prin1-to-string`
of function bodies. Reading is fine without asking.

## 3. Change state --- only when asked

Evaluating something that changes the user's Emacs (re-running `(bcpg:<pkg>:config|main)` after a
file reload, toggling a mode, visiting files) needs the user's agreement. Never save or kill the
user's buffers, and never leave a mode changed: restore what you changed (and say so).

## 4. Regenerate dblocks

The user's keys (global `F7 d` map) and the functions behind them --- Claude calls the same
functions through emacsclient:

| Keys     | Function                          | Does                                                    |
|----------|-----------------------------------|---------------------------------------------------------|
| `F7 d d` | `org-dblock-update-re-major-mode` | set the dblock regexp from the major mode, update the one dblock at point |
| `F7 d b` | `org-dblock-update-buffer-bx`     | re-run the major mode, update *all* dblocks in the buffer |
| `F7 d u` | `org-dblock-bx-blank-this`        | blank one dblock                                        |
| `F7 d c` | `org-dblock-bx-blank-buffer`      | blank all                                               |

They work in every COMEEGA mode (elisp, bash, python, latex). `F7 d d` always works because it
sets the regexp first. `org-dblock-start-re` / `-end-re` are *globals*: `F7 d b` only gets the
right one through a mode hook (elisp, LaTeX, shell, HTML, org have one in `poly-dblock.el`; Python
none), so in a Python buffer it uses whatever was set last. So Claude always sets the regexp from
the target file's mode first --- `(org-dblock-mode-comment-regexp-bx)` in that buffer --- then
updates.

Ask before regenerating a real file: it acts in the user's Emacs and writes the file.

*The file is open in the user's Emacs* (`(get-file-buffer "<file>")` non-nil, and not modified):
work in that buffer and save it; do not kill it.

```bash
timeout 90 emacsclient -s $S --eval "(with-current-buffer (get-file-buffer \"<file>\")
  (revert-buffer t t t)                 ; pick up Claude's edits on disk (only if unmodified!)
  (org-dblock-mode-comment-regexp-bx)   ; as F7 d d does
  (list (condition-case e (progn (org-dblock-update-buffer-bx) 'ok) (error (list 'ERR e)))
        (progn (when (buffer-modified-p) (save-buffer)) (buffer-modified-p))))"
git -C <repo> diff -- <file>
```

If the buffer is *modified*, stop and ask: the user has unsaved work there.

*The file is not open:* `bx-dblock` (= `F7 d b` from a shell) finds the Emacs by the pid in
`G_emacsServerPid` (socket `*<pid>*` under `/run/user/<uid>/emacs/`, then `/tmp/emacs<uid>/`, then
one named `server`); the value Claude's shell inherited is usually stale, so pass the pid from the
socket name. Set the regexp first, from a buffer in the file's mode, as above.

```bash
G_emacsServerPid=1522377 /bisos/blee/bin/bx-dblock -i dblockUpdateFiles <file>
```

`org-dblock-update-file-bx` (what `bx-dblock` calls) visits the file, updates, `save-buffer`,
`kill-buffer` --- so it would kill a buffer the user has open --- and turns
`global-auto-revert-mode` on and then *off*: read it before, restore it after with
`(global-auto-revert-mode 1)`.

Either way:
- *All* dblocks of the file are updated, not only yours: stale file-level blocks (proclamations,
  authors, top controls, folding sections, package-adoption defuns) are refreshed to the current
  templates. Review the whole diff and say so to the user.
- Errors inside the update are swallowed and the exit status is 0: only `git diff` tells.

Verified 2026-10-27: `bx-dblock` on a scratch copy of `containerProc-seed.cs` (first run found no
block --- the regexp was the elisp one --- the second, after setting it, regenerated a deleted
line); and the open-buffer form on `bcpg:magitPlus.el`.

## 5. Authoring Blee elisp the COMEEGA way

New definitions in Blee elisp are dblock headers, not plain `defun`s (see `comeega-authorship`).
Write the header and the part after `;;;#+END:` (arguments, the `" #+begin_org ** DocStr: ..."`
docstring, the body); regenerate to get the org heading and the `(cl-defun name (` line:

```elisp
;;;#+BEGIN:  b:elisp:defs/cl-defun :defName "pkg:area/verb" :advice ()
;;;#+END:
                     )
   " #+begin_org
** DocStr: One line on what it does.
#+end_org "
  ...body...)
```

`b:elisp:defs/cl-defun` (the user's preference) or `b:elisp:defs/defun`; group related definitions
under `;;;#+BEGIN: blee:bxPanel:foldingSection :outLevel 1 :title "..." :extraInfo "..."`.
`b:elisp:defs/defvar` exists but nothing uses it yet; a plain `defvar` inside the section is fine.
A `bcpg:` package's `setq` / `add-hook` forms go in its `bcpg:<pkg>:config|main`.

## Pitfalls

- *A file reload does not re-run `bcpg:<pkg>:config|main`*: the `setq` / `add-hook` forms of a Blee
  package adoption live there. After the user reloads such a file, they (or you, when asked) run
  `(bcpg:<pkg>:config|main)` once; `add-hook` does not duplicate.
- *Several Emacs instances:* check which one has the setup (`(emacs-pid)`, a variable you expect).
- *Dead sockets:* `timeout` every call; a socket that does not answer in 5--10 s is stale.
- *Magit's commit buffer* is marked unmodified after `git-commit-setup-hook`; text inserted by a hook
  is not saved unless the buffer is marked modified before finishing (see
  `bcpg:magitPlus.el`, `bap:magit:claudeCommitMsg/offer`).

## References

- `/bisos/blee/bin/bx-dblock`, `/bisos/blee/bin/bleeLib.sh` (`vis_thisEmacsClient`,
  `vis_emacsServerSocketFromEnv`).
- `/bisos/blee/env3/legacy/bx-dblock-lib.el` (`org-dblock-update-file-bx`,
  `org-dblock-update-buffer-bx`, `org-dblock-mode-comment-regexp-bx`);
  `/bisos/git/bxRepos/blee/poly-dblock/poly-dblock.el` (the mode hooks).
- Skill `dblock-authoring` (which dblocks need Blee).
