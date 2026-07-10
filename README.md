# eduardo-tools — Claude Code marketplace

A small [Claude Code](https://code.claude.com) plugin marketplace. Currently ships one plugin:

| Plugin | What it does |
|---|---|
| **[demo-gif](plugins/demo-gif)** | Record a calm, followable demo GIF of any web UI (Playwright → ffmpeg), saved under `.demo-gifs/` and gitignored. |

## Install

In Claude Code:

```
/plugin marketplace add oaklake-studio/demo-gif
/plugin install demo-gif@eduardo-tools
```

Then ask Claude to *"make a demo gif of &lt;your feature&gt;"*. Update later with `/plugin marketplace update eduardo-tools`.

> **demo-gif prerequisites:** Python ≥ 3.9, `ffmpeg`, and Chrome (or a one-time bundled-Chromium download). POSIX shell — on Windows use WSL/git-bash. See [plugins/demo-gif/README.md](plugins/demo-gif/README.md).

## License

MIT — see [LICENSE](LICENSE).
