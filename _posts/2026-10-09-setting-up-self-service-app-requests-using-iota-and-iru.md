---
layout: post
title: "How I facilitated Self-Service App Requests Using IOTA and Iru"
date: 2026-10-09
categories: mdm automation
description: "Automating one of the boring bits - Users requesting app packages"
excerpt: "A walkthrough on giving end users the power to request the apps they need — without flooding IT with tickets."
---

## The Problem

Let's face it - app requests aren't great.

No, it's okay to admit they do. Iru has taken a lot of the brunt of this by having Auto Apps, but what if the app you need isn't an Auto App? We can go out to the internet, get the package, find the settings for managing it for things like updates, making sure users can't sign in with personal accounts that sync to who-knows-where. Yawn.

With the advent of managing our MDMs with code, why not manage app requests by code too? Request comes in, automation checks if the app package lives somewhere, pulls it down, packages it, merges into MDM, assign it to the required users. Utopia. Bliss. A warm snuggly feeling.

Because I'm a bit of an unhinged tinkerer, I thought I'd give it a go. And it worked. With IruCtl and IOTA working hand-in-hand, the app packages are built, provisioned, and pushed into Iru ready for users to get them installed and get working.

## What Are IOTA and IruCtl?

Now, if you're newer to Iru, I'll break down what each tool does, to add more context to how the entire thing fits together.

### IOTA

IOTA stands for Iru Orchestration Tool for AutoPkg. If you weren't aware, AutoPkg is part of the pipeline that facilitates Auto Apps at Iru. IOTA gives you that experience at home/in the office, giving you a programmatic way to manage your packages. THe magical thing is that this can run completely in your CI - No need for any hosted Mac Minis, especially now with the price spikes both physically, and hosted!

A link to IOTA's GitHub can be found here: [[IOTA GitHub](https://github.com/kandji-inc/IOTA)]

### IruCtl

Iru Control, or IruCtl for short, is a utility for managing resources via the Iru API. By leveraging IruCtl, you can both import existing, and create new, custom profiles, scripts, and app pakcages without having to move them around to diffent workstations or network locations.

Easy to install and get going, it's an invaluable tool for managing Iru. I personally have a workflow that runs in GitHub for drift detection, where it runs daily, identifies any changes that have been made in the UI, and prompts if I want to pull them into the repo and have them become managed, or discard them and have IruCtl restore the desired state.

That repo-to-tenant loop is the foundation this request pipeline builds on: app requests add a reviewed recipe to the repo, then IOTA handles the installer upload while IruCtl continues to manage the rest of the tenant configuration.


## The workflow

![Workflow diagram showing request intake, recipe review, packaging, draft pull request, and IOTA deployment](/assets/images/Mac%20App%20Self-Service%20with%20IOTA%20and%20iructl%20-%201.%20Request%20pipeline.png)

The beauty of this setup is that the request channel is separate from the packaging pipeline. A user could raise a request in an ITSM system, a form, or GitHub; each front door only needs to pass the same required information to the workflow. I started with a GitHub issue form, but the reusable workflow accepts a normalised request, so another source can be added as a small adapter.

The requester names the app, explains who needs it and why, and provides the vendor and deployment details. Intake validates those fields, checks whether the app is already managed or requested, and searches AutoPkg's public recipe index for `.pkg` recipes. It ranks the candidates, favouring an exact recipe-name match and trusted repositories. If the app is already present, the request is flagged as a duplicate. If no `.pkg` recipe is available, the team is pointed toward manual packaging with IruCtl.

Here is the Firefox request as it appears in the issue form, and the three test requests with their resulting status labels.

![Firefox app request form with vendor URL, justification, audience, install preference, and licensing fields](/assets/images/iru-app-request/firefox-request.png)

![GitHub issue list showing Firefox marked recipe-ready, CrowdStrike Falcon marked duplicate, and Zyxqor Notes marked manual-packaging](/assets/images/iru-app-request/request-statuses.png)

The intake comment includes the matching recipes and their parent chains, so the reviewer can see what the automation intends to try before approving it.

![Intake comment listing the Firefox AutoPkg recipe candidates and their parent chains](/assets/images/iru-app-request/recipe-search-results.png)

No installer is downloaded at this stage. The IT team is notified with the request details and any matching recipes. A maintainer makes the first decision: close the issue to decline it, or add the `approved` label to let packaging proceed.

After approval, a macOS runner tries up to the three best recipe candidates. For each one, it brings the recipe's full parent chain and any required non-core processors into the repository, then runs AutoPkg against that vendored copy alone. This checks that the recipe can build the package in the same self-contained form IOTA will use later. The job has no Iru credentials or tenant access, and the downloaded package and recipe code do not go straight to production.

