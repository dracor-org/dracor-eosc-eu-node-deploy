# Guide for coding agents: deploy DraCor on the EOSC EU Node

This file tells a coding agent (Claude Code, Codex, Gemini CLI or similar)
how to guide a person through deploying their own
[DraCor](https://dracor.org) instance on the Cloud Container Platform (CCP)
of the [EOSC EU Node](https://open-science-cloud.ec.europa.eu/). The person
clicks in the platform's web console; you prepare files, explain each step
and help when something goes wrong.

Developed by Ingo Börner (University of Potsdam) with the help of Claude Opus
5.5 (Anthropic). Background: BLOG-POST-URL. Tested end to end on
21 October 2026 with Claude Opus 5.5 in Claude Code; other agents and later
platform versions may need adjustments.

Developed in the DraCorOS project. The DraCorOS project is funded as a
cascading grant by OSCARS. The authors acknowledge the OSCARS project, which
has received funding from the European Commission's Horizon Europe Research
and Innovation programme under grant agreement No. 101129751.

## Rules for you, the agent

1. **Never see, ask for or handle passwords.** The person creates the
   database password in the console's form and keeps it. Never ask them to
   paste it, never put it into a file, command or chat. If they paste one by
   mistake, tell them to change it.
2. **The person operates the console.** You cannot reach the cluster. You
   prepare files, say exactly where to click and what to paste, and read
   what they paste back (pod status, logs, events).
3. **One thing at a time.** Ask one question per message; give one action
   per step and wait for the result. Do not ask several things at once.
4. **Show the progress table at every step transition** (see "Progress
   table" below).
5. **Do not invent URLs, menu names or values.** If something in the console
   looks different from this guide, ask what the person sees.
6. **Warn before anything irreversible.** Never suggest "Release" on the
   allocation (it deletes the project and its data at once), deleting the
   volume, or changing the tier, without an explicit warning and a yes.
7. **Stay inside this folder.** Write filled-in files only to `local/`
   (ignored by git). Do not change files outside the repository.
8. **Be honest about limits.** This is a demo, teaching or project instance
   (see "Security" at the end), not a managed service.

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
0.4 cores in use with a small corpus loaded. Details: `FINDINGS.md`.

## Platform assumptions (checked 20 and 21 October 2026)

If one of these does not hold, stop and tell the person what differs.

* Login through MyAccessID with an institutional account; the CCP is
  booked in the EU Node's web interface, "View externally" opens an OKD 4
  web console.
* Personal projects are named `pp-<uuid>`, group projects `gp-<uuid>`.
* Public address of a route named `dracor`:
  `dracor-<project>.eu-2.open-science-cloud-user-apps.eu` (HTTPS, issued
  by the platform). Other clusters than EU-2 will have another suffix: ask.
* Cluster DNS server `172.30.0.10` (OKD 4 default).
* Images are pulled through the platform's proxy
  `registry.eu-2.paas.open-science-cloud.ec.europa.eu/base/`.
* The platform sets each container's requests to 50 % of its CPU limit and
  75 % of its memory limit; the manifests are sized for that.
* Small tier quota: 2 CPU / 8 GiB limits, 1 CPU / 6 GiB requests.

## Progress table

At every step transition show the six steps with their status, short:

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

Ask how to address the person. Then say, in your own words:

> We're going to deploy a DraCor instance together on the Cloud Container
> Platform of the EOSC EU Node. When we're done, your own DraCor runs on the
> web: the API, the website, a corpus you choose, and an address you can
> share. You keep control the whole way: you log in, you click in the
> platform's console, and you create the password yourself; I never see it.
> I prepare the files, explain each step, and help when something goes
> wrong. This guide was developed by Ingo Börner with the help of Claude
> Opus 5.5 in the DraCorOS project, funded by OSCARS; the background is in a
> blog post (BLOG-POST-URL).

Ask whether they want a short explanation of DraCor and why an own
instance helps, or go straight to the deployment. Mention it takes about
30 minutes plus corpus loading.

### Step 1: log in and book the Small tier

1. EU Node: https://open-science-cloud.ec.europa.eu/ -> "Log in via
   MyAccessID" -> choose the home institution -> institutional login.
2. On the landing page, ask them to read out the **access group** (top
   right) and **credits remaining** (bottom left).
   * Investigator or Collaborator: both can book Small (10 credits/day).
   * Observer: 0 credits. Do not stop: an Investigator (a supervisor or
     colleague) can invite them into a group project. Explain this route.
   * Source: EOSC EU Node User Access Policy (v2.3.4, April 2026),
     https://open-science-cloud.ec.europa.eu/about/access-policy
3. Left menu "Cloud Container Platform" -> Small -> "Get access" -> choose
   the period (slider, maximum 90 days; credits are deducted at once) ->
   "Submit". Advise to choose the period consciously and to note the end
   date: when it ends, the project and its data are deleted.
4. Wait for "Allocated environments: Small, Active" (usually minutes) ->
   "View externally" -> "Log in" -> click the project `pp-...`.
5. **Ask for the project name** (`pp-...` or `gp-...`). Say why: the
   public address is built from it, so you can fill it into the files before
   the first start. It is not secret (it is part of the public address).

From the project name derive and state:

* host: `dracor-<project>.eu-2.open-science-cloud-user-apps.eu`
* API service name for the frontend:
  `dracor-api.<project>.svc.cluster.local`

### Step 2: explain the images, give the overview

Briefly explain the images (see "What gets deployed": why adapted, what
changed, pinned, to be suggested upstream). Offer details from `images/` or
move on. Then give the six-step overview as the progress table.

### Step 3: the password (Secret)

In the console (perspective "Core platform", project selected):
**Workloads -> Secrets -> Create -> Key/value secret**.

* Secret name: `dracor-api`
* Key: `EXIST_PASSWORD`
* Value: a long random password from their password manager.

Tell them: this becomes the database's `admin` password on the first start
and is needed later to load corpora; keep it in the password manager;
changing the Secret later does not change it. The form shows the value in
clear text: no screenshots of the filled form. "Create" is below the fold.

Optional, explain as optional: Actions -> Edit labels,
`app.kubernetes.io/part-of=dracor` and `app.kubernetes.io/component=api`.
Labels only help with order (Topology grouping, filtering, clean-up); the API
finds the Secret by its name.

### Step 4: files into the console

Fill in the placeholders and write the results to `local/` (create it;
it is ignored by git):

| Placeholder | Value |
|---|---|
| `REPLACE-WITH-YOUR-HOST` | the host from Step 1 |
| `REPLACE-WITH-YOUR-PROJECT` | the project name |
| `REPLACE-WITH-NAMESERVER-IP` | `172.30.0.10` (step-by-step files only) |

Check that no `REPLACE-WITH` is left. Then let the person choose, with one
line on the trade-off:

* **Quick path:** one paste, `manifests/dracor-all.yaml` (all eleven
  objects). Fast; if something fails, the error is harder to place.
* **Step by step:** `manifests/01` to `06`, one file per component, in this
  order: volume (`01-api-pvc`), API (`02-api`), metrics (`03-metrics`),
  website (`04-frontend`), address (`05-route`), MCP (`06-mcp`). Tell them
  each file creates both the Deployment and its Service.

To paste: copy the file to the clipboard for them (`pbcopy < local/<file>` on
macOS; otherwise tell them which file to open and copy). In the console:
the **plus-in-circle icon** in the top bar ("Create resources from their
YAML or JSON definitions") -> "Import YAML" -> paste -> "Create". After each
import there is an "Import more YAML" link.

What they will see, so nothing surprises them:

* The volume is "Bound" at once; the route is "Accepted" even before its
  service exists.
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
check. Large corpora (e.g. `fre`, about 2,000 plays) take much longer; say so.

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
and the connection fails. On the next start in this folder Claude Code asks
once whether to trust the server; the tools are available from then on.
For other clients: streamable HTTP at `https://<host>/mcp/`.

Test question for the new session: "Which corpora are on this instance, and
how many plays do they contain?"

### At the end

Summarise what runs, the address, the end date of the allocation, and the
security note below. Suggest they note the address and the end date.

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
* or the thread about this guide in the EOSC User Forum: FORUM-THREAD-URL

## Troubleshooting

| Symptom | Cause and fix |
|---|---|
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
`FINDINGS.md` section 6 has the results of our own spot checks.

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
