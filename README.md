# Offline Content Grabber

Grab web pages and files while you're on Wi-Fi, then read them anywhere, offline.
Rust core + (soon) Flutter UI.

## Layout

```
ocg/
├── core/              # ocg-core: all the logic, no Flutter deps
│   ├── schema.sql     # blobs, items, resources, jobs
│   └── src/
│       ├── store.rs   # SQLite + content-addressed blob folder
│       ├── fetch.rs   # Fetcher trait, PageFetcher, FileFetcher, Grabber
│       ├── queue.rs   # run_queue(): drain jobs while can_download() is true
│       └── server.rs  # axum server on 127.0.0.1 for offline viewing
└── cli/               # `ocg` binary: desktop test harness for core
```

## Try it

```bash
cargo test
cargo run -p ocg-cli -- add https://example.com   # queue it (works offline)
cargo run -p ocg-cli -- run                       # "on Wi-Fi": download the queue
cargo run -p ocg-cli -- list
cargo run -p ocg-cli -- serve                     # open http://127.0.0.1:8787/
```

Turn off your internet, run `serve`, and the saved pages still load.

## How a page is saved

1. Download the HTML (`reqwest`).
2. Pass 1 (`lol_html`): find the `<title>` and every asset (`img`, `script`, `source`, stylesheet, icon).
3. Download each asset and hash it (SHA-256). Same bytes → same file, stored once.
4. Pass 2: rewrite each asset reference to `/blob/<hash>`, drop `srcset` and `<base>`.
5. Store blobs + metadata in one transaction.

Fetchers return a `Bundle` and never touch the database, so they're easy to test
and the futures stay `Send` (needed for background tasks).

## Roadmap

- [x] **Phase 1** – save a URL with its assets, view it offline
- [ ] **Phase 2** – stream big files to disk with HTTP `Range` resume; download assets concurrently;
      rewrite `url(...)` inside CSS files; progress events
- [ ] **Phase 0/3** – Flutter app (see below), Wi-Fi-only rule, Android WorkManager background runs
- [ ] **Phase 4** – RSS/podcasts, Kiwix ZIM, storage limit + cleanup, search

## Hooking up Flutter (Phase 0)

```bash
cargo install flutter_rust_bridge_codegen
flutter_rust_bridge_codegen create app      # Flutter app with a rust/ crate inside
```

Then make `app/rust` depend on `ocg-core` and expose a thin API, e.g.:

```rust
pub async fn enqueue(data_dir: String, url: String) -> Result<i64, String>;
pub async fn run_queue(data_dir: String) -> Result<usize, String>;   // Flutter calls this on Wi-Fi
pub fn list_items(data_dir: String) -> Result<Vec<ItemDto>, String>;
pub async fn start_server(data_dir: String, port: u16) -> Result<(), String>;
```

Flutter side: `connectivity_plus` to know when you're on Wi-Fi, `path_provider` for `data_dir`,
`webview_flutter` pointed at `http://127.0.0.1:<port>/item/<id>` to read saved pages.
On Android, add `android:usesCleartextTraffic` (or a network security config) for 127.0.0.1.

## Notes

- Server binds to `127.0.0.1` only. Saved pages share one origin, so a saved page's scripts
  could read other saved content. Consider a "strip scripts" option for untrusted sites.
- Only save content you're allowed to; DRM streaming services are off-limits.
# Ocg
