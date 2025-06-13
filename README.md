# forcing_precommit

Or *"How I solved the need to selectively force pre-commit"*

This is a description of how I solved a specific situation for me and my team. It may not be the best solution for you. It may not even be the best solution for me, so you are more than welcome to provide feedback or suggestions (via issues or pull requests).

We wanted to make sure that no team member pushed big files, using git-lfs instead. This can be done with [pre-commit](https://pre-commit.com/), and we also planned to use pre-commit for other functions.

The problem is that, for obvious security reasons, pre-commit does not *magically* run after you clone a repository. You must remember run `pre-commit install` to set it up everyt time after cloning. Even if you have clear policies for that, people may sometimes forget to do that, and all you need is single mistake to *contaminate* the repository, needing cumbersome operations to fix it (e.g: [BFG](https://rtyley.github.io/bfg-repo-cleaner/))

You can edit your local environment to avoid this, so that `pre-commit install` is run automatically for you. And this is fine for our internal repositories (we use GHE), because we trust each other. But then you are opening yourself to attackers every time you clone from a public uncontrolled repository.

So ideally we would like to have something that automatically has pre-commit activated whenever you clone from _github.mycompany.com_ but has it deactivated when you clone from somewhere else, like _github.com_

I was playing with "`[includeIf "hasconfig:remote.*.url ...`" in git config to modify `init.templatedir` conditionally , but I wasn't able to make it work. If you know, please let me know

So I came up with this second-best solution:

First we create the folder for the  custom hook: `mkdir -p $HOME/.git-template/hooks`
 
In that folder we create the `pre-commit`file with the following content:

```bash
#!/usr/bin/env bash
# If we are already called legacy, then pre-commit is already installed
if [[ $0 != *legacy ]]; then
    if [ -f .pre-commit-config.yaml ]; then
        echo 'pre-commit configuration detected, but pre-commit install was never run' 1>&2
        echo "please run 'pre-commit install' to install pre-commit hooks" 1>&2
        echo "or delete||edit .git/hooks/pre-commit to disable this test" 1>&2
        exit 1
    fi
fi
# We want to include in commits any .gitignore ot .gitattribute file that is not yet committed
files=$(git ls-files --others --exclude-standard --modified | grep -E "(.*/)?\.gitattributes|(.*/)?\.gitignore")
if [[ -n "$files" ]]; then
  echo "Detected modified or untracked .gitattributes or .gitignore file(s). Staging them now..." 1>&2
  git add -v $files
fi
```

Then we make it executable: `chmod +x $HOME/.git-template/hooks/pre-commit`

And finally we activate it: `git config --global init.templatedir ~/.git-template`

You only need to do that once (and every new team member is instructed to do it) and are protected against any future time that you forget to follow the policy

How does this work:

- The hook is copied automatically to the `.git/hooks` folder of every new repository you clone.
- Most of the time we clone internal repositories just to check some colleague's work. In that case, it doesn't matter if you have pre-commit installed or not because you are not doing any commit.
- Most of the time if you are going to modify something, you'll remember to run `pre-commit install` before committing.. IN that case, the initial hook has been already renamed to `..legacy`and the check is skipped.
- But if you forget to run `pre-commit install`, the hook will remind you to do so, and it will not let you commit until you do.
- If you are cloning a public repository that does not have a `.pre-commit-config.yaml` file, the hook will not do anything, so you can commit as usual.
- If you are cloning a public repository that has a `.pre-commit-config.yaml` file, then:
  - If you cloned it just for reading. then it doesn't matter if you have pre-commit installed or not because you are not doing any commit. But you will get the hook error message if you try later to do a commit. At that point ...
  - ... either you decide that you trust the repository and you run `pre-commit install` to activate it, or ...
  - ... you decide that you do not trust the repository, you delete or edit the `.git/hooks/pre-commit` file to disable the check and then you can commit without running the payload in `.pre-commit-config.yaml`

The final lines of the hook are not related to pre-commit, but to the way we use git-lfs. I left them here because they are useful to us, but you can remove them if you do not need them.

BTW: Once you have pre-commit, what we have in place to force that no big file is sent without git-lfs is the following (in `.pre-commit-config.yaml`):

```yaml
repos:
  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v5.0.0
    hooks:
      - id: check-added-large-files
        args: ['--maxkb=500']
```
