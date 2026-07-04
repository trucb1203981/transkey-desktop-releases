# TransKey on-device translation models (bộ dịch)

`manifest.json` is the single source of truth for the on-device Lens MT
models. The desktop app fetches it at runtime from this repo's raw URL:

```
https://raw.githubusercontent.com/trucb1203981/transkey-desktop-releases/main/models/manifest.json
```

then downloads the model binaries from the `url` field (Cloudflare R2).
Model binaries NEVER enter git - the `models-to-r2.yml` workflow mirrors
them from `source_url` (Hugging Face) to R2 and verifies size + sha256.

## Layout on R2 (bucket `transkey-releases`)

```
models/manifest.json          <- copy of this manifest (uploaded by CI)
models/<file>.gguf            <- model binaries
```

## Manifest structure (schemaVersion 1)

```jsonc
{
  "schemaVersion": 1,          // app rejects manifests with a newer schema
  "updatedAt": "YYYY-MM-DD",
  "models": [
    {
      "id": "qwen2.5-3b-instruct-q4km",  // unique, stable
      "tier": "3b",             // app slot: '3b' (quality) | '1.5b' (light)
      "engine": "llm",          // must be 'llm' (node-llama-cpp)
      "format": "gguf",         // must be 'gguf'
      "file": "qwen2.5-3b-instruct-q4_k_m.gguf",  // filename on R2 + on disk
      "bytes": 2104932768,      // EXACT content-length (verify with curl -sIL)
      "sha256": "…",            // integrity check after download
      "url": "",                // R2 public URL; empty -> app uses source_url
      "source_url": "https://huggingface.co/...", // mirror source + app fallback
      "minAppVersion": "2.0.0"  // older apps ignore this entry
    }
  ]
}
```

## Adding or upgrading a model - checklist

The app's contract: each entry fills one capability TIER (`3b` = strong
machines, `1.5b` = light). Upgrading = pointing a tier at a new file.

1. The model MUST be a GGUF chat/instruct model loadable by node-llama-cpp
   (`engine: llm`). Nothing else works in the app - no ONNX, no safetensors.
2. Test it locally FIRST: drop the file into `~/.transkey/models/llm/`,
   temporarily edit the app's bundled registry (or local manifest cache) to
   the new file/bytes, run live mode on a real region and check quality +
   latency + RAM (see `transkey-desktop/HANDOFF.md` item 2 for the bar).
3. Fill `bytes` from the REAL content-length
   (`curl -sIL <source_url> | grep -i content-length`) - a wrong value makes
   the app re-download forever. Fill `sha256` (`shasum -a 256 <file>`) -
   CI refuses a manifest without it and re-verifies during the mirror.
4. Fill `url` with the R2 public URL
   (`https://<R2-public-base>/models/<file>`). While `url` is `""` the app
   downloads from `source_url` (HF) instead - still works, just not mirrored.
5. Bump `minAppVersion` if the model needs app-side changes (new prompt
   format, bigger context...). Older apps then keep their bundled defaults.
6. Keep the OLD entry's file on R2 until the fleet has moved; CI never
   deletes model files.
7. Push to main -> `models-to-r2.yml` validates the manifest against the app
   contract (schema, one model per tier, gguf/llm only, sha256 present) and
   mirrors any missing binary to R2. A malformed manifest fails CI and never
   reaches users - the app additionally validates at runtime and falls back
   to its bundled defaults.