When a recipe passes, a separate publish job checks the output, then opens a **draft pull request** containing the vendored recipe files plus updates to `recipe_list.txt` and `recipe_map.json`. The team reviews the code that will run in CI, confirms the Custom App name and settings, and agrees how the app should be scoped. This is the second human checkpoint. Only after review and merge does the existing IOTA workflow run: it builds the package, uploads it to Iru, and creates or updates the Custom App. IOTA does not assign the app to devices, so an admin completes that step in the Iru console. The resulting console metadata is then brought back into the repository through the usual `iru-console-sync` pull request.

In the first end-to-end intake tests, Firefox found matching recipes and produced a draft PR after the approved dry run. 

A request for an app already in the repo was caught as a duplicate, as shown here.

![CrowdStrike Falcon duplicate request form](/assets/images/iru-app-request/duplicate-request.png)

![Intake comment identifying the existing CrowdStrike Falcon Custom App and asking the team to confirm the duplicate](/assets/images/iru-app-request/duplicate-detected.png)

A made-up app with no recipe was sent down the manual-packaging path. The team can still fulfil it, but it takes the manual IruCtl route rather than the AutoPkg/IOTA path.

![Zyxqor Notes request form used to exercise the no-recipe path](/assets/images/iru-app-request/no-recipe-request.png)

![Intake comment reporting no AutoPkg recipe and recommending manual packaging with iructl app new](/assets/images/iru-app-request/no-recipe-result.png)

![Issue labelled manual-packaging after no AutoPkg recipe was found](/assets/images/iru-app-request/manual-packaging-label.png)

For Firefox, the first packaging run exposed a useful edge case: AutoPkg left a `__pycache__` directory beside the vendored processor, so the publish job rejected the output and committed nothing. After fixing the handoff to include only the vendored files, the rerun passed and opened the draft PR. This is the kind of check that keeps unexpected files from crossing from the recipe-running job into a change that could later run with Iru credentials.

![GitHub issue timeline showing the first Firefox output rejected because of __pycache__, followed by a successful dry run and draft pull request](/assets/images/iru-app-request/packaging-and-draft-pr.png)

The draft PR makes the recipe chain, built version, vendored files, and review checklist visible before merge.

![Firefox draft pull request details with recipe chain, vendored files, CodeSignatureVerifier result, review checklist, and post-merge steps](/assets/images/iru-app-request/draft-pr-review.png)

The Firefox test PR was closed, so it demonstrated the request-to-review path; it was not a live deployment to the tenant.

## Where IruCtl steps in

IruCtl is still the source of truth for the tenant configuration around the installer. Profiles and scripts live in `profiles/` and `scripts/`; each Custom App's metadata lives in `apps/`. The installer itself is not stored in Git, which is why IOTA handles vendor packages separately.

The app-request draft PR changes the AutoPkg side of the repo: it adds the reviewed recipe files and registers the recipe in `recipe_list.txt` and `recipe_map.json`. The normal pull request validation still runs, including IruCtl dry runs that show the tenant changes for profiles, scripts and app metadata. Once the recipe change is merged, IOTA takes over the installer work. Its Monday 07:00 UTC run (or a manual run) builds the package and creates or updates the Custom App in Iru.

IOTA does not assign the app to a Blueprint or Assignment Map. An admin handles that in Iru, then **Pull from Iru** brings console changes and the new app's metadata back into the repo as an `iru-console-sync` pull request. It runs daily, on every pull request, and on demand. Reviewing and merging that PR records the app under `apps/`.

There is one important detail: the regular **Deploy to Iru** workflow pushes profiles and scripts, but it does not deploy Custom App changes. The pull request dry run previews app metadata, and **Pull from Iru** records console changes, but merging a PR alone does not apply an app metadata change to the tenant. For the apps in `recipe_map.json`, IOTA owns the installer; don't use the regular manual `iructl app push` path, which could upload a stale local package over IOTA's version.

For apps without an AutoPkg recipe, manual packaging and `iructl app push` remain the fallback. Keeping that route separate from IOTA ensures each installer has one clear owner.

This is how the app request fits into the wider Iru configuration-as-code loop. For profiles and scripts, an admin makes a change on a branch, IruCtl's pull request dry run previews its tenant impact, and merging deploys the repo state. Changes made directly in the console travel back through **Pull from Iru** as a reviewable `iru-console-sync` pull request.

![IruCtl CI diagram showing branch changes validated and deployed to Iru, with console changes returned to the repo through a sync pull request](/assets/images/Mac%20App%20Self-Service%20with%20IOTA%20and%20iructl%20-%202.%20iructl%20in%20CI.png)

The app request adds an installer-specific path alongside that loop: IOTA builds and uploads the vendor package, while IruCtl keeps the surrounding configuration and metadata reviewable in the repo. The two paths meet when the app's metadata is pulled back from Iru and reviewed in the sync PR.

## Wrapping up

Now you've seen the **how**, I'll be sharing week-by-week on how I built this out, deep-dive into issues I ran in to, and answer any questions that I may get in.

But for now, enjoy!