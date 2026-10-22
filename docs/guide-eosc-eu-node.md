---
title: "A Guide to Deploying DraCor on the EOSC EU Node"
author: "Ingo Börner"
orcid: "0000-0001-8294-2541"
date: "2026-10-21"
description: "How to run your own DraCor instance (API, website, MCP server, corpora of your choice) on the Cloud Container Platform of the EOSC EU Node: logging in, booking resources, working in the platform's web console, deploying from ready-made configuration files, and loading corpora. Works on your own or together with a coding agent."
keywords: ["DraCor", "EOSC", "EOSC EU Node", "Cloud Container Platform", "OKD", "OpenShift", "Kubernetes", "coding agent", "MCP", "infrastructure"]
license: "CC BY 4.0"
---

# A Guide to Deploying DraCor on the EOSC EU Node

```{admonition} Guide metadata
:class: tip

**Author:** Ingo Börner ([ORCID 0000-0001-8294-2541](https://orcid.org/0000-0001-8294-2541))  
**Version:** 0.1 (2026-10-21), draft; some screenshots still to come  
**Review status:** Not reviewed  
**Tested:** 20 to 22 October 2026, EU-1 and EU-2 clusters, Small tier, personal projects; including a new account on a fresh machine  
**Repository:** [dracor-org/dracor-eosc-eu-node-deploy](https://github.com/dracor-org/dracor-eosc-eu-node-deploy)  
```

```{note}
**How to use this guide:** The guide takes you through the whole deployment, from the first login to a running DraCor with a corpus of your choice. You can follow it **on your own**: the configuration files are ready in the repository, and you only fill in a few values. Or you work **together with a coding agent** (an AI assistant that works in your terminal, reads and writes files and runs commands with your approval; we used Claude Code): the agent fills in the values, explains each step and helps when something goes wrong; boxes marked "With an agent" say where it takes over. The steps on the platform itself (logging in, booking, clicking in the web console, creating the password) are always yours.

**Draft:** a few screenshots are still missing (marked NEW SCREENSHOT) and will follow.
```

## 1. Overview

