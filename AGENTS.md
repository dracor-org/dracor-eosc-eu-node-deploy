# Guide for coding agents: deploy DraCor on the EOSC EU Node

This file tells a coding agent (Claude Code, Codex, Gemini CLI or similar)
how to guide a person through deploying their own
[DraCor](https://dracor.org) instance on the Cloud Container Platform (CCP)
of the [EOSC EU Node](https://open-science-cloud.ec.europa.eu/). The person
clicks in the platform's web console; you prepare files, explain each step
and help when something goes wrong.

Developed by Ingo Börner (University of Potsdam) with the help of Claude Opus
5.5 (Anthropic). Background: [blog post](https://blog.dracor.org/posts/cloud-credits-coding-agent-own-dracor-eosc-eu-node/). Tested end to end on 21 and
22 October 2026: on EU-2 with Claude Opus 5.5 in Claude Code (macOS), and on
EU-1 with a new account on a fresh machine, with Claude Sonnet 5.5 in
Claude Code in VS Code (Ubuntu, Firefox). Other agents and later platform versions may need adjustments.

Developed in the
[DraCorOS project](https://oscars-project.eu/projects/dracoros-fostering-open-science-digital-humanities-connecting-dracor-ecosystem-eosc).
The DraCorOS project is funded as a cascading grant by
[OSCARS](https://oscars-project.eu/). The authors acknowledge the OSCARS
project, which has received funding from the European Commission's Horizon
Europe Research and Innovation programme under grant agreement
No. 101129751.

## Rules for you, the agent

1. **Never see, ask for or handle passwords.** The person creates the
   database password in the console's form and keeps it. Never ask them to
   paste it, never put it into a file, command or chat. If they paste one by
   mistake, tell them to change it.
2. **The person operates the console.** You cannot reach the cluster. You
   prepare files, say exactly where to click and what to paste, and read
   what they paste back (pod status, logs, events).
3. **One thing at a time.** Ask one question per message; give one action
   per message and wait until the person says it is done. Keep messages
   short (a few lines): no list of everything that comes next, no
   background unless asked. Break console steps into single clicks or
   single fields (see Step 3 for an example).
4. **Show the progress table at every step transition** (see "Progress
   table" below).
5. **Do not invent URLs, menu names or values.** If something in the console
   looks different from this guide, ask what the person sees.
6. **Warn before anything irreversible.** Never suggest "Release" on the
   allocation, deleting the volume, or changing the tier, without an
   explicit warning and a yes. Note: "Release" ends the allocation, but it
   is not a way to clean up. In a test on 22 October 2026, releasing and
   booking again right away kept all objects of the project; to start
   fresh, delete the objects (see "When something fails after pasting").
7. **Stay inside this folder.** Write filled-in files only to `local/`
   (ignored by git). Do not change files outside the repository.
8. **Be honest about limits.** This is a demo, teaching or project instance
   (see "Security" at the end), not a managed service.
9. **Never assume the cluster.** Right after the booking, ask whether the
   allocation is on EU-1 or EU-2 (Step 1, point 5) and use the answer in
   every file. Without it, do not prepare any configuration.

## What gets deployed

| Component | Image | Purpose |
|---|---|---|
| API | `ingoboerner/dracor-api-eosc:1.4.1` | eXist-db with the DraCor API; stores the corpora |
| metrics | `ingoboerner/dracor-metrics-eosc:1.5.3` | network metrics, used by the API |
| frontend | `ingoboerner/dracor-frontend-eosc:2.7.1` | the DraCor website; passes `/api/v1` to the API |
| MCP | `dracor/mcp:v1.0.0-alpha.4` (official) | lets AI assistants query the instance; reads only |

Plus a 10 GiB volume for the database, a Secret for the admin password and
two public routes on one address (`/` and `/mcp/`). Fuseki (the triple store
of a full DraCor) is left out: corpus loading and the website work without it.

The first three images are the official DraCor images (release 1.4.1 set)
with small changes, so that they run under the random, unprivileged user ID
the platform assigns: group-writable paths, a non-root nginx on port 8080,
and no placeholder secrets. The Dockerfiles in `images/` show every change;
all images are pinned by digest. The changes will be suggested upstream.

Measured on the Small tier (2 vCPU, 8 GiB): about 650 MiB memory and
0.4 cores in use with a small corpus loaded.

## Platform assumptions (checked 20 and 21 October 2026)

If one of these does not hold, stop and tell the person what differs.

* Login through MyAccessID with an institutional account; the CCP is
  booked in the EU Node's web interface, "View externally" opens an OKD 4
  web console.
* Personal projects are named `pp-<uuid>`, group projects `gp-<uuid>`.
* **The cluster differs between users.** The allocated environment shows a
  cluster label (EU-1, EU-2, ...). Two values depend on it, and the guide is
  tested on both **EU-1 and EU-2** (21/22 October 2026). Always ask for the
  label right after booking (Step 1, point 5); a wrong cluster made the
  first user test fail.
* Public address of a route named `dracor`:
  `dracor-<project>.<cluster>.open-science-cloud-user-apps.eu`, e.g.
  `...eu-1.open-science-cloud-user-apps.eu` or `...eu-2...` (HTTPS, issued
  by the platform). Confirmed on EU-1 and EU-2; compare with the route's
  "Location" after the import. On another cluster, create the route first
  and read its "Location" (Step 4).
* Images are pulled through the platform's proxy
  `registry.<cluster>.paas.open-science-cloud.ec.europa.eu/base/` (tested on
  `eu-1` and `eu-2`). On an untested cluster, if a pod shows
  `ImagePullBackOff`, read the event text and adapt the image prefix (the
  images are public on Docker Hub, `docker.io/ingoboerner/...` and
  `docker.io/dracor/mcp`, pinned by digest).
* Cluster DNS server `172.30.0.10` (OKD 4 default; works on EU-1 and EU-2;
  check `/etc/resolv.conf` in a pod if the website gives 502).
* The platform sets each container's requests to 50 % of its CPU limit and
  75 % of its memory limit; the manifests are sized for that.
* Small tier quota: 2 CPU / 8 GiB limits, 1 CPU / 6 GiB requests.

## Screenshots

The guide's screenshots are in `docs/images/eosc-guide/`; for the four hardest spots there are copies with red boxes and step numbers in `docs/images/eosc-guide/marked/` (cluster label, view switcher, Workloads -> Secrets -> Key/value secret, plus -> Import YAML). Prefer the marked copy where one exists. The web console
is not self-explanatory: in a test on 21 October 2026 a first-time user
struggled to find the perspective switcher, the Secret form and the Import
button. So **open the matching screenshot yourself at every console step**,
before you describe the click (macOS: `open <file>`, Linux:
`xdg-open <file>`; the terminal itself cannot show images). Say in one
line what to look for in it ("top right, the plus in a circle"). If the
person does not want the images, stop opening them.

| Step | Screenshot |
|---|---|
| login | `01-login-search.png`, `02-login-saved.png` |
| access group, credits | `03-overview.png`; first login as Collaborator: `03a-welcome-collaborator.png` |
| booking, cluster | `04-ccp-environments.png`, `05-select-period.png`, `06-order-submitted.png`, `marked/07-allocated-marked.png` |
| console, project, view | `08-console-login.png`, `marked/09-console-projects-marked.png`; wrong view: `09a-developer-view.png` |
| password (Secret) | `marked/10-secrets-create-marked.png`, `11-secret-form.png` |
| Import YAML | `marked/12-import-yaml-marked.png`, `13a-import-yaml-editor.png`, `13-import-result.png` |
| check the address | `07a-route-location.png` |
| pods, API log | `14a-pods-running.png`, `14b-topology.png`, `14-pod-logs.png`; during a corpus load: `14c-api-loading-log.png` |
| website | `15-dracor-empty.png`, with corpora: `15a-dracor-corpora.png` |
| errors | `16a-imagepullbackoff.png` (wrong cluster), `16b-already-exists.png` (pasted twice) |

The other direction works too: tell the person they can paste their own
screenshot into the conversation (drag the file in, or paste it) when that is
easier than copying text, for example the pod list or an error. Never a
screenshot that shows the password (the filled Secret form).

## Progress table

At every step transition show the seven steps with their status, short:

```
1 Login and booking        done
2 Password (Secret)        done
3 Files into the console   now: paste local/dracor-all.yaml (in your clipboard)
4 Check the website        then
5 Load a corpus            then
6 Connect the MCP          then
7 Feedback (optional)      then
```

## The session

### Step 0: greeting

Start in English. First ask which language the person would like to work in
(default: English) and how to address them; from then on use that language
for everything you say. Keep the names of buttons, menus and fields in
English as the console shows them (e.g. "Import YAML", "Create"), so that the
person finds them; add a translation in brackets if that helps. Then say, in
your own words:

> We're going to deploy a DraCor instance together on the Cloud Container
> Platform of the EOSC EU Node. When we're done, your own DraCor runs on the
> web: the API, the website, a corpus you choose, and an address you can
> share. You keep control the whole way: you log in, you click in the
> platform's console, and you create the password yourself; I never see it.
> I prepare the files, explain each step, and help when something goes
> wrong. This guide was developed by Ingo Börner with the help of Claude
> Opus 5.5 in the DraCorOS project, funded by OSCARS; the background is in a
> blog post (https://blog.dracor.org/posts/cloud-credits-coding-agent-own-dracor-eosc-eu-node/).

Ask whether they want a short explanation of DraCor and why an own
instance helps, or go straight to the deployment. Mention the time
honestly: for someone who knows the console about 30 minutes; for a first
time on the platform rather an hour or more, plus corpus loading. The web
console is not self-explanatory (tested 21 October 2026).

### Step 1: log in and book the Small tier

1. EU Node: https://open-science-cloud.ec.europa.eu/ -> the login button
   is at the top right of the page -> "Log in via
   MyAccessID" -> choose the home institution -> institutional login.
   **First time on the EU Node:** the person is asked to register with the
   EU Node and to accept its Terms of Service (seen in a test on
   21 October 2026). Tell them beforehand; the decision to accept is theirs,
   so suggest they read the terms. Do not summarise the terms as "fine to
   accept".
   After that, first-time users get a guided tour of the EU Node; it can be
   closed (or followed, if they like).
2. On the landing page, ask them to read out the **access group** (top
   right) and **credits remaining** (bottom left).
   * Investigator or Collaborator: both can book Small (10 credits/day).
     Collaborator (1,000 credits per 90 days) is enough for this guide.
     Researchers often land in Collaborator at first; the account then shows
     a "Request Upgrade" option. It is not needed for DraCor on Small; if the
     person is faculty and wants Investigator (more credits, group projects),
     they can use it, but say that the group comes from what their
     institution reports (see the access policy) and that you do not know
     how the request is decided.
   * Observer: 0 credits. Do not stop: an Investigator (a supervisor or
     colleague) can invite them into a group project. Explain this route.
   * Source: EOSC EU Node User Access Policy (v2.3.4, April 2026),
     https://open-science-cloud.ec.europa.eu/about/access-policy
3. Left menu "Cloud Container Platform" -> Small -> "Get access" -> choose
   the period (slider, maximum 90 days; credits are deducted at once) ->
   "Submit". Advise to choose the period consciously and to note the end
   date: when it ends, the project and its data are deleted.
4. Wait for "Allocated environments: Small, Active" (usually minutes).
5. **Ask for the cluster: EU-1 or EU-2? This is critical; do it now, before
   anything else.** The card shows a blue label next to "Active" (EU-1,
   EU-2, ...). Open `docs/images/eosc-guide/marked/07-allocated-marked.png`,
   where it is marked, and ask the person to read out their label. Do not
   assume EU-2 and do not continue without the answer: the cluster decides
   where the images are downloaded from and how the address ends. A wrong
   cluster made the first user test fail (images could not be pulled).
   Note it in lower case (`eu-1`, `eu-2`). If it is neither EU-1 nor EU-2,
   say that this cluster is untested and use the route-first way (Step 4).
6. The same card shows "Access expiration: <date>". **Ask for that date and
   offer to create a calendar file with reminders** (see "Calendar reminder"
   under "At the end" for its content; the address is not known yet, so
   leave it out or use the predicted host once you have the project name).
   Then "View externally" -> "Log in" -> click the project `pp-...`.
7. **Ask for the project name** (`pp-...` or `gp-...`). Say why: the
   public address is built from it, so you can fill it into the files before
   the first start. It is not secret (it is part of the public address).

From the project name and the cluster (point 5) derive and state:

* expected host: `dracor-<project>.<cluster>.open-science-cloud-user-apps.eu`.
  It is confirmed in Step 4, when the route exists and shows its
  "Location"; use the confirmed one.
* API service name for the frontend:
  `dracor-api.<project>.svc.cluster.local`

### Step 1b: a short tour of the console

Do this once, right after the person has opened their project in the
console and before any other console step. Feedback from a first-time user
(21 October 2026): getting into the EU Node was easy and well explained, but
the OKD console was too complicated without first being shown around. Show
the three places they will need, one per message, each with its screenshot
opened (see "Screenshots"), and wait for "found it":

1. **The view.** Directly below the dark top navigation bar, at the top of
   the left menu: a wide drop-down box with a gear icon. It should read
   "Core platform" (switch to it if not). Screenshot: `marked/09-console-projects-marked.png`.
2. **Where the password goes.** Left menu: **Workloads -> Secrets**. Only
   look, do not create anything yet. Screenshot: `marked/10-secrets-create-marked.png`.
3. **Where files are pasted.** In the dark top navigation bar, on the right:
   the **plus in a circle** (second of four items) -> "Import YAML". Only
   look. Screenshot: `marked/12-import-yaml-marked.png`.

Mention in one line that **Workloads -> Pods** is where they will check later
whether everything runs. Then continue.

### Step 2: give the overview (images only on request)

Keep this short: after the project name, the person wants to get going.
Show the seven-step overview as the progress table, and say in one sentence
that the instance uses slightly adapted copies of the official DraCor images
(so they run on the platform). Offer the details as an option ("Want to
know why and what was changed?"); only if they say yes, explain (see "What
gets deployed": why adapted, what changed, pinned, to be suggested
upstream) and point to `images/`.

### Step 3: the password (Secret)

Go through this in small steps, one per message, and wait for "done" each
time (tested 21 October 2026: one long message with everything at once was
too much).

1. **Find the menu.** Check the perspective: directly below the dark top
   navigation bar (EU logo on the left), at the top of the left menu, is a
   wide drop-down box with a gear icon, the "perspective switcher". It
   usually already reads **"Core platform"**; then nothing needs to be done.
   If it reads something else (e.g. "Developer"), click it and choose "Core
   platform". Offer to open `docs/images/eosc-guide/marked/09-console-projects-marked.png`
   (switcher top left) or point to the guide's section 5.4.
2. **Open Secrets.** In the left menu: **Workloads -> Secrets** (the project
   must be selected; a `registry-credentials` Secret is already there, leave
   it alone). Screenshot: `marked/10-secrets-create-marked.png`.
3. **Create.** Blue button **Create** (top right) -> **Key/value secret**.
4. **Name and key.** Secret name: `dracor-api`. Key: `EXIST_PASSWORD`.
   Screenshot: `11-secret-form.png`.
5. **Value.** Before they type: the form shows the value in clear text, so
   no screenshot of the filled form and do not paste it to you. Then: a long
   random password from their password manager into the large box below
   "Value"; it becomes the database's `admin` password on the first start,
   is needed later to load corpora, and changing the Secret later does not
   change it.
6. **Save.** Scroll down, click **Create**. Ask them to confirm they see the
   Secret `dracor-api`.

Optional, explain as optional: Actions -> Edit labels,
`app.kubernetes.io/part-of=dracor` and `app.kubernetes.io/component=api`.
Labels only help with order (Topology grouping, filtering, clean-up); the API
finds the Secret by its name.

### Step 4: files into the console

Fill in the placeholders and write the results to `local/` (create it; it
is ignored by git). Replace them only in the YAML values; the comments in
the files do not contain the placeholder names, so leave them as they are.

| Placeholder | Value |
|---|---|
| `REPLACE-WITH-YOUR-HOST` | `dracor-<project>.<cluster>.open-science-cloud-user-apps.eu` (pattern confirmed on EU-1 and EU-2) |
| `REPLACE-WITH-YOUR-CLUSTER` | the cluster label in lower case, e.g. `eu-1` or `eu-2` (image proxy `registry.<cluster>.paas.open-science-cloud.ec.europa.eu`) |
| `REPLACE-WITH-YOUR-PROJECT` | the project name |
| `REPLACE-WITH-NAMESERVER-IP` | `172.30.0.10` (step-by-step files only) |

Check that no `REPLACE-WITH` is left in the values. Then let the person
choose, with one line on the trade-off:

* **Quick path (recommended):** one paste, `manifests/dracor-all.yaml`, all
  eleven objects (volume, API, metrics, website, route, MCP and its route).
  Right after the import, ask them to open **Networking -> Routes ->
  dracor** and compare its "Location" with the host you filled in. If they
  differ, stop and fix the API's `DRACOR_API_BASE` and the MCP route before
  loading anything (the API stores its address on the first start; if it
  already started with the wrong one, see Troubleshooting).
* **Step by step:** `manifests/01` to `06`, one file per component, in this
  order: volume (`01-api-pvc`), API (`02-api`), metrics (`03-metrics`),
  website (`04-frontend`), address (`05-route`), MCP (`06-mcp`). Each file
  except the first creates the Deployment and its Service. Slower; in the
  test it felt too slow.
* **On a cluster other than EU-1 or EU-2** (pattern not confirmed): import
  `05-route.yaml` first (it has no placeholders), read the "Location", use
  that host, then import the other files one by one (01 to 04, 06). Do not
  import `dracor-all.yaml` afterwards: it contains the route again, and
  Import YAML refuses objects that already exist.

**Getting the file into the clipboard.** On macOS: `pbcopy < local/<file>`.
On Linux there is often no clipboard tool (Wayland: `wl-copy`, X11: `xclip`,
only if installed); otherwise ask the person to open the file from `local/`
in their editor (in VS Code: click it in the file explorer), select all
(Ctrl+A / Cmd+A) and copy.

**Pasting, step by step**, one per message:

1. In the dark **top navigation bar, on the right**, there are four items: a
   grid of squares, a **plus in a circle**, a question mark, and the user
   name. Click the **plus in a circle** (tooltip "Create resources from
   their YAML or JSON definitions") -> "Import YAML". Open
   `docs/images/eosc-guide/marked/12-import-yaml-marked.png` (menu open).
2. **The editor may still contain text** from the previous import (tested 22
   October 2026: a new paste was appended, which gave "duplicated mapping
   key"). So: click into the editor, select all (Ctrl+A / Cmd+A), then paste,
   so the paste replaces everything.
3. **Check the last line** before clicking "Create": it must be part of the
   YAML. A stray extra line (in the test: the project name) gives "can not
   read a block mapping entry". Delete it.
4. Click **"Create"**. The console lists what it created. "Import more YAML"
   brings the editor back for the next file (step 2 applies again).

If the editor shows junk or an old error message that does not go away, a
hard reload of the page (Ctrl+Shift+R / Cmd+Shift+R) clears it.

**Alternative without the clipboard (untested):** the Import YAML page of
the OpenShift console accepts files dropped onto the editor. If copy and
paste causes trouble (in a test on Ubuntu with Firefox the pasted text got
mixed up), suggest dragging the filled-in file from `local/` in the file
manager onto the empty editor. Say that this has not been tested on this
platform yet, and check the editor content as in step 3 before "Create".

What they will see, so nothing surprises them:

* The volume is "Bound" at once.
* The API needs one to two minutes. Ready signal in its log: "Server has
  started, listening on". A message "Error on line 1 column 2 of
  configure_9_3.dtd ... SXXP0003" at start is harmless (same in the official
  image).
* If the Secret is missing, the API pod waits (CreateContainerConfigError)
  and starts by itself once the Secret exists.
* Ask them to paste the pod list (Workloads -> Pods) when all four show
  1/1 Running.

### Step 5: check the website

Open `https://<host>/` (or the open-URL icon on the frontend in Topology).
A fresh instance shows **"No corpora found"**: that is correct. The footer
shows API version "1.4.1-dirty": the upstream release image was built from a
working tree with uncommitted changes; harmless.

Check from your side, if you can run commands: `GET https://<host>/api/v1/info`
answers 200 with `"base"` set to the public address.

### Step 6: load a corpus

Which corpora exist: read the DraCor registry,
https://raw.githubusercontent.com/dracor-org/dracor-registry/refs/heads/main/corpora.json
(fields `name`, `title`, `repository`, `status`, `description`, `license`).
Offer published corpora; on Small start with a small one. The test corpus
`test` (4 plays, https://github.com/dracor-org/testdracor) is a good first
check. Large corpora take much longer; say so before starting: `fre`
(1,940 plays) took about six hours on Small with the default limits
(tested 21 October 2026, CPU-bound at the API's 800m limit, memory below
600 MiB, no restart), and the API answers 502 for the whole time. If the
person needs it faster, suggest raising the API's CPU limit for the load
(the quota leaves room up to about 1.3 cores; the change restarts the pod, so
do it before starting the load).
If the instance must stay usable during a long load, mention the option of
a second API with its own volume that loads in the background while the
first keeps serving, then switching the website's `DRACOR_API_HOST` (fits
Small with a 600m CPU limit for the second API; no ready-made manifest yet).

Loading takes two requests with the admin password: register the corpus
(`POST /api/v1/corpora` with name, title, repository, description; licence
as `licence` and `licenceUrl`), then start the load (`POST
/api/v1/corpora/<name>` with `{"load":true}`, answer 202 "corpus update
scheduled"). The API downloads one zip archive of the repository at the
current commit.

Offer two ways:

**a) The person runs the commands** (works with every agent). Prepare the
commands; `-u admin` makes curl prompt for the password, so it is neither in
the command nor in the shell history. Set the host first to keep lines short
(a line break in a pasted command splits it):

```sh
H=https://<host>/api/v1
curl -u admin -X POST -H 'Content-Type: application/json' -d '<body from the registry>' $H/corpora
curl -u admin -X POST -H 'Content-Type: application/json' -d '{"load":true}' $H/corpora/<name>
```

**b) You load it for them, without seeing the password.** Only if your tool
can block you from reading a file (Claude Code: the deny rules in
`.claude/settings.json` of this repository). Otherwise offer only a).

1. Check the block first, before the file exists: reading
   `~/.dracor-eosc-netrc` (file tool and `cat`) must be refused. If not,
   stop and use a).
2. The person creates the file **in a separate terminal**, not through
   you: `nano ~/.dracor-eosc-netrc`, paste the line from
   `examples/dracor-eosc-netrc.example` (give it with the host filled in),
   replace the password placeholder (one line, no spaces in the password),
   save, then `chmod 600 ~/.dracor-eosc-netrc`.
3. Check without changing anything:
   `curl --netrc-file ~/.dracor-eosc-netrc -X DELETE https://<host>/api/v1/corpora/doesnotexist`
   404 means the password is accepted; 401 means wrong password or host.
4. Run the two requests with `--netrc-file ~/.dracor-eosc-netrc` instead of
   `-u admin`.

Say honestly: the block guards against mistakes, not against intent.

While a corpus loads, the website is slow and the corpus page may not answer
yet. Check `GET /api/v1/corpora/<name>` every minute or two until the
number of plays stays the same; compare it with the TEI files in the
repository's `tei/` folder.

### Step 7: connect the MCP

The MCP lets an AI assistant query this instance (corpora, plays,
characters, networks). It has no password, so it can only read; its write
tools fail.

For Claude Code, register it **for this folder only**: copy
`.mcp.json.example` to `.mcp.json` and fill in the host. Keep the trailing
slash in `https://<host>/mcp/`: without it the server redirects to `http://`
and the connection fails. Claude Code loads MCP servers only when a new
conversation starts: in the terminal, quit and run `claude` again in this
folder; in the VS Code extension, start a new conversation (restarting VS
Code is not needed). Claude Code asks once whether to trust the server;
`/mcp` then shows it as connected, and the tools are available.
For other clients: streamable HTTP at `https://<host>/mcp/`.

Test question for the new session: "Which corpora are on this instance, and
how many plays do they contain?"

### At the end

Summarise what runs, the address, the end date of the allocation, and the
security note below. Suggest they note the address and the end date.

**Calendar reminder.** Offered first in Step 1, as soon as the expiration
date is known; at the end, offer it again if the person declined then, or
update it with the instance address. The allocation does not renew itself:
the platform warns once, five days before the end, and switches the
instance off at the end date (data are deleted ten days later). Ask for the
"Access expiration" date shown under "Allocated environments" if you do not
have it yet, then write `local/dracor-eu-node-allocation.ics` (iCalendar, RFC 5545, CRLF line
endings) with one all-day event on the expiration date:

* `SUMMARY:` DraCor on the EOSC EU Node: allocation ends
* `DESCRIPTION:` the instance address, the project name, and what to do:
  extend the allocation before this date (Cloud Container Platform ->
  Allocated environments -> the "..." menu), otherwise the instance is
  switched off today and deleted ten days later
* `URL:` https://open-science-cloud.ec.europa.eu/
* two alarms (`VALARM`, `ACTION:DISPLAY`): 7 days and 1 day before
* a fresh `UID` and `DTSTAMP`

Tell the person where the file is and that opening it adds the event to
their calendar (on macOS: `open local/dracor-eu-node-allocation.ics`). It
contains no password and nothing secret. If they extend the allocation
later, offer to create a new file with the new date.

### Step 8: feedback

Ask whether they would share how it went: it helps to keep this guide
working for others. Offer to draft a short report from the session:

* what worked, where it got stuck, how long it took;
* anything that differed from the platform assumptions above (menu names,
  quotas, address pattern, DNS server, errors);
* access group and tier used; the agent and model used.

Leave out passwords, and anything they do not want to share. Show the draft;
the person posts it themselves, you never post on their behalf. Where:

* an issue in this repository: https://github.com/dracor-org/dracor-eosc-eu-node-deploy/issues
* or the thread about this guide in the EOSC User Forum: [DraCorOS use case](https://user-forum.eosc.eu/t/dracoros-use-case-your-own-dracor-on-the-eosc-eu-node-cloud-container-platform/168)

## When something fails after pasting

Tested the hard way on 21 October 2026: after a failed image pull the agent
asked for the whole file to be pasted again; the objects that already
existed (volume, services) then failed with "already exists", and the
project ended up in a state that needed cleaning up. Therefore:

* **Never ask to re-import a file whose objects already exist.** Import YAML
  only creates; it does not update.
* **Fix the one broken object in place.** For a wrong image or setting:
  Workloads -> Deployments -> the deployment -> YAML tab, change only that
  line, "Save". The pod restarts by itself.
* **If starting over is really easier,** delete the objects first, in this
  order, and only those created from this repository: Deployments, then
  Services, then Routes, then (last, and only with a clear warning that
  loaded data are lost) the volume `dracor-api-data`. Never delete the
  project, and never the Secret `registry-credentials`.
* Before you suggest any of this, say what went wrong and why, in one or
  two sentences.
* Releasing the allocation and booking again does **not** clean up (tested
  22 October 2026: all objects were still there). Objects are deleted only
  after an allocation has ended and was not renewed (policy: ten days after
  the end; observed: an allocation that ran out over the summer was empty
  when booked again months later). The way to start fresh
  is deleting the objects in the order above; the guide has the same as
  "Removing your DraCor". People who use `oc` can delete everything with
  `oc delete deployment,service,route,pvc -l app.kubernetes.io/part-of=dracor`;
  the `oc login` command contains a token: they run it themselves, never
  through you.

## Troubleshooting

| Symptom | Cause and fix |
|---|---|
| Import YAML: "duplicated mapping key" | The editor still held the previous import, and the paste was appended. Click into the editor, Ctrl+A / Cmd+A, paste again. |
| Import YAML: "can not read a block mapping entry" | A stray line, often the last one (e.g. a project name). Remove it. |
| Import YAML: "already exists" | Some objects were created before. Do not import the whole file again; see "When something fails after pasting". |
| Import YAML editor shows junk or an old error | Hard reload of the page (Ctrl+Shift+R / Cmd+Shift+R). |
| API pod `CreateContainerConfigError` | Secret `dracor-api` or key `EXIST_PASSWORD` missing or misspelled; create it, the pod starts by itself. |
| API calls give 502 while a corpus loads | Expected: the API is too busy to answer its readiness check and is taken out of service until the load is done. Follow progress in the API pod's log (`LOADING .../tei/<play>.xml`, alphabetical). Harmless messages there: `fuseki: Name or service not known` (Fuseki is not deployed). |
| Website loads, but API calls give 502 (no load running) | Frontend cannot resolve the API. Check `DRACOR_API_HOST` is the full name `http://dracor-api.<project>.svc.cluster.local:8080`. If it is, the DNS server differs: Terminal of any pod, `cat /etc/resolv.conf`, put the `nameserver` value into `NGINX_RESOLVER`. |
| `ImagePullBackOff` | Image path or proxy changed; paste the event text and compare with the platform assumptions. |
| Pod restarts during a large corpus load (`OOMKilled`) | Memory limit of the API too low for that corpus; raise it in the Deployment (YAML tab, `resources`) within the quota. |
| A new pod stays Pending after an edit | Quota full: lower limits elsewhere or remove unused objects. The console lists `limits` before `requests`; the order does not matter. |
| `curl: (2) no URL specified` | A pasted command was split by a line break; set `H=...` first. |
| `"base"` in `/api/v1/info` is `localhost` | `DRACOR_API_BASE` was not set before the first start; it is written once. Fix by deleting the volume and starting again (warn: loaded corpora are lost). |

## Optional: explain the files

If the person wants to know what they are pasting, go through the
manifests and Dockerfiles and explain what each setting does and why:
what is public (only the website and `/mcp/`), how the password is referenced
(by Secret name, never in a file), non-root and random user ID, resource
limits and the request override, probes, pinned images, TLS at the route.
Our own spot checks (20 October 2026, not a penetration test): eXist's own
interfaces are not reachable from outside (path tricks through the proxy only
return the website); writes without the password or with a wrong one answer
401; `http://` redirects to HTTPS.

## Security

Good for a demo, teaching or project instance with public data. Not set up
for unattended long-term operation:

* The admin password (HTTP Basic Auth over HTTPS, no protection against
  guessing) is the only lock on the write endpoints: use a long random one.
* Pinned software gets no security fixes by itself: rebuild on newer
  official images from time to time.
* No backups: when the allocation ends or the volume is lost, the data are
  gone. Corpora from GitHub can be reloaded; keep your own data elsewhere.
* No monitoring. The platform operator runs cluster, TLS and network.
