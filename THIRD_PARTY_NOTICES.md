# Third-party notices

This repository includes copies of third-party material. Each entry lists what was copied, where
it lives, the upstream source pinned by commit, and the licence text that must travel with it.

## mattpocock/skills (MIT)

- **Copied:** the agent skills `tdd`, `diagnosing-bugs`, `resolving-merge-conflicts`, `to-tickets`
  and `writing-for-agents`.
- **Where:** `.claude/skills/<name>/` and `.agents/skills/<name>/` (identical copies).
- **Upstream:** https://github.com/mattpocock/skills at commit
  `3cca18b368ae95cdbdebbff572ccafa662551015`. Intake record:
  `docs/harness/intake/mattpocock-skills.yaml`.

```
MIT License

Copyright (c) 2026 Matt Pocock

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.
```

## Referenced, not included

These are used as development tools or studied as prior art. No code from them is in this
repository yet. Any future port adds an entry above with its licence text.

| Project | Licence | Relationship |
|---|---|---|
| [mattpocock/sandcastle](https://github.com/mattpocock/sandcastle) (`@ai-hero/sandcastle`) | MIT | Planned dev dependency for sandboxed agent runs (`docs/harness/sandcastle.md`). Not vendored |
| [Gsync/jobsync](https://github.com/Gsync/jobsync) | MIT | Prior art. Ports are planned per `docs/landscape.md`; none made yet |
