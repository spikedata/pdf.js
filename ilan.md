# Node 24 install failure: options

## Root cause

This checkout pins `canvas@^2.11.2` in `package.json`, and Node-side code loads it from `src/display/node_utils.js`.

On Node `24.10.0` / ABI `137`, `canvas@2.11.2` has no matching prebuilt binary for `darwin arm64`, so install falls back to a source build.

That source build then fails in `nan` against Node 24's V8 API, so this is not just a missing native toolchain problem.

## Options

1. Upgrade only `canvas` on this branch to `3.x`, preferably current `3.2.2` or at least `3.1.1`.

This is the smallest practical fix that still stays on Node 24. `node-canvas` 3.x moved to N-API, and `3.1.1` specifically includes a Node 24 crash fix. Because this repo only calls `createCanvas` in the local Node integration, the API jump is probably manageable, but Node-side rendering still needs retesting.

2. Move this `pdf.js` snapshot closer to current upstream `pdf.js`.

This is the cleanest long-term option. Current upstream `pdf.js` replaced `node-canvas` with `@napi-rs/canvas`, and upstream `package.json` explicitly allows Node `>=24`. Bigger change, lower long-term friction.

3. If Node-side rendering/tests are not needed, split canvas out of the default install path.

That means making the canvas dependency optional, or only installing it for the scripts that actually need Node rendering. This only works if the target workflow is browser/viewer work, not Node image/reference tests.

4. Carry a private patched fork of `canvas@2.11.2`.

This keeps the rest of the branch stable, but it is the highest-maintenance option. Since the failure is in `nan`/V8 compatibility, this is real code maintenance, not a packaging tweak.

## What probably will not fix it

Installing Cairo, Pango, Xcode, or forcing source build while staying on `canvas@2.11.2` is unlikely to solve this. The logged failure is in `nan` / V8 compatibility rather than missing system headers.

## What option 3 requires

Option 3 is valid for Node text extraction, metadata reads, and similar non-rendering workflows.

To make it work cleanly, the repo has to stop treating `canvas` as a mandatory install-time dependency while keeping a clear runtime error for rendering paths.

That means:

1. `canvas` must be optional in `package.json`, so `npm install` can succeed even when `canvas@2.11.2` cannot build on Node 24.

2. The Node rendering path must continue to load `canvas` lazily, and throw a clear message only if code actually tries to render in Node.

3. Node-only rendering examples such as `examples/node/pdf2png/pdf2png.js` must clearly state that they need the optional `canvas` dependency.

4. Non-rendering Node use cases, such as `examples/node/getinfo.js`, should continue to work without `canvas`.

## How to regenerate package-lock.json

From the `pdf.js` directory:

```bash
rm -rf node_modules package-lock.json
npm install
```

That regenerates `package-lock.json` from `package.json` using the current `npm` version.

Since this branch needs to work on Node 24, make sure the install is being run with `node v24.x`, because the npm version and platform can affect lockfile metadata.

If only the lockfile needs to be regenerated, without keeping installed modules:

```bash
rm -f package-lock.json
npm install --package-lock-only
```

In this repo, `npm install` also runs the `postinstall` Puppeteer download, while `npm install --package-lock-only` avoids that.
