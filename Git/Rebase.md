# Rebase your feature branch

```bash
git checkout feature/jira_ticket_123
git fetch origin
git rebase origin/develop
```

At this point you could force-push your changes. If not possible, create a temporary branch and push it (so you don't risk losing changes)

```bash
git checkout -b feature/jira_ticket_123_rebased

git push --set-upstream origin feature/jira_ticket_123_rebased
```

Then after deleting the remote old branch:

```bash
git checkout feature/jira_ticket_123_rebased
git branch -D feature/jira_ticket_123
git checkout -b feature/jira_ticket_123
git push -u origin feature/jira_ticket_123
```
