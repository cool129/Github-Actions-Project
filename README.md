# Next.js + Jest

This example shows how to configure Jest to work with Next.js.

This includes Next.js' built-in support for Global CSS, CSS Modules and TypeScript. This example also shows how to use Jest with the App Router and React Server Components.

> **Note:** Since tests can be co-located alongside other files inside the App Router, we have placed those tests in `app/` to demonstrate this behavior (which is different than `pages/`). You can still place all tests in `__tests__` if you prefer.

## Deploy your own

[![Deploy with Vercel](https://vercel.com/button)](https://vercel.com/new/clone?repository-url=https://github.com/vercel/next.js/tree/canary/examples/with-jest&project-name=with-jest&repository-name=with-jest)

## How to Use

Quickly get started using [Create Next App](https://github.com/vercel/next.js/tree/canary/packages/create-next-app#readme)!

In your terminal, run the following command:

```bash
npx create-next-app --example with-jest with-jest-app
```

```bash
yarn create next-app --example with-jest with-jest-app
```

```bash
pnpm create next-app --example with-jest with-jest-app
```

## Running Tests

```bash
npm test
```

``````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````
GitHub Actions Project — What I Learned

A log of everything I ran into while building out a CI/CD pipeline with GitHub Actions for a Next.js app, and how I actually fixed each problem.

1. Connecting a local repo to GitHub

Problem: git push failed with fatal: No configured push destination.

Cause: The local repo had no origin remote pointing to a GitHub repository yet.

Fix:

bash
git remote add origin https://github.com/<username>/<repo>.git
git push --set-upstream origin <branch-name>

Lesson: A local Git repo and a GitHub repo aren't automatically linked — you have to explicitly tell Git where to push with git remote add.

2. Wrong remote URL / repo name mismatch

Problem: remote: Repository not found. after adding a remote.

Cause: The repo name in the URL didn't match the actual GitHub repo name.

Fix: Since origin already existed, git remote add failed with "remote origin already exists" — had to use git remote set-url instead to overwrite it:

bash
git remote set-url origin https://github.com/<username>/<correct-repo-name>.git

Lesson: git remote add only works for creating a new remote. To fix an existing one, use git remote set-url.

3. "Entirely different commit histories"

Problem: GitHub's compare page said main and my feature branch had nothing in common and couldn't offer a normal pull request — it just showed thousands of additions with no PR form.

Cause: My branch was created independently (its own fresh Git history) instead of being branched off the GitHub repo's existing main. Git had no shared ancestor commit to compare against.

Fix:

bash
git fetch origin
git merge origin/main --allow-unrelated-histories

This gave both branches a shared history so GitHub could generate a normal diff and PR.

Lesson: A feature branch needs to share commit history with the branch you're comparing it to. If they were built as separate, disconnected repos, GitHub treats them as unrelated.

4. Duplicate nested project folders

Problem: After merging unrelated histories, my repo ended up with the entire project duplicated inside nested subfolders (e.g. next-js-app-main/next-js-app-main/...).

Cause: Merging two projects that both had files at the root level, without realizing one was already nested inside the other.

Fix:

bash
git rm -r <duplicate-folder>
git commit -m "Remove duplicate nested folder"
git push

Lesson: Always check folder structure after a merge, especially an --allow-unrelated-histories merge — it's easy to end up with unwanted nesting.

5. Git submodule error while deleting a folder

Problem: git rm -r failed with fatal: could not lookup name for submodule.

Cause: One of the duplicate folders had its own .git folder inside it, so Git registered it as a submodule (a link to another repo) instead of normal tracked files.

Fix:

bash
git rm -r --cached <folder>
rm -rf <folder>

--cached removes it from Git's tracking without trying to recurse into it as a submodule; rm -rf deletes it from disk separately.

Lesson: A folder containing its own .git directory gets treated specially by Git. Watch for this when copying folders from other projects/repos.

6. "Device or resource busy" / folder won't delete

Problem: rm -rf failed with Device or resource busy, and Windows File Explorer said "Folder In Use."

Cause: Another running process (an open VS Code window in this case) had a file handle open inside that folder.

Fix:

Closed all open editor tabs and terminal sessions pointing into the folder
Closed VS Code entirely
When that still didn't release it, restarted the computer to force-clear the lock
Deleted the folder from File Explorer before reopening VS Code

Lesson: File locks on Windows are stubborn — closing the visible window isn't always enough if another background process or window still references the folder. A restart is the guaranteed fix.

7. Wrong shell syntax (rm -rf in PowerShell)

Problem: Remove-Item : A parameter cannot be found that matches parameter name 'rf'

Cause: rm -rf is bash/Linux syntax. PowerShell's rm alias (Remove-Item) uses different flags.

Fix:

powershell
Remove-Item -Recurse -Force <folder>

Lesson: Git Bash and PowerShell are different environments with different command syntax — know which terminal you're in.

8. Recreating an existing branch

Problem: fatal: a branch named 'X' already exists

Cause: Used git checkout -b <branch> (create new) on a branch that was already created earlier.

Fix: Just switch to it without -b:

bash
git checkout <branch>

Lesson: -b means "create," not "switch to." Use it only the first time a branch is made.

9. Missing type declarations (red squiggles in test files)

Problem: TypeScript couldn't find expect, it, or @testing-library/react — lots of red underlines in test files.

Cause: node_modules hadn't been (re)installed after all the folder restructuring, and tsconfig.json's "types" array was missing "jest".

Fix:

bash
npm i

and added "jest" to the types array in tsconfig.json:

json
"types": ["@testing-library/jest-dom", "jest"]

Lesson: Type errors for testing libraries usually mean either dependencies aren't installed, or the relevant type package isn't declared in tsconfig.json.

10. Docker image pushed to the wrong account

Problem: GitHub Actions failed on docker push with denied: requested access to the resource is denied.

Cause: The workflow's image name (<username>/next-js-app) didn't match the Docker Hub account actually logged in via the stored secrets — first because it was still the tutorial author's username, then because I'd used my GitHub username instead of my actual Docker Hub username.

Fix: Updated .github/workflows/deploy.yml so both the docker build -t and docker push lines use the exact same, correct Docker Hub username on both lines:

yaml
- run: docker build . -t <dockerhub-username>/next-js-app
- run: echo "${{secrets.DOCKERHUB_PASSWORD}}" | docker login -u ${{secrets.DOCKERHUB_USERNAME}} --password-stdin
- run: docker push <dockerhub-username>/next-js-app:latest

Lesson: The image tag used in docker build and docker push must match exactly, and must use the Docker Hub username tied to the login credentials — not the GitHub username, which can be different.

11. Local file edit didn't register with Git

Problem: git commit kept saying "nothing to commit, working tree clean" even after trying to make changes.

Cause: The file edit was never actually saved in the editor.

Fix: Confirmed the change was saved (no unsaved-dot on the file tab), verified with:

bash
git status

which then correctly showed the file as modified, and the commit went through normally.

Lesson: git status is the fastest way to confirm whether Git actually sees your edits before assuming something is broken.

12. Comparing identical branches shows nothing

Problem: GitHub said "There isn't anything to compare" / "main and workflow/deploy are identical" when trying to open a pull request.

Cause: The fix had already been committed directly to main before the feature branch was created from it — so the two branches had no differences left to show.

Fix: Made an actual new change on the feature branch first, then pushed it, which gave GitHub a real diff to display and allowed opening a PR.

Lesson: A pull request needs an actual difference between branches. If you fix something on main first and branch off afterward, there's nothing left to compare.
