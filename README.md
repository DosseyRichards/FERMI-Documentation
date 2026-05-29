# Fermi · WaveLedger Documentation

Source for the WaveLedger chain documentation site.

| Domain | Source dir | Build |
|---|---|---|
| `docs.fermi.world` / `docs.fermi.world` | `docs/` | `mkdocs build` → `site/` |

Built with [MkDocs Material](https://squidfunk.github.io/mkdocs-material/) and deployed via Vercel.

## Build locally

```bash
pip install -r requirements.txt
mkdocs serve   # http://127.0.0.1:8000
```

## Deploy

Vercel project root: this directory. Build command and output directory are declared in [`vercel.json`](vercel.json).

The companion Fourier language site lives in a separate repo: [DosseyRichards/Fermi-Fourier](https://github.com/DosseyRichards/Fermi-Fourier).

## Contributing

Open work items live in [TODO.md](TODO.md). When you write a page that references something not yet documented, append it there rather than writing a stub.
