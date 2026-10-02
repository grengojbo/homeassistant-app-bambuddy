## 1.2.5.7

**Bambuddy 1.2.5.7**

**What this is**

A feature release on top of 1.2.5.6. 25 new features, four changes, 40 fixes and four dependency advisories closed. 29 community PRs were merged, and 28 people are credited.

The headline is **post-print outcome confirmation**. Until now a print that completed counted as a success, even if the part was warped or the wrong colour. A print can now ask whether it came out good, and the answer can come from the web UI, the printer card, a notification, or a 👍 / 👎 on the Telegram message. The File Manager gets the most new features: a Column view, larger previews with zoom and fullscreen, image previews, PDF thumbnails made on the server, notes, links and photos on library files, and **Combine to 3MF** for putting several STLs on one plate. Inventory gains material numbers and a managed supplier list. SSO logins can keep Bambuddy groups in step with the identity provider. Other applications can now sign people in with their Bambuddy account and send messages through your notification channels. Bambuddy itself can now show announcements from the maintainers.

Several of the fixes are notifications that had a toggle but never fired: Low Filament, Reorder Alert and Stock Break Alert all send now. Read **Before you update** below: some alerts will fire right after the upgrade.

New tables and columns are added automatically on both SQLite and PostgreSQL. There are no breaking API changes, but HMS fault `severity` values change; see below.

If you are coming from 1.2.5.5 or earlier, read the 1.2.5.6 release notes too. If you are coming from 1.2.5 or earlier, read the 1.2.5 notes first; all of their upgrade callouts apply to you as well.

**Before you update**

These change behaviour you may rely on. None of them needs action unless it applies to your setup.

