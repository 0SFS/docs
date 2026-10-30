# Find, install and publish mods

0SFS mods are public GitHub repositories. A mod can supply an aircraft, an
engine, a reusable component, a payload, a plugin, or a collection of mods.
The simulator's **Mods** tab searches GitHub directly without a GitHub login
or API token.

## Find and add a mod

Open **Mods → Add mod** and type part of a repository's name or description,
such as `camera` or `DJI`. Matching repositories appear below the field as
you type. Choose a category to narrow the list, then choose a result's
**Add** action. Leave the field empty to browse mods in that category.

Search includes public repositories with the `0sfs-mod` topic, including
forks. Template repositories are excluded from the install list. Topics
make a repository discoverable; the simulator still checks its manifest
when you add it. A search result is not a guarantee that the mod works
with the current simulator.

You can also enter a source directly in the same field:

| Source | Example |
| --- | --- |
| GitHub owner and repository | `0SFS/DJI-Mini2` |
| GitHub repository URL | `https://github.com/0SFS/DJI-Mini2` |
| A particular tag or branch | `alice/my-plane@v1.2.0` |
| GitHub branch URL | `https://github.com/alice/my-plane/tree/main` |
| Published GitHub Pages site | `https://alice.github.io/my-plane/` |
| A direct manifest URL | `https://mods.example.org/my-plane/0sfs-mod.json` |

Direct entry also works for repositories whose owners have not added the
discovery topics. An unpinned GitHub source loads the published GitHub
Pages manifest. A tag or branch source loads `public/0sfs-mod.json` from
that ref. Other assets must be available at the paths the manifest names.

The development server also offers **This computer** for supported local
checkouts. That mode loads the checkout through the dev server and does
not search GitHub.

GitHub limits unauthenticated repository searches to
[10 requests per minute](https://docs.github.com/en/rest/search/search#rate-limit).
The tab waits briefly after typing, reuses cached results for ten minutes,
and displays a waiting state when it must pause. A GitHub limit or network
failure does not prevent adding a known source directly.

## Choose discovery topics

Every mod needs **`0sfs-mod`** and the category topic matching its manifest's
`kind`. The universal topic makes it appear under **All**; the category
topic makes the matching category filter find it.

| Category | Manifest `kind` | Category topic | Examples |
| --- | --- | --- | --- |
| Aircraft | `aircraft` | `0sfs-aircraft-mod` | Airplanes, helicopters, drones |
| Engines | `engine` | `0sfs-engine-mod` | Piston engines, turbines, electric motors and drone propulsion |
| Components | `component` | `0sfs-component-mod` | Gimbals, cameras, propellers, shared flight models, sound renderers |
| Payloads | `payload` | `0sfs-payload-mod` | Weapons, pods, external fuel tanks |
| Plugins | `plugin` | `0sfs-plugin-mod` | Optional features such as a camera preview |
| Modpacks | `modpack` | `0sfs-modpack-mod` | Collections that list other mods as dependencies |

For example, the DJI Mini 2 aircraft uses `0sfs-aircraft-mod`, its electric
propulsion mod uses `0sfs-engine-mod`, and a camera or gimbal uses
`0sfs-component-mod`. A reusable quadcopter flight component is a component;
the complete flyable drone is an aircraft.

Older category topics such as `0sfs-aircraft` and `0sfs-engine` may remain
for older clients. Add the new category topic alongside them. Search's
current category filters use the `0sfs-{kind}-mod` names above.

Repository owners can add topics through GitHub's **About → Edit → Topics**
controls. Add both required topics, keep any other useful topics, and save.
See GitHub's [topic instructions](https://docs.github.com/en/repositories/managing-your-repositorys-settings-and-features/customizing-your-repository/classifying-your-repository-with-topics).
With the GitHub CLI, an aircraft owner can instead run:

```sh
gh repo edit OWNER/REPO --add-topic 0sfs-mod --add-topic 0sfs-aircraft-mod
```

## Publish a mod

1. Start from the appropriate [aircraft](https://github.com/0SFS/mod-template-aircraft),
   [engine](https://github.com/0SFS/mod-template-engine),
   [payload](https://github.com/0SFS/mod-template-payload), or
   [plugin](https://github.com/0SFS/mod-template-plugin) template, or use an
   existing component or modpack as a reference.
2. Edit `public/0sfs-mod.json` and the assets it names. Give the mod its own
   identity, set `kind`, declare its owner and repository in `provenance`,
   and list the other mods it needs in `dependencies`.
3. Run the repository's manifest validation command. Keep the manifest and
   every referenced asset available together when publishing.
4. Publish the contents of `public/` using the repository's GitHub Pages
   workflow. Confirm that `https://OWNER.github.io/REPO/0sfs-mod.json` is
   reachable and that the assets load. A public source repository alone
   does not publish its mod files.
5. Add `0sfs-mod` and the category topic. Try both direct entry and discovery
   in the Mods tab. GitHub's search index may take time to reflect new topics.

Dependencies are loaded once even when several mods need them. The default
modpack and its dependencies come with the simulator; added mods can be
shared through the **Share modded flight** link in the Mods tab.

## Confirmation and code

Official `0SFS` and `foss-earth` sources load directly. Other owners ask for
confirmation. The optional developer username setting also lets repositories
under that username load directly; entering it is not a GitHub sign-in.

Declarative mods provide data. WASM mods run in a restricted sandbox.
JavaScript mods run unsandboxed code and require particular care: third-party
JavaScript never starts automatically from a shared flight link. Add it in
the Mods tab only when you trust its author.

The [modding architecture specification](proposals/github-modding.md) describes
the manifest, dependencies, execution modes and runtime interfaces in detail.
