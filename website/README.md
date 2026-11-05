# Audelle web client

Next.js (App Router) web client for Audelle, the browser audio editor.

## Run

```bash
bun install
bun dev
```

Open [http://localhost:3000](http://localhost:3000).

Other scripts from `package.json`:

```bash
bun run build
bun start
bun run lint
```

## Browser-only mode

When `NEXT_PUBLIC_API_URL` is unset, the build has no backend. DSP tools then run
on the device in a Web Worker. Set `NEXT_PUBLIC_API_URL` to a running Audelle
backend for server features.