- **Notifications that never fired now do.** Low Filament (#2913), Reorder Alert and Stock Break Alert (#2955) had toggles on every provider, but nothing ever sent them. If you switched Low Filament on in the past, expect one alert for each assigned spool that is already low right after updating, and again after each restart while it stays low. Stock alerts fire once per SKU when it reaches its reorder point or will run out before a reorder could arrive.

- **HMS faults use the printer's own level (#2728).** The status response, WebSocket, Camera Wall and MQTT relay carry the new `severity` values, so anything that reads `severity` will see different numbers for the same fault. Printer-error notifications now also go out for `hms[]` faults that have a description, which almost never happened before, so you may get more of them.

- **The queue starts jobs in the order the queue page shows (#3200).** Jobs pinned to a printer and jobs queued for "Any <model>" used to be ordered separately, and which one won a printer depended on the database. If your queue mixes both, jobs may now start in a different order than you are used to, namely the one on screen.

- **Spoolman: each spool's own size is used (#3194).** Editing **Label Weight** now changes only that spool, not its filament. **Cost per kg** is converted at the spool's size, both ways. For 1000 g spools nothing changes. Spools that an earlier version reset with **Reset usage to 0** had their remaining weight overwritten in Spoolman and need re-weighing once (#2906).

- **Announcements from the maintainers.** Bambuddy now fetches one signed file from the public `maziggy/bambuddy-notifications` repo on GitHub, at startup and every 6 hours. Nothing about your install is sent. Administrators see them by default. **Settings → General → Updates** can show them to every user, or turn them off entirely, in which case nothing is fetched.

- **PostgreSQL: the default connection pool is now 80 instead of 100.** A stock PostgreSQL allows 97, so every install without its own `DB_POOL_SIZE` / `DB_MAX_OVERFLOW` logged a warning at startup. If you raised the server's `max_connections` and want the old ceiling, set `DB_MAX_OVERFLOW=80`.

- **Telegram reactions need a bot of their own (#3129).** If you choose **Reaction (👍 / 👎)** as a Telegram provider's verdict mode, Bambuddy polls that bot for reactions. Another app polling the same bot, such as Home Assistant's Telegram integration, stops receiving its messages. In a group, the bot must be an admin to see reactions.

- **A printer added by discovery with the wrong model keeps it.** The P-series code table was shifted, so a discovered P1S was saved as a P1P, a P1P as a P1S, and an X1E as a P2S. New printers are saved correctly. For one already saved wrong, pick the right model under **Model** in **Edit** from the printer card's menu.

- **Native installs get one new Python dependency**, `pypdfium2`, for PDF thumbnails. The update script and the manual `pip install -r requirements.txt` below install it; it bundles its own binary, so no system package is needed.

---

**How to update**

**Docker**

```bash
docker compose pull
docker compose up -d
```

**Native install - recommended path**

```bash
sudo BRANCH=main /opt/bambuddy/install/update.sh
```

**Native install - manual path**

```bash
sudo systemctl stop bambuddy
cd /opt/bambuddy
sudo -u bambuddy git fetch --prune --tags --force origin
sudo -u bambuddy git checkout main
sudo -u bambuddy git reset --hard origin/main
sudo /opt/bambuddy/venv/bin/pip install -r requirements.txt
cd frontend && sudo npm i
sudo systemctl start bambuddy
```

**Windows install**

Download `bambuddy-1.2.5.7-windows-x64-setup.exe` from this release page (or the unversioned `bambuddy-windows-x64-setup.exe` alias). Existing Windows installs upgrade in place via the in-app Install Update flow.

---

**New**

**Print outcome:**

- **Post-print outcome confirmation** (#1898, requested by @FedericoPuntelli, contributed by @Thomansky in #3047) - turn on **Ask for Outcome** in the print dialog, or as a default under Settings → Workflow, and Bambuddy asks "How did your print come out?" with the finish photo and **Good** / **Reject**. You can answer in the web UI, on the printer card, or from a notification: ntfy and Telegram get buttons, every other channel a link. A reject takes an optional reason and offers **Print again**. Unanswered prints get an **outcome?** badge and an **Unconfirmed** filter on the Archives page.
- **Answer the outcome prompt with a Telegram reaction** (#3046, requested and contributed by @Thomansky in #3129) - a 👍 or 👎 on the message records the verdict. The phone only talks to Telegram, so it works away from home and needs no external URL. See **Before you update**.

**File Manager:**

- **Column view** (#3020, requested and contributed by @Thomansky in #3190) - a third view mode next to Grid and List, as in the macOS Finder: one column per folder level, with full keyboard navigation and the same actions as the list view.
- **Larger previews with zoom and fullscreen, and image previews** (#2976, requested and contributed by @Thomansky in #2990) - PDF, spreadsheet and 3D previews share one large window with a fullscreen button. PDFs zoom with Ctrl/⌘ + wheel, pinch or keys. PNG, JPG, GIF, WebP and BMP get a preview with zoom and pan. Double-click a file to open its preview.
- **PDF thumbnails as soon as the file arrives** (#2976) - page one is rendered on the server on upload, from ZIPs and from external folders. **Generate Thumbnails** backfills existing PDFs.
- **External link, notes and photos on library files** (#3077, requested by @SergioFuchs, contributed by @Thomansky in #3128) - **File details** opens the file's facts, an editable notes field, a link and a photo gallery.
- **Combine several STLs, or several copies of one, onto one plate** (#2999, requested by @Markus98, contributed by @adman234 in #3162) - select STLs, click **Combine to 3MF**, set the copies, and Bambuddy saves one 3MF with every object on one plate, optionally opening the Slice dialog with auto-arrange on.

**Inventory and spools:**

- **Material numbers** (#2870, requested and contributed by @Thomansky in #2994) - your own number per product, such as `15` for Bambu Lab PLA Basic, filled in automatically for new spools of the same product, with a column, filter, Bulk Edit and a **By Material Number** statistics widget.
- **Suppliers as a managed list** (#2988, requested and contributed by @Thomansky in #2996) - assign any number of suppliers to a spool, each with an article number and a quoted price per kg, mark where it was bought, and see consumption and cost **By Supplier** in Statistics.
- **Find a spool by its label number, and assign it to a slot from the spool** (#2978, requested and contributed by @pd81 in #2998) - `#3` finds only spool 3, and a spool without a slot has an **Assign Spool** button, which is also where scanning its QR code lands.
- **Choose what goes on a spool label, preview it, and save labels as PNG** (#2981, requested by @apizz) - a checkbox per line, a live preview, and PNG output at 203, 300 or 600 dpi for label printer software.
- **Ambient drying can wait for sustained humidity** (#2518, requested by @ryansouza, contributed by @M2ABRAMSTANK in #2895) - opening the AMS lid no longer starts a drying cycle of up to 12 hours. Off by default; 15 minutes when switched on.

**Queue and printing:**

- **Queue cards show the filament each job will print with** (#3132, requested and contributed by @bgrr74 in #3184) - colour swatches and names, the mapped slot and spool, and a yellow warning when a mapped slot has been emptied since.
- **Colour swatches for printer slots in the Print / Schedule dialog** (#3159, requested by @frantiseklorenc) - each slot shows its actual colour and hex, so a real colour mismatch can be told apart from two names for the same colour.
- **A batch can record the external order it fulfils** - `external_source` and `external_ref` on `POST /queue/batches`, unique together, so an integration that retries can't queue the same order twice.

**Notifications and camera:**

- **Camera snapshots reach more notifications and more providers** (#3089, requested and contributed by @bbbenji in #3199) - Plate Not Empty and AI Failure Detection carry a photo; Home Assistant, Bark and Slack-format webhooks get photos too, with an **Attach Photo** switch per provider.
- **Other applications can send messages through your notification channels** - a per-provider **Messages from connected apps** switch, off by default, and an API key with the new **Send Notifications** permission.
- **The streaming overlay can show the printer model** (#3080, requested and contributed by @adamspicedev in #3099 and #3134).
- **A second streaming overlay design, for portrait and landscape sources** (#3177, requested and contributed by @adamspicedev in #3183) - **Artwork: Version 2**; Classic stays the default.

**Accounts, integrations and the app itself:**

- **SSO group sync** (#3107, requested and contributed by @willuhmjs in #3122) - each OIDC provider gains a **Group Claim** and a **Group Mapping**, and the mapped Bambuddy groups follow the identity provider on every login, the same rule as LDAP group mapping.
- **Connected apps** - other applications can sign people in with their Bambuddy account (OAuth 2.0 authorization code with PKCE), registered under Settings → API Keys → Connected Apps. Requires authentication to be enabled.
- **Announcements from the Bambuddy maintainers, inside Bambuddy** - security fixes, breaking changes, releases and calls for testers, signed and fetched from GitHub. See **Before you update**.
- **An app shown in the sidebar can match Bambuddy's theme, and open a Bambuddy page in place** - the framed page is told the theme, and can ask Bambuddy to open one of its own pages. Both only for the link's own origin.
- **The API-key printer status carries layers, the job id, HMS faults and the serial** (#2919, requested by @simplytoast1) - additive; existing fields are unchanged.

**Changed**

- **The Slice dialog uses more of a large screen** - up to 1536 px wide, with a left column that grows with it.
- **A STEP preview says it is converting, and for how long** (#2976) - a large STEP file can take over a minute in the browser, and now shows a running counter instead of a spinner that looked stuck.
- **Leftover Web Push code and the unused `pywebpush` dependency are gone** (#3171).
- **The frontend build no longer warns about `path` and `crypto` for the STEP previewer** (#2976).

**Fixes**

**AMS slots, K profiles and drying:**

- AMS slots that lost their K profile after a printer restart stayed on the default K, and queued jobs printed with it (#3219). Bambuddy now restores a lost selection while the printer is idle and checks again right before a queued job is sent.
- A slot that read empty for one status update lost its spool assignment for good (#3186, reported by @Sawtaytoes).
- Orca Cloud filaments assigned to an AMS slot showed up in OrcaSlicer as Generic (#3216, reported by @mrnoisytiger).
- A drying command the AMS never started showed as an active cycle and could hold the queue forever (#2896, contributed by @M2ABRAMSTANK in #3096).
- Re-reading a slot's RFID reported success when the printer refused, and older firmware now gets the command it understands (#3206, reported by @Sawtaytoes).
- Filament printed from the external spool was deducted from an AMS spool (#3166; also #2880).

**Spools, inventory, Spoolman and SpoolBuddy:**

- In Spoolman mode, a spool's own size was ignored in favour of its filament's, so a 250 g spool was charged as if it held 1000 g (#3194, reported by @worried-networking).
- In Spoolman mode, weighing ignored the vendor's empty spool weight (#3195, reported by @worried-networking).
- A Spoolman spool's own empty spool weight was never saved (#2908, contributed by @ojimpo in #3011).
- "Reset usage to 0" on a Spoolman spool set it back to full (#2906, contributed by @ojimpo in #2939).
- A new Bambu Lab roll in Spoolman mode was linked to another product line of the same colour (#2907, contributed by @ojimpo in #2944).
- SpoolBuddy showed multi-colour and effect spools as one flat colour (#3033, reported by @Sawtaytoes).
- "Print labels…" ignored the spools you had ticked (#2980, reported by @apizz).

**Queue and dispatch:**

- A printer that refused uploads emptied the print queue: one P2S failed 43 jobs in ten minutes (#3210, reported by @Leander-Vh). Those jobs now stay queued and the printer is retried with a growing wait.
- The queue did not start jobs in the order shown when pinned and "Any <model>" jobs were mixed (#3200, reported by @bgrr74).
- A queued file whose only plate isn't plate 1 hung the printer (#2947, contributed by @sgiffhorn in #2951).

**Notifications and HMS:**

- The Reorder Alert and Stock Break Alert notifications were never sent (#2955, contributed by @ojimpo in #3196), and their toggles could never be turned on (#2945, regression tests contributed by @ojimpo in #2956).
- The Low Filament notification never fired (#2913, contributed by @ojimpo in #2940).
- Progress milestones sent 75% at the start of a print, and then never sent 25% or 50% (#3211, reported by @BurgerKerman).
- The `{finish_photo_url}` link in a notification failed when authentication was on.
- HMS faults now show Bambu's own description and the right level, and faults without published text no longer vanish (#2728, reported by @gzimbric).
- A print the printer's AI camera stopped for spaghetti was archived with no failure reason (#2946, contributed by @ojimpo in #2954).

**Camera and stream overlay:**

- A built-in camera could show the same old picture for hours while reporting a healthy stream (#3218, reported by @adamspicedev; also #3189, reported by @Sawtaytoes).
- A live camera view stopped for good after about half an hour on X1, H2 and P2 printers.
- The stream overlay's camera stayed frozen until its browser source was refreshed (#3205, contributed by @adamspicedev in #3213).
- Stream overlay and Cam Wall links stopped working after a reload, and signed the browser out (#3204, contributed by @adamspicedev in #3212).

**Login and directory:**

- LDAP group mapping found no groups on lldap and OpenLDAP, and StartTLS never worked with Active Directory or Samba AD (#3197, reported by @TOFM).
- Remember Me disappeared from the login page when only SSO sign-in was allowed (#2784, contributed by @vuthanhtrung2010 in #3117).

**File Manager and interface:**

- PDF previews failed on any browser older than Chrome 145 or Firefox 144 (#2976).
- The File Manager stopped 64 px short of the bottom of the window (#3215, reported by @koder-guy).
- Number fields could not be cleared and retyped (#3182, reported by @Carter3DP).
- Settings tabs lost their icons when a label was long (#3191, contributed by @Thomansky in #3192).
- Reading a 3MF's details loaded all of its geometry into memory.

**Install, platform and database:**

- Installing or updating ran out of memory on 2 GB machines, such as a standard Proxmox LXC or a 2 GB Raspberry Pi (#3181, reported by @PhilippeP62).
- PostgreSQL installs warned at every start that the connection pool may exceed the server. See **Before you update**.
- Turning off "Check printer firmware" did not stop every firmware lookup.
- Saving an empty value for a number setting through the API broke the Settings page until the database was fixed by hand. An affected install now recovers on its own.
- A P1P, P1S or X1E added by discovery was saved as the wrong printer.
- A connection check that overran in the support bundle discarded everything it had found (#3164).

**Security (dependencies)**

- `PyJWT` to 2.15.1 and `urllib3` to 2.8.0. Bambuddy's session tokens and SSO sign-in were not open to the PyJWT issues; the fixes for malformed tokens and key sets do reach SSO sign-in.
- `dompurify` to 3.4.16, for a low-severity advisory in a mode Bambuddy does not use.
- `js-yaml` to 5.4.2 and `brace-expansion` to 5.0.12, both development-only; neither is in the shipped image.

**Merged community PRs in this release**

- #2895 by @M2ABRAMSTANK - sustained humidity for ambient drying.
- #2939, #2940, #2944, #2954, #2956, #3011, #3196 by @ojimpo - Spoolman reset, empty-weight and product-line fixes, Low Filament and stock alerts, AI spaghetti failure reason.
- #2951 by @sgiffhorn - queued files whose only plate isn't plate 1.
- #2990, #2994, #2996, #3047, #3128, #3129, #3190, #3192 by @Thomansky - File Manager previews, material numbers, suppliers, outcome confirmation, library file details, Telegram reactions, Column view, Settings tabs.
- #2998 by @pd81 - spool ID search and assignment from the spool.
- #3096 by @M2ABRAMSTANK - parked AMS drying commands.
- #3099, #3134, #3183, #3212, #3213 by @adamspicedev - printer model and Version 2 design for the stream overlay, overlay links and camera recovery.
- #3117 by @vuthanhtrung2010 - Remember Me with SSO-only login.
- #3122 by @willuhmjs - SSO group sync.
- #3162 by @adman234 - Combine to 3MF.
- #3184 by @bgrr74 - filament on queue cards.
- #3199 by @bbbenji - camera snapshots for more notifications.

**Thanks**

To everyone who requested, reported, captured logs, or contributed: @adamspicedev, @adman234, @apizz, @bbbenji, @bgrr74, @BurgerKerman, @Carter3DP, @FedericoPuntelli, @frantiseklorenc, @gzimbric, @koder-guy, @Leander-Vh, @M2ABRAMSTANK, @Markus98, @mrnoisytiger, @ojimpo, @pd81, @PhilippeP62, @ryansouza, @Sawtaytoes, @SergioFuchs, @sgiffhorn, @simplytoast1, @Thomansky, @TOFM, @vuthanhtrung2010, @willuhmjs, @worried-networking.

---
**Sponsors**

Bambuddy is sustainable thanks to people who put their money where their use is. If this release saved you time or kept your farm running, the project runs on recurring contributions - there's no paid tier, no telemetry, no upsell, just sustainable maintenance.

- GitHub Sponsors (recurring, 5 tiers from $5/mo to $300/mo) - https://github.com/sponsors/maziggy
- Ko-fi (one-time or recurring) - https://ko-fi.com/maziggy

