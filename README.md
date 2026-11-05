# AI Recommended Disposition Codes for Dynamics 365 Contact Center

When a conversation ends, this solution reads the Copilot conversation summary, asks an AI Builder prompt (GPT-4.1-mini) which of your **own** disposition codes fit, and shows the agent a modal with the top 1–3 recommendations. The agent confirms, changes or adds codes and saves. If the AI has nothing to recommend, the modal shows a clickable list of all your codes (with search) so the agent is never stuck.

[![CI](https://github.com/moliveirapinto/d365-ai-disposition-codes/actions/workflows/ci.yml/badge.svg)](https://github.com/moliveirapinto/d365-ai-disposition-codes/actions/workflows/ci.yml)

| | |
|---|---|
| **Solution** | `AIDispositionCode` (unmanaged), version **1.0.0.3** |
| **Components** | Form handler (web resource), Power Automate flow, AI Builder prompt, canvas custom page |
| **Download** | [Releases](https://github.com/moliveirapinto/d365-ai-disposition-codes/releases/latest) → `AIDispositionCode_1_0_0_3.zip` |
| **Full documentation** | [docs/DEPLOYMENT-GUIDE.md](docs/DEPLOYMENT-GUIDE.md) (architecture, data flow, security, troubleshooting, rollback) |

This is a hardened and extended version of the *AI Disposition Codes* tool from [microsoft/d365cc-fde](https://github.com/microsoft/d365cc-fde/tree/main/tools/ai-disposition-codes), kept here because [PR #22](https://github.com/microsoft/d365cc-fde/pull/22) with these changes has not been merged. See [What changed](#what-changed-from-the-original).

---

## Before you start

You need all of these in the target environment:

| # | Requirement | How to check |
|---|-------------|--------------|
| 1 | Dynamics 365 Customer Service Enterprise with **Omnichannel / Contact Center** | The `msdyn_ocliveworkitem` table exists |
| 2 | **Copilot for Customer Service** enabled, with conversation summaries being generated | After a test conversation, the `msdyn_conversationinsights` row has a non-empty `msdyn_copilotsummary` |
| 3 | **Disposition codes** created and assigned to the workstream | Customer Service admin center → Workstreams → your workstream → Disposition codes |
| 4 | **Creator Kit Core** (provides the `PowerCAT Picker` control) | Install from [AppSource](https://appsource.microsoft.com/product/dynamics-365/microsoftpowercatarch.creatorkit1) or the [Creator Kit GitHub repo](https://github.com/microsoft/powercat-creator-kit) |
| 5 | **AI Builder** credits and the prompt feature available in your region | [Region availability](https://learn.microsoft.com/ai-builder/availability-region). Production or Sandbox environment (not Developer) |
| 6 | You are **System Administrator** or Solution Customizer | |

---

## Install

> **Read this first.** The solution contains the whole **Active Conversation** form of `msdyn_ocliveworkitem` (that is how it registers its `onLoad` handler). Importing with *Overwrite unmanaged customizations* replaces the form's unmanaged layer and can **silently remove other custom handlers** on that form. Note the form's current Events (Form properties → Events) before you import, and do not enable that option.

1. **Download** `AIDispositionCode_1_0_0_3.zip` from the [latest release](https://github.com/moliveirapinto/d365-ai-disposition-codes/releases/latest).
2. Go to [make.powerapps.com](https://make.powerapps.com), pick the target environment → **Solutions** → **Import solution** → browse to the zip.
3. When asked for the **Dataverse connection**, create or select one. Use a service account for production: the flow runs as this connection's owner, not as the agent.
4. Click **Import** and wait (typically 1–3 minutes; a warning that the existing flow definition was replaced is normal on re-import).
5. Open the solution → **Cloud flows** → **Disposition Recommender** → **Turn on**.
6. **Publish all customizations**.
7. **Verify the form handler:** Tables → Conversation (`msdyn_ocliveworkitem`) → Forms → *Active Conversation* → Events → `onLoad` should list `ccaasfde_dispositioncodeformhandler` with *Pass execution context* ticked. Re-add it if it is missing, and confirm any handlers you had before are still there.
8. **Verify the web resource is published** and is the new version: open `ccaasfde_dispositioncodeformhandler` (Solutions → Web resources) and check the text contains `Form Handler v2.2`. If an import leaves an older script served, select the web resource and choose **Publish**.
9. Hard-refresh the Customer Service workspace (Ctrl+F5).

### Test it

1. Start a chat or voice conversation in the workspace, exchange a few messages, and end it.
2. Wait for the Copilot summary (a few seconds to a minute). The modal opens on its own.
3. **Expected:** up to three recommended codes appear as tags, chosen from your own code list. Change the selection and click **Save** (the button is disabled until at least one code is selected).
4. **Check the result:** a new row appears in the *Conversation disposition code map* table for that conversation, and the Active Conversation form shows the codes after the refresh.
5. **No-recommendation path:** end a conversation with no clear topic (or before Copilot has produced a summary). The modal says *"No AI recommendations available…"* and shows the list of codes. Type in the search box to narrow it, click a code to add it.

The full checklist, including negative tests, is in [Section 12 of the guide](docs/DEPLOYMENT-GUIDE.md#12-post-deployment-validation).

---

## Configuration

| Setting | Where | Default | Meaning |
|---------|-------|---------|---------|
| `CONFIG.ENABLED` | Web resource `dispositioncodeformhandler.js` | `true` | Kill switch. Set to `false` and publish to stop the modal without removing the registration |
| `CONFIG.ENABLED_WORKSTREAM_IDS` | Same file | `[]` (all) | Only open the modal for these workstreams (lowercase GUIDs) |
| `maxRecommendations` | Flow action **Config** | `3` | How many codes to recommend |
| `minConfidence` | Flow action **Config** | `0.4` | Drop recommendations below this confidence (0–1) |
| `pollMaxAttempts` | Flow action **Config** | `15` | Maximum attempts to find the Copilot summary, polled every 2 s (a hard 2-minute limit also applies) |
| Prompt | AI Builder → Prompts → *Disposition code classifier* (source: [`src/prompt`](src/prompt/disposition-classifier.prompt.txt)) | | Edit and test in AI Builder, or edit the source file and rebuild |

Edit the web resource or the flow in the maker portal, then publish. Details in [Section 13](docs/DEPLOYMENT-GUIDE.md#13-configuration).

---

## Troubleshooting

| Symptom | Likely cause and fix |
|---------|----------------------|
| The modal never opens | Handler not registered or not published (install steps 7–8). Open the browser console (F12) and look for `[ccaasfde dispo v2.2]`. `CONFIG.ENABLED` may be `false`, the workstream may not be in the allowlist, or the workstream has no disposition codes |
| The modal opens twice | An old copy of the script is still cached. Hard-refresh (Ctrl+F5). Fixed in this version for script reloads |
| Spinner never ends / blank modal | Creator Kit Core is not installed, or the flow is off. Check Cloud flows → Disposition Recommender → *On* and the flow's run history |
| "No AI recommendations available" every time | Copilot summary is not being generated, or no code matches. Check `msdyn_conversationinsights` and the flow's run history (a run that polled to its limit found no summary) |
| Flow shows *connection not authorized* | Open Solutions → Connection references and fix the Dataverse connection |
| Save does nothing | Select at least one code. Check the *Conversation disposition code map* table and the user's privileges on it |
| Form lost another handler after import | The import overwrote the Active Conversation form. Re-add the handler and import without *Overwrite unmanaged customizations* next time |

More in [Section 18](docs/DEPLOYMENT-GUIDE.md#18-troubleshooting). Rollback steps are in [Section 19](docs/DEPLOYMENT-GUIDE.md#19-rollback-procedure).

---

## What changed from the original

Relative to the `1.0.0.2` tool in `microsoft/d365cc-fde`:

- **Flow:** Try/Catch so the app always gets an answer; input validation (GUID, non-empty code list with `dispositionId`) before any AI call; waits for a non-empty summary (bounded to 2 minutes); model output is parsed and validated (ids must exist in your list, confidence must be a number in 0–1, max 3).
- **Prompt:** prompt-injection guard, strict JSON output, few-shot examples. Source is in `src/prompt/`.
- **Canvas app:** test mode removed; flow or parse errors fall back to manual selection; search filters as you type; **new clickable code list when the AI has no recommendation**; error messages are shown instead of an endless spinner; fixed a hang caused by `Set(x, IfError(Flow.Run(...)))`.
- **Form handler v2.2:** retry after a failed open, form refresh after save, kill switch and workstream allowlist, skips the modal when there are no codes, no duplicate callbacks when the script reloads.
- **Solution:** removed forced workspace app settings, copied out-of-box event handlers/role maps and 37 stale dependencies.
- **Tooling:** dependency-free build script, 58 automated tests, GitHub Actions CI.

## Known limitations

- **Save from the canvas page** is the original formula, unchanged. A rewrite did not save during testing and was not shipped. Verify rows are created (Test it, step 4).
- The solution is **unmanaged** and carries the Active Conversation form (see the warning above).
- The flow runs as the connection owner, so it cannot enforce an agent's own access to a conversation.
- Only the first 500 active disposition codes are loaded. The full list is in [Section 17](docs/DEPLOYMENT-GUIDE.md#17-known-limitations).
- Not covered by the automated tests: import into Dataverse, Power Fx and Power Automate evaluation, AI quality, and live CIF events. Run the smoke test in a sandbox before production.

---

## Build from source

Needs Node.js 22+. No npm dependencies. The Power Platform CLI is not needed to build.

```bash
npm test         # 58 tests: handler (mocked Xrm/CIF), flow definition, prompt, solution package, canvas formulas
npm run build    # sync src/ into solution/ and rebuild AIDispositionCode_<version>.zip
npm run check    # fail if solution/ or the zip is out of sync with src/
```

Layout: `src/` (editable sources), `solution/` (unpacked solution), `scripts/` (build and zip), `tests/`, `docs/`. Details in [Section 21](docs/DEPLOYMENT-GUIDE.md#21-building--testing).

## License and credits

MIT. Original solution by Karthik Mallaya in [microsoft/d365cc-fde](https://github.com/microsoft/d365cc-fde); modifications by moliveirapinto. This is a community project and is **not an official Microsoft product**. See [LICENSE](LICENSE).
