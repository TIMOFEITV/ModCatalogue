# ModCatalogue

The live mod list for the ScamWYF launcher's **Get mods** tab.

The launcher fetches `mods.json` from this repository's `main` branch on every check, so adding
a mod means editing that file — not cutting a launcher release. Merging here reaches users
immediately, which is also why the file is validated strictly: anything unexpected rejects the
whole file and the launcher falls back to the list compiled into it, saying so on its status line.

## The file

One JSON object with a schema version and an array of mods:

```json
{
  "schemaVersion": 1,
  "mods": [
    {
      "owner": "swyf-modding",
      "repo": "AI-Backend",
      "displayName": "AI Backend",
      "summary": "Routes the game's AI calls to your own LLM.",
      "files": [
        { "fileName": "ScamWYF.AiBackend.dll", "folder": "plugins", "expectsPluginAttribute": true },
        { "fileName": "ScamWYF.Modding.Core.dll", "folder": "core", "expectsPluginAttribute": false }
      ]
    }
  ]
}
```

| Field | Required | Rules |
|---|---|---|
| `schemaVersion` | yes | Must be `1`. Anything else and old launchers refuse the file rather than guess at it |
| `owner` | no | A GitHub account name. Defaults to `swyf-modding` when absent |
| `repo` | yes | A GitHub repository name: letters, digits, `.`, `-`, `_` — no URLs, no slashes |
| `displayName` | yes | Up to 80 characters, shown in the list |
| `summary` | yes | Up to 500 characters, shown in the list. Describe what the mod does, not which other mods exist — the list grows and numbering it here rots |
| `files[].fileName` | yes | A bare `.dll` name: no folders, no `..`, nothing but the file. It becomes a file under the game folder, so this is enforced, not advised |
| `files[].folder` | yes | `plugins` for a mod, `core` for the shared library. Nothing else is accepted |
| `files[].expectsPluginAttribute` | yes | `true` for a mod (BepInEx will not load it without `[BepInPlugin]`), `false` for the shared library, which deliberately has none |
| `postInstall` | no | Up to four optional setup steps, offered one by one after installing — see below. Absent means none |

Keep entries alphabetical by repository name. At most 64 mods and 16 files per mod — caps, not targets.

### Optional setup steps

For what the download cannot do by itself: a stream mod's ChatBridge, which talks to Twitch and is
configured per machine, is the case this exists for. Each step names a script shipped **inside the
release zip** and a description shown in the prompt:

```json
"postInstall": [
  {
    "script": "setup-chatbridge.ps1",
    "description": "Installs and configures ChatBridge so the stream mod can read chat. Skipping leaves the mod installed but chat features off."
  }
]
```

| Field | Rules |
|---|---|
| `script` | A bare `.ps1` name — no folders, no `..` — resolved against the unpacked download. At most 64 KB, since the launcher shows it before asking |
| `description` | Up to 280 characters. Say what it does and what skipping costs; it is shown verbatim |

The launcher offers each step separately after the files are installed, shows the script, and defaults
to skipping — denying everything leaves the install as it was. A declared script the release does not
contain fails the install rather than being fetched from anywhere else. Scripts receive the game
folder as `-GameDir` and run with the download as their working directory.

## Getting your mod listed

Open a pull request editing `mods.json`. Your mod must already be installable by the launcher's
rules, because the entry only *names* it — the checks below still apply at install time:

1. **A public GitHub repository with versioned releases.** The launcher reads `releases/latest`,
   which ignores prereleases, so publish finished releases. Every release needs a downloadable
   `.zip` asset; a release with no zip cannot be installed from and is reported as such.
2. **Plugin DLLs the loader will actually load.** Each one carries a `[BepInPlugin]` attribute
   whose version BepInEx can parse: two to four dot-separated numbers, no letters, no `+` or `-`.
   A SemVer build-metadata version (`1.2.3+g0a1b2c3`) is valid SemVer and unloadable by BepInEx —
   the launcher flags it in the list rather than installing something that silently does nothing.
3. **The shared library routed to `core`, if you ship one.** `ScamWYF.Modding.Core.dll` goes in
   `core` with `expectsPluginAttribute: false`. In `plugins` the chainloader would try to load it
   as a plugin and the install would break in a way nothing explains.
4. **A display name and an honest summary.** Say what the mod does. The install confirmation shows
   the files, their versions read out of the binaries, and anything they would replace — there is
   no field here that overrides what is actually in the download.

Third-party repositories are welcome: set `owner` to your account. The launcher marks those
entries with their owner in the list and names the repository again in the install confirmation,
because unlike the project's own releases there is no shared review behind them.

## Trust

Listing here is curation, not a safety verdict. A merged entry means someone with merge access
chose to point at that repository — nothing more. The launcher downloads over https, checks the
download against the SHA-256 GitHub publishes (which catches corruption, not malice), and asks
before writing anything into the game folder. Only install mods from authors you trust, listed
or not.
