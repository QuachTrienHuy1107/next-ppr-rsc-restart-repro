# Cache Components: complete PPR pages lose their `.rsc` on disk (self-hosted `next start`)

Minimal reproduction, created from `npx create-next-app -e reproduction-template`.
Tested with `next@16.4.0-canary.63` (also reproduces on 16.3.5 and 16.3.8).

`next.config.ts` enables `cacheComponents` and `partialPrefetching`; `app/data.ts` is a `'use cache'`
function with `cacheLife({ revalidate: 10 })` that returns the time its entry was filled (`at=`).

- `app/[locale]/p/[id]/page.tsx` — only `seed` is in `generateStaticParams`; other ids are served as a
  fallback shell first and then upgraded to a static page in the background.
- `app/[locale]/page.tsx` — prerendered at build time, regenerated at runtime.

## Run

```sh
npm install
./repro.sh                       # case A (restart) + case B
PROBE_MEM=40000 ./repro.sh evict # case A via in-memory cache eviction instead of a restart
```

`repro.sh` builds, starts `next start`, requests the pages, restarts the server and requests them again.

## Output (next@16.4.0-canary.63)

```
== Case A: page outside generateStaticParams
/en/p/a (1st request: fallback shell)        at=1791309264848 x-nextjs-postponed: 1
/en/p/a (2nd request: upgraded)              at=1791309264848 x-nextjs-cache: HIT
   files on disk: a.html a.meta a.segments
== Case B: prerendered page regenerated at runtime
/en (build-time data)                        at=1791309264043 x-nextjs-cache: HIT
/en (stale, triggers regeneration)           at=1791309264043 x-nextjs-cache: STALE
/en HTML (regenerated)                       at=1791309278060 x-nextjs-cache: HIT
/en RSC (regenerated)                        at=1791309278060 x-nextjs-cache: HIT
== restart next start
/en/p/a  expected: HIT from disk             at=1791309280893 x-nextjs-postponed: 1
/en HTML                                     at=1791309278060 x-nextjs-cache: HIT
/en RSC  expected: same at= as HTML          at=1791309264043 x-nextjs-cache: HIT
```

- Case A: after the restart the upgraded page is served as a fallback shell again and re-rendered; no
  `a.rsc` was ever written next to `a.html`. Same after eviction with a small `cacheMaxMemorySize`.
- Case B: after the restart the HTML has the regenerated data but the RSC payload (client navigation)
  has the build-time data, labelled `HIT`.