[DraCor](https://dracor.org) is a platform for the computational analysis of drama: corpora of plays in many languages, encoded in TEI, with an API that serves texts, characters, networks and metrics. The platform runs at dracor.org, but all its components are open source and published as container images, so they can be reused and run anywhere, for example on your own computer. The DraCor Textbook explains how: see the chapter [Infrastructure, "Running DraCor locally"](https://dracor-org.github.io/dracor-textbook/chapters/06-infrastructure.html#practical-examples-running-dracor-locally).

Your own instance is useful for teaching, for a project, for a corpus that is not on dracor.org yet, or for a fixed version of the data that does not change while you write. But an instance on your laptop is not on the web: colleagues, students or reviewers cannot open it.

This is where the [European Open Science Cloud (EOSC)](https://eosc.eu/) comes in. EOSC is a federation of nodes that offer services for researchers; one of them is the [EOSC EU Node](https://open-science-cloud.ec.europa.eu/), run by the European Commission. With an institutional login, it offers computing resources free of charge, among them a [Cloud Container Platform](https://open-science-cloud.ec.europa.eu/services/cloud-container-platform): a managed [Kubernetes](https://kubernetes.io/) environment, that is, a system that runs containers and keeps them running, here in the variant [OKD](https://okd.io/), with a public web address for what runs on it. The smallest tier is enough for a complete DraCor. Why this matters for DraCor and how the guide came about is told in the blog post BLOG-POST-URL.

At the end of this guide you will have:

* your own DraCor on the web, with the API, the website and a public HTTPS address;
* a corpus of your choice, loaded from its GitHub repository;
* a Model Context Protocol (MCP) server that lets AI assistants query your instance.

## 2. Requirements

* An **institutional login** through MyAccessID ([eduGAIN](https://edugain.org/)), with an access group that gets credits on the EU Node (Investigator or Collaborator; see 4.1).
* About **100 credits** for a ten-day test (the Small tier costs 10 credits per day).
* A **text editor** and a **terminal** with `curl` (on macOS and Linux built in; on Windows use [WSL](https://learn.microsoft.com/en-us/windows/wsl/install)). Optionally `git`; the repository can also be downloaded as a ZIP file.
* A **password manager** for the database password.
* Optional: a **coding agent**. We tested with [Claude Code](https://claude.com/claude-code); other agents that read an `AGENTS.md` file should work as well, but were not tested.
* Time: about **30 minutes** if you know the platform's console; for a first time rather **an hour or more**, because the console takes some getting used to. Plus the time to load corpora (minutes for a small corpus, hours for a large one).

No programming is required. You fill in a few values in text files and paste them into a web form.

## 3. Learning Outcomes

After completing this guide, you will be able to:

1. Log in to the EOSC EU Node and book resources on the Cloud Container Platform.
2. Find your way around the platform's web console (OKD): projects, Secrets, Import YAML, pods, logs.
3. Deploy a DraCor instance from configuration files, on your own or together with a coding agent.
4. Load corpora into your own instance and connect an AI assistant to it through MCP.
5. Judge what such an instance is good for, and what it is not.

## 4. Background

### 4.1. The EOSC EU Node and its access groups

The EOSC EU Node is the European Commission's node of the European Open Science Cloud, for researchers like you. You log in with your institution's account; which services you get depends on your **access group**, which the Node derives from what your institution reports about you:

| Access group | Who | Credits |
|---|---|---|
| Investigator | usually faculty | yes; can book all tiers |
| Collaborator | usually staff | yes; can book the Small tier |
| Observer | students, affiliates, alumni | none; can join a group project of an Investigator |

Resources are paid with **credits**. The Small tier of the Cloud Container Platform (2 vCPU, 8 GB memory) costs 10 credits per day; you choose the period when you book, and the credits are deducted at once. When the period ends, the project and everything in it are deleted. The details are in the [User Access Policy](https://open-science-cloud.ec.europa.eu/about/access-policy); it changes from time to time.

### 4.2. What gets deployed

A DraCor instance on the platform consists of a handful of objects, all described in the configuration files of the repository:

| Object | Purpose |
|---|---|
| Secret `dracor-api` | the database password, created by you in a form |
| volume `dracor-api-data` (10 GB) | the database, so that loaded corpora survive restarts |
| `dracor-api` | eXist-db with the DraCor API: stores and serves the corpora |
| `dracor-metrics` | computes the network metrics of each play |
| `dracor-frontend` | the DraCor website; passes `/api/v1` on to the API |
| `dracor-mcp` | the MCP server for AI assistants; reads only |
| routes `dracor` and `dracor-mcp` | the public address: the website at `/`, the MCP server at `/mcp/` |

A few platform terms appear here and later: a **Secret** holds a confidential value such as a password, so that configuration files only refer to it by name; a **route** gives a part of the instance a public web address; each component runs in a **pod**, the platform's unit for a running container. The configuration files are written in **YAML**, a plain text format.

The triple store (Fuseki) of a full DraCor is left out: corpus loading and the website work without it.

```{admonition} Why adapted images?
:class: note
The DraCor components are published as images on [Docker Hub](https://hub.docker.com/u/dracor). The platform, however, runs every container under a random, unprivileged user ID; the official images expect to run as a fixed user and do not start this way. The repository therefore uses copies of the official images with small changes (file permissions, an unprivileged web server on port 8080, no placeholder passwords), also published on [Docker Hub](https://hub.docker.com/u/ingoboerner). The Dockerfiles in the repository show every change; the changes are being suggested upstream, so that the official images can be used directly in the future. The MCP server runs from its official image unchanged.
```

### 4.3. On your own or with an agent

You can do everything in this guide yourself. A coding agent takes over the parts that are tedious or error-prone: filling the values into the files, reading log messages, loading corpora, and explaining what each setting does. The division of roles stays the same in both cases:

* **You** log in, book the resources, click in the console, and create the database password. With an agent, the agent never sees the password.
* **The configuration files** come from the repository. You (or the agent) fill in the values of your project.

The agent cannot reach the platform: it does not have your login, and it should not. Everything that happens on the platform happens through your hands.

## 5. Step by Step

### 5.1. Log in to the EOSC EU Node

Open [open-science-cloud.ec.europa.eu](https://open-science-cloud.ec.europa.eu/) and click the login button at the top right of the page. On the login page, choose **"Log in via MyAccessID"**. Search for your home institution and log in with your usual university account.

If this is your first time on the EU Node, you are asked to register and to accept the EU Node's Terms of Service before you continue. Take a moment to read them; registration is needed only once. After that, the EU Node offers a short guided tour; you can follow it or close it.

NEW SCREENSHOT: the registration page with the Terms of Service (first login).

```{figure} images/eosc-guide/01-login-search.png
---
alt: "Login page of the EOSC EU Node with the search field for the home institution."
width: 100%
---
The login page. Search for your home institution (eduGAIN) and log in with your institutional account.
```

On later visits your institution appears as a button; "Add another institution" lets you choose a different one.

```{figure} images/eosc-guide/02-login-saved.png
---
alt: "Login page with a saved institution as a button."
width: 100%
---
Returning users see their institution as a button.
```

### 5.2. Check your access group and credits

After the login you see the overview of your account. Two things matter here: the **access group** under your name (top right) and the **credits remaining** (bottom left).

```{figure} images/eosc-guide/03-overview.png
---
alt: "Overview page of the EOSC EU Node account with the access group under the user name and the remaining credits in the bottom left corner."
width: 100%
---
The account overview. Top right: your name and access group. Bottom left: remaining credits and the days until they are refreshed.
```

If you are a **Collaborator**, everything in this guide works: 1,000 credits per 90 days are more than enough for a DraCor on the Small tier. Your account may show a **"Request Upgrade"** option to become an Investigator (more credits, all tiers, group projects); you do not need it for this guide.

If you are an Observer, you have no credits of your own; the way forward is a group project of a colleague with Investigator status (see 4.1).

If your access group looks wrong (for example, you are a researcher but shown as Observer), the EU Node's helpdesk cannot change it: the group is derived from what your home institution sends at login. Contact your university's IT helpdesk and ask them to check the attributes they send (the relevant one is called `eduPersonAffiliation`). From experience: university accounts are not always set up the right way for this, and it can take a while to fix, so check early.

### 5.3. Book the Small tier

In the left menu choose **Cloud Container Platform**. The page lists the available environments; **Small** (2 vCPU, 8 GB memory, 10 credits per day) is enough for DraCor. Click **"Get access"**.

```{figure} images/eosc-guide/04-ccp-environments.png
---
alt: "The Cloud Container Platform page with the environments Small and Medium and their Get access buttons."
width: 100%
---
The Cloud Container Platform with its environments. Small is enough for a complete DraCor.
```

A dialog asks for the period (up to 90 days) and shows what it costs. Choose it consciously: the credits are deducted at once, and when the period ends the project and its data are deleted. For a first test, ten days (100 credits) are plenty. Click **"Submit"**. Do not worry about choosing too short a period: you can extend a running allocation later.

```{figure} images/eosc-guide/05-select-period.png
---
alt: "Dialog Select period with a slider set to 10 days and a credit cost of 100."
width: 100%
---
Choosing the period. The credit cost is shown below the slider.
```

The order is processed in the background ("Order submitted"); the button shows "Pending". In our test the resources were allocated within a few minutes.

```{figure} images/eosc-guide/06-order-submitted.png
---
alt: "Confirmation Order submitted, with the Small tier showing Pending in the background."
width: 100%
---
The order is submitted; you will get a notification when the resources are allocated.
```

When it is done, the page shows the environment under **"Allocated environments"**, with its cluster (here EU-2) and the **access expiration date**. Note that date, best right away in your calendar with a reminder a week before: the platform warns only once, five days before the end, and does not renew the allocation by itself. (With an agent: as soon as you have the date, it offers to create a calendar file with the date and reminders.)

```{figure} images/eosc-guide/07-allocated.png
---
alt: "Allocated environments: Small, Active, EU-2, access expiration date, and the button View externally."
width: 100%
---
The allocated environment. "View externally" opens the platform's web console.
```

### 5.4. Open the web console and find your project

Click **"View externally"**. The web console of the platform opens; click **"Log in"** (you are logged in with the same account).

```{figure} images/eosc-guide/08-console-login.png
---
alt: "Login page of the EOSC EU Node Cloud Container Platform with a Log in button."
width: 100%
---
The entrance to the web console.
```

The console shows your **projects**. A personal project is called `pp-` followed by a long identifier, a group project `gp-...`. Click your new project and **note its name**: you need it in the next steps. It is not secret; the public address of your DraCor is built from it.

```{figure} images/eosc-guide/09-console-projects.png
---
alt: "Project list in the web console with a personal project, created just now."
width: 100%
---
The project list. Your new personal project appears with "Created: Just now".
```

#### A short tour of the console

The web console of the platform is powerful but not self-explanatory. Before you start, find the three places you will need; nothing is created yet.

1. **The view.** Directly below the dark top navigation bar, at the top of the left menu, is a wide drop-down box with a gear icon. It should read **"Core platform"** (see the screenshot above, top left). If it shows something else, such as "Developer", click it and choose "Core platform".
2. **Where the password goes.** In the left menu, open **Workloads**, then **Secrets** (see 5.6). There is already a Secret called `registry-credentials`; it belongs to the platform, leave it alone.
3. **Where configuration files are pasted.** In the dark top navigation bar, on the right, are four items: a grid of squares, a **plus in a circle**, a question mark, and your name. The plus opens a menu with **"Import YAML"** (see 5.8).

Later you will check under **Workloads → Pods** whether everything runs. Dates and numbers in the console follow your browser's language settings.


### 5.5. Get the configuration files

Download the repository, either with `git` or as a ZIP file from its GitHub page ("Code", then "Download ZIP"):

```sh
git clone https://github.com/dracor-org/dracor-eosc-eu-node-deploy
cd dracor-eosc-eu-node-deploy
```

The configuration files are in the folder `manifests/`.

```{admonition} With an agent
:class: tip
Start the agent in the repository folder (for Claude Code: `claude`) and say hello. The agent asks which language you want to work in (English by default), introduces itself, asks for your project name and shows an overview of the steps; it keeps showing where you are at every step. It fills in the values for you (5.7) and writes the results into a folder `local/`.

NEW SCREENSHOT: the agent's greeting and the progress table in the terminal.
```

### 5.6. Create the database password

The DraCor API stores its corpora in a database with an administrator password. You create that password in a form of the console, so that it never appears in a configuration file.

In the console go to **Workloads → Secrets**, then **Create → Key/value secret**.

```{figure} images/eosc-guide/10-secrets-create.png
---
alt: "The Create menu on the Secrets page with the entry Key/value secret."
width: 100%
---
Creating a Secret. Choose "Key/value secret". (The `registry-credentials` Secret that is already there belongs to the platform; leave it alone.)
```

Fill in:

* **Secret name:** `dracor-api`
* **Key:** `EXIST_PASSWORD`
* **Value:** a long random password from your password manager

```{figure} images/eosc-guide/11-secret-form.png
---
alt: "The form Create key/value secret with the name dracor-api and the key EXIST_PASSWORD; the value is still empty."
width: 100%
---
The Secret form. Type the password into the large box below "Value"; the "Create" button is further down.
```

```{admonition} Careful
:class: warning
The form shows the password in clear text. Do not take a screenshot of the filled form, and keep the password in your password manager: you need it later to load corpora, and changing the Secret afterwards does not change the database password.
```

```{admonition} With an agent
:class: tip
If you want the agent to load corpora for you later (5.10), you will put the same password into a small file in your home folder, `~/.dracor-eosc-netrc`, which the agent can pass to `curl` but cannot read. You create that file yourself when you get there, in a separate terminal. The settings that block the agent from reading it were tested with Claude Code only; with another agent, load corpora yourself.
```

### 5.7. Fill in your values

The configuration files contain placeholders for the values of your project:

| Placeholder | Value | Example |
|---|---|---|
| `REPLACE-WITH-YOUR-CLUSTER` | the cluster shown on your allocated environment (5.3), in lower case | `eu-1` or `eu-2` |
| `REPLACE-WITH-YOUR-PROJECT` | your project name (5.4) | `pp-0123abcd-...` |
| `REPLACE-WITH-YOUR-HOST` | `dracor-` + project name + `.` + cluster + `.open-science-cloud-user-apps.eu` | `dracor-pp-0123abcd-....eu-1.open-science-cloud-user-apps.eu` |

**Check your cluster.** The allocated environment shows a label such as EU-1 or EU-2 (5.3). It decides where the platform downloads the images from and how your address ends; with the wrong cluster, the images cannot be downloaded. The address pattern is confirmed on EU-1 and EU-2. On another cluster, create the route first (`05-route.yaml`, no placeholders), read its "Location" under **Networking → Routes**, and use that address; then paste the other files one by one.

You can deploy in two ways:

* **Quick path (recommended):** one file, `manifests/dracor-all.yaml`, with everything except the password. Open it in your text editor, replace all placeholders (use "Find and replace"), save. After pasting, open **Networking → Routes → dracor** and check that its "Location" is the address you filled in.
* **Step by step:** one file per component, `manifests/01-api-pvc.yaml` to `manifests/06-mcp.yaml`. Replace the placeholders in each; in `04-frontend.yaml` replace `REPLACE-WITH-NAMESERVER-IP` with `172.30.0.10` (the platform's internal name server). This takes longer, but you see each part start.

Before you paste, search the files for `REPLACE-WITH`: nothing should be left in the values.

Take a moment to read the files before you paste them. It is good practice not to run configuration you have not looked at, and the files are short and commented: you see which images run, which parts are public, how the password is referenced, and how much of your quota each part may use.

```{admonition} With an agent
:class: tip
The agent asks for your cluster, fills in the values, checks that nothing is left, and copies the file you need to your clipboard (on macOS; on Linux you copy it from your editor). It asks which of the two ways you prefer.

It can also explain the files to you, line by line if you like: what each setting does, why it is there, and what would happen without it. Ask it for a security review before you paste: which parts are public, where the password goes, what runs with which rights. This is also a good way to learn the basics of Kubernetes on a real example.
```

### 5.8. Paste the configuration

The place to paste is always the same: in the dark top navigation bar, on the right, there are four items (a grid of squares, a **plus in a circle**, a question mark, and your name). Click the **plus in a circle**, then **"Import YAML"**.

```{figure} images/eosc-guide/12-import-yaml.png
---
alt: "The plus-in-circle icon in the top bar with its menu: Import YAML, Import from Git, Container images."
width: 100%
---
Where configuration files go: the plus-in-circle icon, then "Import YAML".
```

NEW SCREENSHOT: the Import YAML editor with pasted content and the "Create" button.

In the editor that opens:

1. **Click into the editor and select all** (Ctrl+A, on a Mac Cmd+A). The editor may still hold the text of your previous import; if you paste without selecting, the new text is appended and you get errors such as "duplicated mapping key".
2. **Paste** the content of the file.
3. **Check the last line.** It must belong to the file; a stray extra line gives "can not read a block mapping entry".
4. Click **"Create"**. The console lists what it created. "Import more YAML" brings you back to the editor for the next file.

If the editor shows strange text or an old error message that does not go away, reload the page completely (Ctrl+Shift+R, on a Mac Cmd+Shift+R).

If copy and paste gives you trouble (in our test on Ubuntu with Firefox, the pasted text got mixed up), you can try dragging the file from your file manager onto the empty editor; the console accepts dropped YAML files. We have not tested this yet, so check the editor content before you click "Create". Step by step, paste in this order: `01-api-pvc`, `02-api`, `03-metrics`, `04-frontend`, `05-route`, `06-mcp`; each file except the first creates two objects (the program and its internal service).

```{figure} images/eosc-guide/13-import-result.png
---
alt: "Result page Resources successfully created, listing a Deployment and a Service."
width: 100%
---
After each paste the console lists the created objects.
```

Now check the pods: **Workloads → Pods**. The database needs one to two minutes to start; everything is ready when all four pods show **Running** and **1/1**.

NEW SCREENSHOT: the pod list with all four pods (api, metrics, frontend, mcp) at Running 1/1.

If a pod does not start, open it and look at its **Logs** or **Events** tab; section 6 lists the usual causes.

```{figure} images/eosc-guide/14-pod-logs.png
---
alt: "The Logs tab of the API pod, ending with Server has started, listening on."
width: 100%
---
The log of the API pod. "Server has started, listening on" means the database is up. A message about "configure_9_3.dtd" at the start is harmless.
```

**Workloads → Topology** shows all parts of your DraCor at a glance. The small arrow on the website's node opens your instance.

NEW SCREENSHOT: Topology with the four nodes, including the MCP.

```{admonition} With an agent
:class: tip
Paste the pod list, a log or an event message into the conversation: the agent tells you what it means and what to do.
```

### 5.9. Open your DraCor

Your DraCor is now on the web at `https://<your host>/`. A fresh instance is empty: **"No corpora found"** is the expected result.

```{figure} images/eosc-guide/15-dracor-empty.png
---
alt: "The DraCor website of the new instance with the message No corpora found; the footer shows the versions of frontend, API and eXist-db."
width: 100%
---
Your own DraCor, still empty. The footer shows the versions; the suffix "-dirty" of the API version comes from the official release image and is harmless.
```

### 5.10. Load a corpus

Loading corpora works on your EU Node instance exactly as on a local one; the DraCor Textbook explains it in more detail in [Infrastructure, "Loading a Corpus"](https://dracor-org.github.io/dracor-textbook/chapters/06-infrastructure.html#loading-a-corpus). In short:

Which corpora exist is listed in the [DraCor registry](https://raw.githubusercontent.com/dracor-org/dracor-registry/refs/heads/main/corpora.json): each entry has a `name`, a `title` and the GitHub `repository`. Start with a small one. The test corpus (4 plays) takes about a minute, the Bashkir corpus (3 plays) about one and a half. The largest, the French corpus with about 1,940 plays, took about six hours on the Small tier; during that time the API does not answer.

Loading takes two requests with the administrator password: the first registers the corpus, the second starts the load. In a terminal, set your address first (a short line is less likely to break when you paste it), then send the two requests. `-u admin` makes `curl` ask for the password, so it appears neither in the command nor in your shell history:

```sh
H=https://<your host>/api/v1
curl -u admin -X POST -H 'Content-Type: application/json' -d '{"name":"test","title":"Test Drama Corpus","repository":"https://github.com/dracor-org/testdracor"}' $H/corpora
curl -u admin -X POST -H 'Content-Type: application/json' -d '{"load":true}' $H/corpora/test
```

For another corpus, use its `name`, `title` and `repository` from the registry. The second request answers "corpus update scheduled"; the API now downloads the corpus from GitHub. Reload the website after a minute or two.

While a large corpus loads, the API is busy: the website shows errors when it asks for data, and the API pod shows "0/1". That is expected; its log shows the progress, play by play (`LOADING .../tei/<play>.xml`).

NEW SCREENSHOT: the DraCor website with the loaded corpora.

```{admonition} With an agent
:class: tip
The agent reads the registry, offers you a choice and prepares the commands. It can also run them for you without seeing the password: you put the password into a small file in your home folder, in a separate terminal, and the repository's Claude Code settings block the agent from reading that file while it passes it to `curl`. The agent checks that this block holds before you create the file.
```

### 5.11. Connect an AI assistant

The fourth part of your instance is the [DraCor MCP server](https://github.com/dracor-org/dracor-mcp). MCP (Model Context Protocol) lets an AI assistant use your DraCor directly: list corpora, look at plays, characters and networks. The server has no password, so it can only read.

Its address is `https://<your host>/mcp/`. Keep the slash at the end: without it, the server redirects to an insecure address and the connection fails. In Claude Code, for example:

```sh
claude mcp add --transport http dracor-eosc https://<your host>/mcp/
```

Other assistants that support MCP over HTTP take the same address. Then ask a question about your own instance, for example: "Which corpora are on this instance, and how many plays do they contain?"

NEW SCREENSHOT: an assistant answering a question through the MCP of the new instance.

```{admonition} With an agent
:class: tip
The agent registers the server for the repository folder only. Claude Code loads it in a new conversation: in the terminal, quit and start `claude` again in the folder; in VS Code, start a new conversation. It asks once whether you trust the server; `/mcp` shows whether it is connected.
```

### 5.12. Tell us how it went

Did you deploy your own DraCor? Tell us in the thread about this guide in the EOSC User Forum (FORUM-THREAD-URL): that you did it, what worked, where you got stuck, how long it took. The forum is the main place for feedback; platforms change, and every report helps to keep the guide working and shows how the EU Node is used.

If something does not work as described, especially if an agent reports that a step behaves differently than expected, please also open an [issue in the repository](https://github.com/dracor-org/dracor-eosc-eu-node-deploy/issues). An agent offers to draft such a report at the end of the session.

## 6. Troubleshooting

| What you see | Cause and fix |
|---|---|
| API pod shows `CreateContainerConfigError` | The Secret is missing or its name or key is misspelled. Create it as in 5.6; the pod starts by itself afterwards. |
| The website loads, but shows errors (502) when it asks for data, and no corpus is loading | The website cannot find the API. Check that `REPLACE-WITH-YOUR-PROJECT` was replaced correctly in the frontend's configuration. |
| Errors (502) and the API pod at "0/1" while a corpus is loading | Expected: the API is busy. Wait; the log shows the progress. |
| A pod shows `ImagePullBackOff` | The platform cannot download the image. Open the pod's Events tab; the image address may have changed. |
| A pod restarts during a large load (`OOMKilled` in its events) | The memory limit is too low for that corpus. Raise it in the Deployment (YAML tab, `resources`), within the limits of your tier. |
| A new pod stays "Pending" after a change | Your tier's quota is used up. Lower the limits elsewhere or remove what you no longer need. |
| Import YAML: "duplicated mapping key" | The editor still held the previous import and your paste was appended. Click into the editor, select all, paste again. |
| Import YAML: "can not read a block mapping entry" | A stray line, often the last one. Remove it. |
| Import YAML: "already exists" | Some objects were already created. Do not paste the whole file again; fix the broken object in its YAML tab, or delete the objects first ("Removing your DraCor" in section 7). |
| `curl: (2) no URL specified` | A pasted command was split by a line break. Set `H=...` first, as in 5.10. |
| `/api/v1/info` shows `localhost` as its address | The host was not filled in before the first start; the API stores it once. Delete the volume and start again (loaded corpora are lost). |
| The corpus request answers 401 | Wrong password, or the user name is not `admin`. |

## 7. What You Should Know

* **Resources.** With a small corpus the whole instance uses about 650 MB of memory and less than half a processor core; the Small tier has 8 GB and 2 cores. The platform reserves part of each container's limit, so the configuration sets the limits low enough to leave room.
* **Large corpora.** Loading is limited by processor time. With the default limits the French corpus (about 1,940 plays) took about six hours, and during that time the API did not answer. If you need a large corpus quickly, raise the API's CPU limit before you start the load (a change restarts the API and would interrupt a running load). If your instance must stay usable during a long load, a second, separate API with its own volume can load in the background while the first one keeps serving; you then point the website to the new one. This fits into the Small tier with slightly lower limits; it is not part of the ready-made files yet.
* **The end date.** When the booked period ends, the project and its data are deleted. Corpora from GitHub can be loaded again in a new project; keep your own data elsewhere. You can extend the period before it ends. And the address stays the same: the address of your instance is built from the name of your personal project, and when you book the Cloud Container Platform again, your personal project gets the same name. So if you deploy again, people who use the address you shared land at your new instance. (Observed in October 2026; platform behaviour can change.)
* **Security.** Good for a demo, teaching or a project with public data. The administrator password is the only lock on the write functions, so make it long and random. The software is pinned to tested versions and does not update itself. There are no backups and no monitoring.

### Removing your DraCor

To remove your instance, or to start fresh after a failed attempt, delete its objects in the console. Releasing the allocation is **not** a way to clean up: in our test, releasing and booking again right away left all objects in place.

Delete in this order (each list has a ⋮ menu at the end of every row, with "Delete"):

1. **Workloads → Deployments:** `dracor-api`, `dracor-metrics`, `dracor-frontend`, `dracor-mcp`. Their pods disappear with them.
2. **Networking → Services:** the four `dracor-...` services.
3. **Networking → Routes:** `dracor` and `dracor-mcp`.
4. **Storage → PersistentVolumeClaims:** `dracor-api-data`. This deletes the database with all loaded corpora.

Keep the Secret `registry-credentials`: it belongs to the platform. The Secret `dracor-api` can stay if you want to deploy again with the same password.

If you use the OpenShift command line (`oc`), one command does the same, because all objects carry the label `app.kubernetes.io/part-of=dracor`: `oc delete deployment,service,route,pvc -l app.kubernetes.io/part-of=dracor`. You log in with the command from "Copy login command" in the console's user menu; it contains a personal token, so run it in your own terminal and never paste it into a conversation with an agent.

## 8. Further Reading and Resources

- The blog post behind this guide: BLOG-POST-URL
- Repository with the configuration files and the agent's instructions: [dracor-org/dracor-eosc-eu-node-deploy](https://github.com/dracor-org/dracor-eosc-eu-node-deploy)
- DraCor Textbook, chapter "Infrastructure" (running DraCor locally with Docker): [dracor-textbook](https://dracor-org.github.io/dracor-textbook/chapters/06-infrastructure.html)
- Börner, I. and Trilcke, P. (2024). *CLS INFRA D7.3 On Versioning Living and Programmable Corpora*. Zenodo. [https://zenodo.org/records/11081934](https://zenodo.org/records/11081934)
- EOSC EU Node: [open-science-cloud.ec.europa.eu](https://open-science-cloud.ec.europa.eu/), [User Access Policy](https://open-science-cloud.ec.europa.eu/about/access-policy)
- DraCor MCP server: [dracor-org/dracor-mcp](https://github.com/dracor-org/dracor-mcp)
- Trilcke, P., Börner, I., Sluyter-Gäthje, H., Skorinkin, D., Fischer, F. and Milling, C. (2025). *Agentic DraCor and the Art of Docstring Engineering: Evaluating MCP-empowered LLM Usage of the DraCor API*. arXiv:2508.13774. [http://arxiv.org/abs/2508.13774](http://arxiv.org/abs/2508.13774)

## 9. AI Use Declaration

This guide was drafted with the assistance of a large language model (Claude Opus 5.5, Anthropic) in Claude Code. The deployment it describes was carried out by the author in tandem with the same model: the author operated the platform, the model prepared and tested the configuration files and kept the deployment log. The model drafted the guide from that log, the screenshots and the author's instructions. All content was reviewed and edited by the author.

## 10. Acknowledgements

We thank Angelika Hechtl for testing the agent mode.

## 11. Funding

The [DraCorOS project](https://oscars-project.eu/projects/dracoros-fostering-open-science-digital-humanities-connecting-dracor-ecosystem-eosc) is funded as a cascading grant by [OSCARS](https://oscars-project.eu/). The authors acknowledge the OSCARS project, which has received funding from the European Commission's Horizon Europe Research and Innovation programme under grant agreement No. 101129751.
