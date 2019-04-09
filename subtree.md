# Subtrees

## Add remote so it's easier to work with

```bash
g r add github_io git@github.com:leshow/leshow.github.io
```

## Update subtree public

```bash
g subtree pull --prefix public github_io master --squash
```

## Pushing subtree

```bash
g subtree push --prefix public github_io master
```
