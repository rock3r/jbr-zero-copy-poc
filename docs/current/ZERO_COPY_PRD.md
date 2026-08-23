# Zero-Copy Rendering — Current Snapshot

Updated: 2026-08-23

## Executive summary

Zero-copy is now an implemented, measured platform strategy on macOS and Windows—not one universal fast path. Its strongest product thesis is three kinds of avoided work:

1. **No readback:** keep finished pixels off the CPU on the way to Java2D.
2. **No rebuild:** hoist stable layer definitions and draw them by reference.
3. **No over-render:** coalesce repaint demand to the display composition clock.

These results support one architecture but come from separate gates. They must not be multiplied into a synthetic end-to-end speedup.

- **macOS:** shared Metal textures already remove the expensive pixel round-trip. Command replay, live child-layer replay, and hoisting work, but replay's incremental throughput value is marginal. Frame pacing has a strong quiet N=1 result and a later N=5 miss, so it remains experimental and default-off.
- **Windows:** the old path was readback-bound. The implemented shared-texture hand-off improves scored marker throughput by 1.59× to 3.62× and reduces CPU per frame by 29% to 66% as pixel area rises. Replay, live layers, hoisting, correctness, and composition-clock pacing are implemented; adoption remains opt-in while product-topology and hardware evidence expand.
- **Linux:** no backend exists. The likely design mirrors Windows—offscreen GPU rendering imported into Java2D and drawn in Swing paint—but it must branch across X11/GLX, XRender/X11, and Wayland/Vulkan.

