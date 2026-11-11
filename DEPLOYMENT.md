# Deployment

Orb Blast 3D is a static Unity WebGL build. There is no bundler, no package manager and
no server code: `scripts/prepare-site.mjs` reshapes the export and GitHub Actions
publishes the result.

## Where it lives

| Piece | Value |
| --- | --- |
| Repository | `jd4rider/orb-blast-3d-web` (public) |
| Play URL | <https://jd4rider.github.io/orb-blast-3d-web/play/> |
| Cover page | <https://jd4rider.github.io/orb-blast-3d-web/> |
| Host | GitHub Pages, Actions source (`pages.yml`), no custom domain |
| Catalogue | `littlejoyarcade/src/data/games.ts` + `supabase/migrations/0034_orb_blast_3d.sql` |
| Portal entry | <https://littlejoyarcade.site/#/game/orb-blast-3d> |
| Game id | `orb-blast-3d` |

The portal embeds the `play/` URL in an iframe, so the game has to keep working inside a
nested browsing context: no custom domain, no service worker, no `X-Frame-Options`.

## Build the payload

The Unity project is `~/Developer/Orb Blast 3D` (Unity 7000.0.0a12). The editor script
`Assets/OrbBlast3D/Editor/WebGLBuildSetup.cs` owns every player setting the host needs
and writes the export to `Builds/WebGL`. Run it from the open editor:

```bash
unity command --project-path "$HOME/Developer/Orb Blast 3D" \
  invoke_menu "Orb Blast/Build WebGL Player"
cat "$HOME/Developer/Orb Blast 3D/Logs/webgl-build-status.txt"   # STATE=done
```

Two things about that build are not preferences:

- **Compression is disabled.** Pages cannot send `Content-Encoding: br`, so a Brotli
  export dies at the progress bar. The script sets
  `PlayerSettings.WebGL.compressionFormat = Disabled` and `decompressionFallback = false`.
- **`-lembind` is passed and a shim is linked in.** The Emscripten toolchain bundled with
  Unity 7000.0.0a12 no longer defines `_emval_take_value` or `_emval_as`, which IL2CPP's
  `bindings.cpp.o` imports once a package ships a WebGL native plugin (here
  `com.unity.ai.inference` pulls in `com.unity.dt.app-ui`'s TextMateLib). Without
  `Assets/Plugins/OrbBlast3D/OrbBlastEmbindShim.jslib` the wasm link fails with
  `undefined symbol: _emval_take_value`.

Then copy the export here and push:

```bash
cp -R "$HOME/Developer/Orb Blast 3D/Builds/WebGL/." build/WebGL/
git add -A && git commit -m "Refresh the WebGL payload" && git push
```

Never commit the generated `site/`; CI rebuilds it.

## Deploy

```bash
git push origin main
gh run list --repo jd4rider/orb-blast-3d-web --limit 3
```

Pages must have its source set to **GitHub Actions** once:

```bash
gh api -X POST repos/jd4rider/orb-blast-3d-web/pages -f build_type=workflow
```

## Verify a deploy

```bash
curl -sI https://jd4rider.github.io/orb-blast-3d-web/play/                     # 200 text/html
curl -s -r 0-3 https://jd4rider.github.io/orb-blast-3d-web/play/Build/WebGL.wasm | xxd -p
# expect 0061736d (\0asm) - a Brotli or gzip payload would start with something else
```

A 200 is not proof the game runs: open the URL in a browser and wait for the progress bar
to disappear. The payload is uncompressed and larger in transit than a Brotli export would
be, because the host cannot set `Content-Encoding`.

## Publishing to the catalogue

The portal reads its catalogue from the `public.games` table, falling back to
`src/data/games.ts`. A new game updates both:

```bash
cd ~/Programming/littlejoyarcade
# 1. add the entry to src/data/games.ts
# 2. add supabase/migrations/0034_orb_blast_3d.sql (same shape as 0022_orb_sort_3d.sql)
npx supabase link --project-ref fqovramltegcfiyebyrt
npx supabase db push --include-all --yes
git add -A && git commit -m "Add Orb Blast 3D to the catalogue" && git push
```

`db push` is what makes the row live; the push to `main` is what rebuilds the portal
site itself.

## Rollback

Pages keeps the last deploy only. To roll back, revert the commit that changed
`build/WebGL/` and push:

```bash
git revert <sha> && git push
```
