# ntsim.uk

Personal website for Nicholas Tsim.

## Getting started

1. Install Node (as specified by `package.json`) via Mise:

   ```bash
   mise install node
   ```

2. Enable pnpm via Corepack:

   ```bash
   corepack enable pnpm
   ```

3. Install dependencies:

   ```bash
   pnpm install
   pnpm dev
   ```

## 🧞 Commands

All commands are run from the root of the project, from a terminal:

| Command                | Action                                           |
| :--------------------- | :----------------------------------------------- |
| `pnpm install`         | Installs dependencies                            |
| `pnpm dev`             | Starts local dev server at `localhost:4321`      |
| `pnpm build`           | Build your production site to `./dist/`          |
| `pnpm preview`         | Preview your build locally, before deploying     |
| `pnpm astro ...`       | Run CLI commands like `astro add`, `astro check` |
| `pnpm astro -- --help` | Get help using the Astro CLI                     |
