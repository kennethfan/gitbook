---
description: git修改已提交记录提交者信息
---

# git修改已提交记录提交者信息

```bash
git filter-branch --env-filter '
if [ "$GIT_COMMITTER_EMAIL" = "old.email@example.com" ]; then
    export GIT_COMMITTER_NAME="New Name"
    export GIT_COMMITTER_EMAIL="new.email@example.com"
fi
if [ "$GIT_AUTHOR_EMAIL" = "old.email@example.com" ]; then
    export GIT_AUTHOR_NAME="New Name"
    export GIT_AUTHOR_EMAIL="new.email@example.com"
fi
' -- --all

git push -f
```
