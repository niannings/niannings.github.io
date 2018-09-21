# 念凝笙git笔记
>使用git pull提示refusing to merge unrelated histories
>创建了一个origin，两个人分别clone
>分别做完全不同的提交
>第一个人git push成功
>第二个人在执行git pull的时候,提示
>fatal: refusing to merge unrelated histories

*解决方法:*
```bash
  git pull --allow-unrelated-histories
```