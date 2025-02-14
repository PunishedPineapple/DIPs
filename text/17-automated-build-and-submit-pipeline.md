- Feature Name: Automated Build and Submit Pipeline
- Start Date: 2022-05-03
- DIP PR: [goatcorp/DIPs#17](https://github.com/goatcorp/DIPs/pull/17)
- Repo-Relevant Milestone: <https://github.com/goatcorp/Dalamud/milestone/1>

# Summary

[summary]: #summary

Introduce a new automated build and submit pipeline to simplify plugin submission, validation and updates.

# Motivation

[motivation]: #motivation

The existing method of submitting and updating a plugin is tedious and labour-intensive. It requires the developer to compile their own code, package it up, and create a matching PR to the DalamudPlugins repo, while ensuring that all package metadata remains consistent.

Once a plugin is submitted for the first time, it must be reviewed by the goatcorp plugin review team. Because we have no guarantee that the code for the plugin matches the binary that was submitted, we must manually decompile and cross-reference the code.

Plugin updates also suffer from this process, but experience less oversight. This means that a developer could potentially deploy malicious code within an update, and it is unlikely to be caught.

To fix this, I would like to propose a design for an automated build and submission pipeline, integrated with GitHub Actions, that can ease this process and help us scale up to accommodate more developers.

Upon completion, this should allow for developers to easily submit new plugins for review, check on the status of their plugin, and release updates, without having to build and package their own plugins. On the goatcorp side, it should ease the review process for initial plugin submission, reduce the overhead of reviewing new code, and remove the need to decompile incoming plugins (as what we receive is what we build is what we publish).

# Guide-level explanation

[guide-level-explanation]: #guide-level-explanation

A new GitHub repository, tentatively `DalamudPluginManifests`, will be created. In this repository, there are several folders, with each folder corresponding to a different "channel" that may be built differently or be located elsewhere. At the time, these are the proposed channels:

- `stable`: The primary repository for Dalamud plugins, and what first-party plugin users will have access to. This is the most stringent to submit to, as it has the widest audience, but we should offer all the tools we can to reduce the barrier to entry.
- `testing/unstable`: A repository for plugins that still require additional testing, or don't meet first-party requirements yet. The requirements to submit to `testing/unstable` are lower than that of `stable`, but plugins still have to follow the general guidelines (e.g. no bots!)
  - Additionally, to assist in the testing process, it is likely that update (not submission) PRs to `testing/unstable` will automatically be approved and merged. This allows developers to rapidly test changes, but ensures that human oversight will still be in place for when they submit to `main`.
- `testing/net6`: Plugins compiled for the .NET 6 version of Dalamud, so that they can be tested for the migration.

Each folder contains files that correspond to a submitted plugin, which is what plugin developers will submit and update. The file will follow this structure (TOML and/or YAML):

```toml
[plugin]
# The open-source Git repository that your code is hosted on.
# Must be visible to the external web, but doesn't have to be GitHub.
repository = "https://github.com/Philpax/plogonscript.git"
# The commit to check out and build from the repository.
commit = "9ef08248497c7f9f1312e14c50650183ee365734882a655345fc06abdac511f1"
# The people authorised to update this manifest (e.g. who can deploy updates).
# Usernames can be specified here, but will be automatically converted to GitHub user IDs.
owners = [
    707827, # Philpax
]
# Optional: where your project/csproj is located within the repository.
# If empty or unspecified, it will be assumed the project is in the root of the repository.
project_path = "plugin"
# The changelog for this version. Will be shown in-game, as well on the Goat Place Discord.
changelog = "added Herobrine"
```

For this to work, the repository must be structured to be amenable to automatic builds and packaging, and it must be accessible to the CI pipeline (which in turn means it must be accessible to the public). Details of this will be covered in [reference-level-explanation].

The `owners` array controls which GitHub accounts are allowed to publish updates for this manifest. You are allowed / recommended to use GitHub usernames for the initial submission:

```toml
owners = [
    "Philpax"
]
```

but they will be automatically converted to GitHub user IDs before being merged in:

```toml
owners = [
    707827, # Philpax
]
```

This allows for developers to easily self-submit new plugins, but hardens the process against the developer's GitHub account name changing. This is especially important as a means to avoid account name hijacking attacks, where a malicious actor steals the username of an existing developer after the developer changes it. This declarative format should also be easy to manage for both developers and administrators.

This DIP also proposes moving all properties associated with `plugin.json` (formerly `$plugin_name.json`) into the project's `csproj` (or equivalent project file, including `fsproj`), and having the `plugin.json` be generated by DalamudPackager using the information within the `csproj`. Developers would still be able to maintain their own `json` if they really want to, but the new process should be easier (only one place to update all project-related metadata).

This change would result in a new section being added to all `csproj`s that choose to participate. An example is provided of what this change would look like for GatherBuddy:

```xml
<ProjectExtensions>
    <DalamudPlugin>
        <ApiLevel>6</ApiLevel>
        <Author>Ottermandias</Author>
        <Name>GatherBuddy</Name>
        <Punchline>Simplify Gathering and Fishing.</Punchline>
        <Description>Adds commands to simplify gathering by finding nodes and fish and their locations via item name and a UI to keep track of special uptime and weather conditions.</Description>
        <IconUrl>https://raw.githubusercontent.com/Ottermandias/GatherBuddy/main/images/icon.png</IconUrl>
        <Tags>
            <Tag>Gathering</Tag>
            <Tag>Fishing</Tag>
            <Tag>Miner</Tag>
            <Tag>Botanist</Tag>
            <Tag>Weather</Tag>
            <Tag>Alarms</Tag>
            <Tag>Timer</Tag>
        </Tags>
        <Hidden>False</Hidden>
    </DalamudPlugin>
</ProjectExtensions>
```

(Note that the `IconUrl` obeys current rules for asset URLs. More work might be done on this in the future.)

When submitting a plugin for the first time, the developer creates the file with the relevant details filled in, and creates a PR. A GitHub Action will retrieve the repository at the specified commit, attempt to build it through the GitHub Actions server pool, package it with DalamudPackager, and produce an artifact that can be downloaded and loaded into Dalamud for testing by a goatcorp plugin reviewer. If accepted, the PR is merged, and the artifact is deployed to DalamudPlugins.

When a new version of a manifest is merged, a GitHub Action will build the repository through the GitHub Actions server pool, as with initial submission, and the resulting artifact will be deployed to DalamudPlugins automatically. The developer does not need to do anything after the manifest update has been merged.

When updating a plugin, the developer merely needs to update the `commit` for the manifest, and make a PR. The PR will be automatically rejected if the submission is by someone who is not on the `owners` list for the manifest on `HEAD` (e.g. a user cannot make themselves the owner of a plugin, they must be authorised by an existing owner or by a repository administrator.) Assuming all is well, the aforementioned steps will take place. Additionally, it is desirable for the buildbot to produce a link to the diff and/or an analysis report in the PR, so that reviewers can scan over the changes.

# Reference-level explanation

[reference-level-explanation]: #reference-level-explanation

With the initial submission, proposed updates or final deployment, the GitHub Action, currently planned to execute on GitHub's CI servers, will pull down the `repository` and build the `csproj` located within `location` (or the root) at the specified `commit`. This Action will only run if it is a new submission, or if an update is being made by someone who is already on the `owners` list of the existing manifest. This code is then automatically packaged with DalamudPackager, with the `plugin.json` (formerly `$plugin_name.json`) being generated using metadata drawn from the `csproj`, and this is used to produce an artifact. This artifact is then made available for download, and upon merging of the PR, the artifact is deployed to the `DalamudPlugins` repository. This build process could be later extended to feature additional verification steps (see [future-possibilities]).

The primary role of the action is to consume a plugin manifest (that is, a `pluginname.toml` in the manifests repo), and produce an artifact that can be loaded by Dalamud, or be included within `DalamudPlugins`. `DalamudPlugins` will only be updated on deployment (e.g. committing of the manifest file to the `master` branch in the repository); it will not be updated otherwise, so there is no risk of the CI accidentally committing an incomplete plugin to the masterlist.

The `csproj` file should describe how to build the plugin. There may be cursory automated checking of the `csproj` to ensure that it is not doing anything malicious, but the primary verification strategy will still be human review. One of the checks will be to ensure that there are no `DalamudPackager` steps that will run on the CI itself, as to prevent double-packaging.

The project will be built with the version of Dalamud appropriate for the channel that is being submitted to. In most cases, this will be the latest publicly released Dalamud, but e.g. the `net6` channel may be built with a Dalamud with .NET6 compatibility, so that testers can easily test plugins for that branch. All other dependencies will be built or sourced from NuGet as appropriate, including recursively pulling down submodules.

Developers will be encouraged to add a conditional step that will not be run on the CI to their `csproj` to run DalamudPackager. In this step, it will be running in a mode where it will output a generated manifest JSON and any files required for the plugin to an output folder, so that developers can easily test their changes and have an experience equivalent to the CI. They're also welcome to continue to build the full ZIP artifacts for second-party and third-party deployments. Note that this is not mandatory, as you can keep locally bundling the DLL and your own JSON if you _really_ want. DalamudPackager will be changed to make this workflow as comfortable as possible, including not building on CI by default.

```xml
<Target Name="PackagePlugin" AfterTargets="Build">
    <DalamudPackager
        ProjectDir="$(ProjectDir)"
        OutputPath="$(OutputPath)"
        AssemblyName="$(AssemblyName)"
        MakeZip="false"
    />
</Target>
```

The CI will not use this build step. Instead, it will build the project as per normal, but ignore the packaging step, and will then run DalamudPackager itself to package up the resulting output. This is done to allow easy migration over to this system (you don't need to include the packaging step if you don't want to), as well as ensuring that the build and packager are equally trusted.

Developers are also encouraged to ensure that the `csproj` is only responsible for building the plugin, and does not build anything unrelated. This may require developers to partition their project, or use a conditional build step similar to the one described above. Additionally, if the plugin has a build-time dependency on something, it is best if that dependency is built ahead of time and the result committed to the repository, as the CI will not be able to build the dependency.

To encourage CI compatibility, as well as making it easier to test, a goatcorp-blessed GitHub Action will be made available that plugin developers can include as part of their repository. This action will build the package, similarly to how the CI would do it, so that developers can test locally with [act](https://github.com/nektos/act) or similar. This action will be made part of the existing plugin samples/templates, so that a user creating a new plugin will be automatically ready to go with CI. This action should be the same as the action run by the CI, or as close as possible.

Finally, build successes and/or failures will be published to the #github Discord channel, or its equivalent, to keep plugin developers in the loop. This is not a strict requirement, and may be adjusted to ensure that the signal-to-noise ratio stays high.

# Drawbacks

[drawbacks]: #drawbacks

This will require all first-party plugin developers to abide by a common project structure that is amenable to being built by CI, and for the code of their plugins to be open-source. The latter is already true, but the former may be a difficult requirement, especially with some of the complex build arrangements that existing plugins use. Upon preliminary exploration, it seems like this should be possible, but we'll need to check in with more developers.

As a whole, it is likely this will require an ecosystem-wide migration, at least for first-party plugins. This will be quite painful and require many people to update their plugins to match a standardised format and build process. This can be mitigated by making it a gradual migration, and supporting the traditional method of submission for some time (next API version or the version after that?). This could potentially be assisted by [DIP#26](https://github.com/goatcorp/DIPs/issues/26).

There are closed-source plugins and plugins with build secrets, like Sonar and PaissaHouse, that are not compatible with this model. For the time being, they will continue to submit to DalamudPlugins. This may be revisited at a later stage.

# Rationale and alternatives

[rationale-and-alternatives]: #rationale-and-alternatives

## Rationale

If we do not implement a solution like this, plugin submission and review will remain tedious. This will cause an increasing amount of frustration for everyone involved, especially as we continue to attract new plugin developers and thus more plugins and updates. Each step in the packaging process is a potential failure point, and supporting a developer through a failure point takes valuable time away. By removing this friction, we can make the experience better for both developers and goatcorp.

This has downstream effects for users, as:

- **the turnaround time for plugin updates will be faster** due to the process being significantly simpler. A developer can make a change, commit it, and then use the GitHub web UI to open a PR to update the manifest; a goatcorp reviewer can then look at the change and approve it, all without having to leave their browser.
- **new plugins can be reviewed faster**, as it will be easier to ensure that the developer is not doing anything malicious, and that what reviewers see through the web frontend is what is being submitted (which is presently not a guarantee).
- **it codifies who is allowed to publish updates for plugins**, so that multi-author plugins don't require goatcorp reviewers to chase up the "primary" author to determine if the update's allowed
- **users can be confident that plugins have been audited to some degree**; while most users do not directly care about this, someone _should_ care, because deploying malware to 100k people would suck
- **happier overall developers** - yes, this is hard to qualify, but the less time your developers spend doing mundane busywork, the happier they'll be, which means they're less likely to burn out, which means more plogons for longer

## This design

This design has a few key benefits over competing designs:

- We don't pay for compute
- It is fully declarative (e.g. what's specified in the metadata repo is what's deployed)
- It does not require us to trust plugin developers' build systems
- It is not a significant change from our existing system

Initial discussion proposed a web portal, likely in addition to existing goatcorp services, where developers could submit their plugins to be reviewed. While this would be nice to have, it's not really immediately necessary.

### Workflow for a new developer

For a new developer, the workflow should look something like this:

#### Development

- A new user clones SamplePlugin or an equivalent that has already been preconfigured to operate within this system, with the inclusion of an appropriately-configured `csproj`
- They write some code, hack away, and get to a state they're happy with
- They update the `csproj` with all of their plugin information
- They build, and assuming they've kept the local `DalamudPackager` step, a folder containing the plugin manifest, as well as the DLL and dependencies is generated
- Dalamud's dev plugin loader can then be pointed directly at that folder, and it loads as if it was in the main repo, including accurate metadata + assembly version et cetera
- They iterate as they please
- They finish up, and push up their changes to a repo

#### Initial submission to `testing/unstable`

- They pop over to `DalamudPluginManifests`, and create a [compliant manifest](#guide-level-explanation) under the `testing/unstable` subfolder, and generate a PR from that
- The CI executes the action, which will:
  - Pull the `repository` at the specified `commit`
  - Build the `csproj` at the `location`, or the root of the project if not available
  - Run a standalone DalamudPackager to package up the build, and include the `changelog` from the manifest
  - Publish the artifact for download on the PR, as well as provide a brief summary of what the CI was able to detect within the project (any risk factors, etc)
- The goatcorp review team can then look over the repository, artifact and summary to make sure that the plugin meets requirements
- If the PR is approved, the `owners` section of the manifest replaces GitHub usernames with GitHub user IDs + comments, the manifest is merged in, and the artifact will be deployed to `DalamudPlugins/testing/unstable/`, making it available for use

#### Submission to `main`

- Once they're sure that the plugin meets all requirements and has been tested sufficiently, they can open a PR to copy `DalamudPluginManifests/testing/unstable/$plugin_name.toml` to `DalamudPluginManifests/main/$plugin_name.toml`
- The goatcorp review team can look over it one last time before deploying to the wider audience
- Merging will deploy the artifact to `DalamudPlugins/main`, making it generally available

#### Updating either `testing/*` or `main`

- The developer has some changes they'd like to submit, so they make some changes and commit it to their repository, making sure to update the `csproj` appropriately
- They open a new PR with `DalamudPluginManifests` to bump the `commit` to the new commit hash
- If the developer is not on the existing authorised `owners`, as checked against the manifest on `HEAD` (not the manifest they're committing!), the PR is automatically rejected
- **If they are updating a testing plugin**, the PR will be automatically approved and merged as long as it builds. This allows for rapid iteration of testing builds.
- Otherwise, a diff report is produced, allowing for the goatcorp review team to read over the changes
- Once the PR is merged, deployment will occur as per normal

#### Removing a plugin

- The developer opens a PR to remove the relevant manifest from the channel
  - We'll want to consider the interaction with [Dalamud#814](https://github.com/goatcorp/Dalamud/issues/814), which provides more information about why a specific plugin is no longer available
- Upon merge, the action will detect this and update DalamudPlugins appropriately

## Alternatives

An alternative design was proposed by [Ay](https://github.com/ayyaruq) where plugin developers, upon main repo approval, would be given keys exclusive to the plugin that could be used to notarise future updates. This would allow new versions of plugins to be deployed without explicit goatcorp approval, and to establish a cryptographic audit trail. However, this solution was rejected due to its relative complexity compared to the proposed centralised solution. It's cool, but overkill, at least for now.

# Prior art

[prior-art]: #prior-art

RFC: Got some cool prior art? Ay mentioned working on something similar in the past.

# Unresolved questions

[unresolved-questions]: #unresolved-questions

Several points within this DIP were discussed during Discord conversations. Here are links, as well as a brief summary:

- <https://discord.com/channels/581875019861328007/860813266468732938/972255490128113684>
  - Concerns raised during this discussion:
    - **Not every dev wants to use DalamudPackager**: they won't need to, but using it will make their life easier, as they will no longer need to maintain the JSON themselves, and what they build is the same thing that Dalamud will consume, and will match what the buildbot will build.
    - **How will the Dalamud community benefit from these changes?** See [rationale-and-alternatives].
    - **Will this lock out projects with unconventional build processes?** Hopefully not; as long as the buildbot can build _a_ `csproj` and produce a DalamudPackager artifact at the end, it'll be happy. Other projects or steps are fine, just make sure they're not in the build path.
    - **What about build-time project dependencies that I rely on (e.g. external assets), but don't want goatcorp to build?** We have no immediate plans to support this. Unfortunately, this means our current answer is "build the dependency separately, and commit it into the repository to be picked up by the buildbot."

# Future possibilities

[future-possibilities]: #future-possibilities

This would form the backbone of any future CI we do around plugin submission. Ideas for improvement include:

- Implementing a web frontend for the plugin submission/review/update process. Instead of manually PRing up changes to the metadata repo, a plugin developer could pop over to the frontend and specify which commit they'd like deployed, then hit the button.
- Use bors-ng, or a relative, to handle approval of reviews to make it easier for people to review and merge plugin submissions/updates asynchronously
- Run our own server to execute the CI actions on, so that we're not beholden to the free GHA organisation minute limit
- As we're building the code, we have an opportunity to analyse it and ensure that it isn't doing anything outright naughty. We would likely use a solution similar to [S&box](https://wiki.facepunch.com/sbox/AccessList)'s. This could be combined with other steps, like asking the plugin to declare the namespaces it will use.
  - This is pretty hard to verify when unsafe code is enabled, as all bets are off. This would be a relatively coarse check to make sure that there isn't anything _immediately_ suspect.
  - This is not an immediate problem as the plugin security model is currently too open to allow for it, anyway. It's something we can revisit in a future DIP.
- This RFC does not currently describe a way to disable a plugin. The metadata structure could be extended with an `enabled: bool` that will govern whether or not the plugin is blacklisted, thus allowing both goatcorp and plugin developers to blacklist plugins with ease.
- This RFC does not address automated building and deployment for plugins with hidden source or secrets, which will continue to submit to DalamudPlugins. This should be addressed at some point to allow plugins under these categories to submit. The current solution in mind for secrets is for the secret to be provided to goatcorp, who will then supply it to the GitHub runner, but this requires more thought. For hidden source plugins, we may ask developers to provide access to their repository for the buildbot, but we will need to ensure that this is done securely.
