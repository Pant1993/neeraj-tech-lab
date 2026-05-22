# Neeraj's Tech Lab

Personal technical blog — deep dives into ARM architecture, SoC internals, firmware bring-up, and silicon engineering.

Built with [Hugo](https://gohugo.io/) + [PaperMod](https://github.com/adityatelange/hugo-PaperMod) theme. Deployed on GitHub Pages.

## Local Development

```bash
# Install Hugo (extended edition)
# https://gohugo.io/installation/

# Clone with submodules
git clone --recurse-submodules https://github.com/neerajpant/neeraj-tech-lab.git
cd neeraj-tech-lab

# Run dev server
hugo server -D

# Build for production
hugo --gc --minify
```

## Writing a new post

```bash
hugo new content posts/my-new-post.md
```

## Deploy

Push to `main` branch → GitHub Actions automatically builds and deploys to GitHub Pages.