The full PRD, measurements, architecture, shared-texture comparison, proof ledger, and Linux plan are at [specs.sebastiano.dev/zero-copy/](https://specs.sebastiano.dev/zero-copy/).

## Product intent

Render Compose on the GPU and present it without transferring finished pixels through CPU memory. Reduce repeated scene construction and redundant submissions without weakening correctness. Preserve a visible fallback whenever the accelerated path cannot prove ABI compatibility, capability, lifecycle health, or rendering correctness.

## Current state

### macOS

The Metal shared-texture hand-off is GPU-resident. Command replay is ABI/capability-gated and covers command, fallback, lifecycle, shared-layer, nested-motion, cache-reset, and eviction paths. Default-on hoisting reduced a 96-layer synthetic frame-build probe from 2,850 to 640 microseconds and recordings from 98 to 1.

Replay is not a blanket performance win over shared textures. A measured CPU-per-frame row was +12.3% against an unchanged +10% regression threshold. Frame pacing reduced deep intervals by 77.68% in a quiet pinned N=1 pair but missed its p95 ceiling; a later N=5 run at changed geometry reduced deep misses by only 4.7%. It remains default-off.

### Windows

Stage A shared-texture interop is complete and scored:

| Approx. pixels | Software markers/s | Shared texture markers/s | Gain | CPU/frame |
|---:|---:|---:|---:|---:|
| 1.0M | 87.1 | 138.6 | 1.59× | 29% less |
| 3.56M | 46.0 | 137.2 | 2.98× | 59% less |
| 7.98M | 26.6 | 96.2 | 3.62× | 66% less |

Pixel parity was 100.0000%; resize and popup gates passed. The hand-off chain covers D3D12 → D3D11.1 → legacy handle → D3D9Ex with strict fallback.

Replay is live across static UI, Canvas, graphicsLayer-heavy content, asynchronous Markdown layers, SwingPanel, popups, menus, and z-order probes. It runs at roughly 99 / 94 / 77 command frames/s in the same scale classes—faster than software, but about 25% below Stage A. A first-fusion icon-anchor defect was fixed; deterministic views retain characterized AA-only residuals, and live child-layer replay fixed the blank asynchronous preview. Hoisting is default-on and produces the same 2,850 → 640 microsecond, 98 → 1 result.

Windows pacing uses DWM composition timing plus a phase-aligned high-resolution waitable timer. Its functional gate is green. A clean pair produced about 2.6× fewer GPU submissions and cut the worst observed interval from 1,333 to 58 ms. The registered deep-miss reduction narrowly missed at 57.1% versus a 60% target, and present timestamps were unavailable; it remains opt-in/default-off as an efficiency and alignment result.

## Shared textures: solved and unsolved

The shared-texture API wraps a platform GPU texture as a Java2D-compatible `Image`; the painter then calls `Graphics2D.drawImage`. Windows likewise opens a shared D3D11 texture on Java2D D3D9Ex and blits only the destination composite clip. This proves the approach removes CPU pixel transport while remaining inside Swing paint order.

It does **not** unify renderers: Compose still rasterizes with Skia and Swing with Java2D, so minor text, antialiasing, and color differences remain. It does **not** create presentation timestamps. It also does **not** automatically solve mixed ownership among Compose overlays, SwingPanel children, glass panes, lightweight menus, and heavyweight popup windows.

The evidence is deliberately split:

- macOS has a passed mixed-window screenshot oracle with explicit Compose, Swing-island, overlay, and animation pixels;
- Windows markers prove SwingPanel + popup + menu used the accelerated path with zero fallback;
- the Windows ledger also records that PrintWindow/occlusion captured the hardware ComposePanel blank in both arms, so visual mixed-z-order parity is not yet proven there.

Therefore mixed-composition visual proof remains a Windows and Linux product gate rather than an inferred benefit of texture sharing.

## Linux plan

Current Skiko source renders Swing-hosted Linux content into an offscreen OpenGL texture, calls `flushAndSubmit(syncCpu = true)`, then uses `SoftwareSwingPainter`, whose implementation calls `readPixels` into a `BufferedImage`. Current JBR source exposes X11 GLX, XRender/X11, and Wayland/Vulkan configurations. This is sufficient proof of the opportunity and of the need for a capability matrix.

1. **Discovery and baseline:** record toolkit, Java2D configuration, compositor, Skiko adapter, device/format identity, exact readback path, pixel-area scaling, CPU/frame, renderer identity, and timing-source quality.
2. **Stage A — shared hand-off:** start with X11+GLX texture sharing/import; preserve Java2D paint clip, viewport, transform, and order. Evaluate external-memory import for Wayland+Vulkan. Investigate dma-buf/DRI3 pixmap import for XRender; fall back when portability or synchronization cannot be proven.
3. **Stage B — replay and retained work:** port the versioned ABI and live layer cache into a JBR-owned Skia backend after Stage A is correct and scored. Reuse Windows hoisting, invalidation, eviction, artifact identity, and fallback contracts.
4. **Stage C — pacing:** ship estimated refresh alignment first; upgrade X11/GLX only with a proven composition/video-sync source. Use Wayland surface callbacks only for the real destination and never label an estimated timer as presentation timing.
5. **Acceptance:** fixed N=5 pixel classes; renderer identity; CPU/frame and throughput; geometry plus AA characterization; resize/scale/output migration; device loss; async layers; SwingPanel/Compose overlap; glass pane; lightweight menu; heavyweight popup; occlusion; cache bounds; pacing idle/recovery; and explicit fallback.

## What remains

- Validate Windows on real IDE workloads, more GPUs, multiple displays, long lifecycle/device-reset sequences, and stable power/thermal campaigns.
- Add a Windows visual mixed-z-order oracle and the missing first-fusion recorder regression test.
- Add present-time or equivalent displayed-frame evidence and multi-display clock selection.
- Revisit draw-node diffing only if a real workload contains a large frequently re-recorded layer with low inline churn.
- Build and score the Linux backend end to end.

## Operating contract

- Finished pixels remain GPU-resident on supported paths.
- ABI, capability, lifecycle, or rendering failure selects a visible named fallback.
- Correctness covers resize, popup, mixed ownership, asynchronous layers, cache reset, and recovery—not only a static scene.
- Layer and image caches remain bounded and context-scoped.
- Pacing coalesces demand, becomes quiescent when idle, and recovers from a stalled timing source.
- Evidence uses fresh processes, pinned geometry, artifact identity, pre-registered criteria, and retained failed rows.

## Canonical code copies

These maintained hard copies are intentionally separate from similarly named upstream forks:

- [JBR zero-copy PoC](https://github.com/rock3r/jbr-zero-copy-poc/tree/jbr-skia-compose-poc)
- [JBR API zero-copy PoC](https://github.com/rock3r/jbr-api-zero-copy-poc/tree/jbr-api-skia-poc)
- [Magic Jewel zero-copy harness](https://github.com/rock3r/magic-jewel/tree/magic-jewel-skia-poc)
