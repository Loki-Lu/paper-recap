# 可乐桶的 Paper Recap

个人论文阅读库，保存读过的论文、阅读时的疑问和自己的判断。

## 添加笔记

使用 Codex 的 `$update-paper-recap` Skill，或从 [`content/TEMPLATE.md`](content/TEMPLATE.md) 复制模板，在 `content/papers/` 中新增一篇 Markdown。标签从 [`content/tag-taxonomy.json`](content/tag-taxonomy.json) 选择。

## 本地预览

```sh
python3 scripts/build_site.py
python3 -m http.server 8000 --directory dist
```

访问 [本地预览页面](http://localhost:8000)。

## 发布

推送到 `main` 会触发 GitHub Actions 校验、构建并发布到 GitHub Pages。

站点代码源自 [lawrence-cj/paper-recap](https://github.com/lawrence-cj/paper-recap)，遵循 [MIT License](LICENSE)。阅读记录和媒体的版权见 [COPYRIGHT](COPYRIGHT)。
