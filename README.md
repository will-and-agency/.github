## 🛠 Development Workflow & Git Standards

To maintain a clean and traceable project history, all contributors must follow these branching and commit conventions. Every task begins with a **GitHub Issue**.

---

### Branching Strategy
We use an **Issue-First** branching model. All branches must be prefixed with the appropriate type and include the **Issue ID**.

**Format:** `type/ID-short-description`

| Type | Purpose | Example |
| :--- | :--- | :--- |
| `feature/` | New functional requirements or tasks | `feature/42-add-login-form` |
| `bugfix/` | Standard bug fixes found during dev/QA | `bugfix/108-resolve-header-overlap` |
| `hotfix/` | Urgent fixes for the production branch | `hotfix/215-fix-crashing-api` |
| `refactor/` | Code changes that improve structure only | `refactor/77-cleanup-auth-logic` |
| `docs/` | Documentation changes and updates | `docs/15-update-readme-rules` |

> [!TIP]
> Always create your branch from `develop`. Ensure your local environment is synced before branching: `git checkout develop && git pull`.
> Always test the code before pushing, using any type of testing.

---

### Commit Message Guidelines
We use structured commit messages to make the project history scannable.

**Format:** `type(ID): brief description of change`

* **feature:** A new feature for the user.
* **bugfix:** A bug fix for the user.
* **hotfix:** An urgent fix for production.
* **refactor:** Code changes that neither fix a bug nor add a feature.
* **docs:** Documentation only changes.
* **chore:** Updating build tasks, dependencies, etc.

**Examples:**
* `feature(42): add validation to email input field`
* `bugfix(108): mobile menu now closes on backdrop click`
* `docs(15): add branching rules to readme`

> [!TIP]
> Always run this line before doing git add:
> ```markdown
> cargo fmt --all
>```
> This format all packages within a workspace, ensures consistent coding style by applying rules defined in rustfmt.toml. Also the CI pipeline will fail if you didnt run this line on your local terminal.
---

### Pull Request (PR) Process
1. **Link the Issue:** In your PR description, include the keyword `Closes #ID`. This automatically moves the task to **Done** on our Team Planning board.
2. **Review:** At least one team member must approve the PR before merging.
3. **Clean Up:** Delete your feature/bugfix branch immediately after the PR is successfully merged.
4. **Remember:** Always create the pull request to the develop branch, to keep main only clean and always working, so we push to main once a week.


---

### Assigned Tasks Process
We use a team planning board which is under **Projects** on the organization's page. There we add all tasks, and all task information needed and we get assigned task there aswell. For task description we use this text to create a task description:
```markdown
### Task

> [!NOTE]
> Implement the logic of the management where the Root Moderator creates initial company and the super mod

### Mobile Requirements (iOS/Android)
- [ ] frontend
- [ ] backend
- [ ] testing

### Web/Browser Requirements
- [ ] frontend
- [x] backend
- [ ] testing

### Connectivity & Sync
- [ ] API Endpoints defined/updated
- [ ] Real-time sync (WebSockets/Polling) validation
- [ ] Conflict resolution (what happens if both update at once?)

### metadata
**Estimated Time:** 1 day
**Type:** [x] Software [ ] Documentation [ ] Research  

```

> [!TIP]
> Remember to put X inside the brackets to select a rubric for the task description. For example here:
> ```markdown
> ### metadata
> **Estimated Time: 1 week  
> **Type:** [x] Software [ ] Documentation [ ] Research  
> **Assignee:** @tomhoq
> ```

### Documentation
We will use documentation for each branch we create and want to push. There's a structure to documentation:
1. We create .md files for documentation
2. If we do a bugfix, hotfix or refactor then we just update the existing documentation
3. When we write a documenation, we use the baby explaining technique


