# GitHub Actions Project — What I Learned
A log of everything I ran into while building a CI/CD pipeline with GitHub Actions for a Next.js app — and exactly how I fixed each problem
1. 🔗 Connecting a local repo to GitHub

❌ Problem: git push failed with:

fatal: No configured push destination.

🔍 Cause: The local repo had no origin remote pointing to a GitHub repository yet.

✅ Fix:

bash
git remote add origin https://github.com/<username>/<repo>.git
git push --set-upstream origin <branch-name>

💡 Lesson: A local Git repo and a GitHub repo aren't automatically linked — you have to explicitly tell Git where to push with git remote add.

2. 🌐 Wrong remote URL / repo name mismatch

❌ Problem:

remote: Repository not found.

🔍 Cause: The repo name in the URL didn't match the actual GitHub repo name.

✅ Fix: Since origin already existed, git remote add failed with "remote origin already exists" — had to use git remote set-url instead to overwrite it:

bash
git remote set-url origin https://github.com/<username>/<correct-repo-name>.git

💡 Lesson: git remote add only works for creating a new remote. To fix an existing one, use git remote set-url.

3. 🕳️ "Entirely different commit histories"

❌ Problem: GitHub's compare page said main and my feature branch had nothing in common — no PR form, just thousands of raw additions.

🔍 Cause: My branch was created independently (its own fresh Git history) instead of being branched off the GitHub repo's existing main. Git had no shared ancestor commit to compare against.

✅ Fix:

bash
git fetch origin
git merge origin/main --allow-unrelated-histories

This gave both branches a shared history so GitHub could generate a normal diff and PR.

💡 Lesson: A feature branch needs to share commit history with the branch you're comparing it to.

4. 📁 Duplicate nested project folders

❌ Problem: After merging unrelated histories, the entire project got duplicated inside nested subfolders — e.g. next-js-app-main/next-js-app-main/...

🔍 Cause: Merging two projects that both had files at the root level, without realizing one was already nested inside the other.

✅ Fix:

bash
git rm -r <duplicate-folder>
git commit -m "Remove duplicate nested folder"
git push

💡 Lesson: Always check folder structure after a merge — especially an --allow-unrelated-histories merge.

5. 🧩 Git submodule error while deleting a folder

❌ Problem:

fatal: could not lookup name for submodule

🔍 Cause: One of the duplicate folders had its own .git folder inside it, so Git registered it as a submodule instead of normal tracked files.

✅ Fix:

bash
git rm -r --cached <folder>
rm -rf <folder>
<details> <summary>ℹ️ Why two commands?</summary>

--cached removes it from Git's tracking without trying to recurse into it as a submodule; rm -rf deletes it from disk separately.

</details>

💡 Lesson: A folder containing its own .git directory gets treated specially by Git — watch for this when copying folders from other projects/repos.

6. 🔒 "Device or resource busy" / folder won't delete

❌ Problem: rm -rf failed with Device or resource busy, and Windows File Explorer said "Folder In Use."

🔍 Cause: Another running process (an open VS Code window) had a file handle open inside that folder.

✅ Fix:

✔️ Closed all open editor tabs and terminal sessions pointing into the folder
✔️ Closed VS Code entirely
✔️ Restarted the computer to force-clear the lock
✔️ Deleted the folder from File Explorer before reopening VS Code

💡 Lesson: File locks on Windows are stubborn — a full restart is the guaranteed fix.

7. 💻 Wrong shell syntax (rm -rf in PowerShell)

❌ Problem:

Remove-Item : A parameter cannot be found that matches parameter name 'rf'

🔍 Cause: rm -rf is bash/Linux syntax. PowerShell's rm alias (Remove-Item) uses different flags.

✅ Fix:

powershell
Remove-Item -Recurse -Force <folder>

💡 Lesson: Git Bash and PowerShell are different environments with different command syntax — know which terminal you're in.

8. 🌿 Recreating an existing branch

❌ Problem:

fatal: a branch named 'X' already exists

🔍 Cause: Used git checkout -b <branch> (create new) on a branch that already existed.

✅ Fix: Just switch to it — no -b:

bash
git checkout <branch>

💡 Lesson: -b means "create," not "switch to." Use it only the first time a branch is made.

9. 🧪 Missing type declarations (red squiggles in test files)

❌ Problem: TypeScript couldn't find expect, it, or @testing-library/react — red underlines everywhere in test files.

🔍 Cause: node_modules hadn't been (re)installed after folder restructuring, and tsconfig.json's "types" array was missing "jest".

✅ Fix:

bash
npm i

Plus, added "jest" to the types array in tsconfig.json:

json
"types": ["@testing-library/jest-dom", "jest"]

💡 Lesson: Type errors for testing libraries usually mean dependencies aren't installed, or the relevant type package isn't declared in tsconfig.json.

10. 🐳 Docker image pushed to the wrong account

❌ Problem: GitHub Actions failed on docker push:

denied: requested access to the resource is denied

🔍 Cause: The workflow's image name (<username>/next-js-app) didn't match the Docker Hub account logged in via the stored secrets — first because it was the tutorial author's username, then because I'd used my GitHub username instead of my actual Docker Hub username.

✅ Fix: Updated .github/workflows/deploy.yml so both the docker build -t and docker push lines use the exact same, correct Docker Hub username:

yaml
- run: docker build . -t <dockerhub-username>/next-js-app
- run: echo "${{secrets.DOCKERHUB_PASSWORD}}" | docker login -u ${{secrets.DOCKERHUB_USERNAME}} --password-stdin
- run: docker push <dockerhub-username>/next-js-app:latest

⚠️ Lesson: The image tag in docker build and docker push must match exactly, and must use the Docker Hub username tied to the login credentials — not the GitHub username.

11. 💾 Local file edit didn't register with Git

❌ Problem: git commit kept saying:

nothing to commit, working tree clean

🔍 Cause: The file edit was never actually saved in the editor.

✅ Fix: Confirmed the file was saved (no unsaved-dot ● on the tab), then verified with:

bash
git status

— which then correctly showed the file as modified.

💡 Lesson: git status is the fastest way to confirm whether Git actually sees your edits.

12. 🪞 Comparing identical branches shows nothing

❌ Problem: GitHub said:

There isn't anything to compare.
main and workflow/deploy are identical.

🔍 Cause: The fix had already been committed directly to main before the feature branch was created — so the two branches had nothing left to diff.

✅ Fix: Made an actual new change on the feature branch first, then pushed it — giving GitHub a real diff and unlocking the PR button.

💡 Lesson: A pull request needs an actual difference between branches.

🏁 Overall Takeaways
Concept	Key Takeaway
🔗 Git remotes	Connect local ↔ remote repos — nothing pushes until origin is explicitly set correctly
🕳️ Shared commit history	Required for GitHub to generate a normal diff/PR between two branches
🔒 File locks (Windows)	Can block deletions even after closing the obvious window — a full restart is the reliable fix
🔑 GitHub Actions secrets	DOCKERHUB_USERNAME / DOCKERHUB_PASSWORD must exactly match the real account — every reference in the workflow must be consistent
🩺 git status	The first command to run whenever something isn't behaving — it almost always explains what's wrong
<div align="center">

🎉 Completed the full GitHub Actions CI/CD tutorial — remotes, merges, Docker builds, and pull requests.

</div>
