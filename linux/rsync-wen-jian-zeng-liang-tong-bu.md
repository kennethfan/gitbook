# rsync文件增量同步

```bash
rsync -avl --delete <source> <dest>
```

rsync支持远程同步

```
rsync -avl --delete <user>@<host>:<source> <dest>

rsync -avl --delete <source> <user>@<host>:<dest>
```
