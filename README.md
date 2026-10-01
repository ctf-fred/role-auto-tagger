# Role auto tagger

A Contentful App Framework app that tags new entries automatically, based on the roles of the user
who created them.

An admin maps each role in a space to a set of tags. When an entry is created, a Contentful
Automation calls this app's **Auto-tag by Role** App Action. The action reads the entry's creator,
looks up their roles, and adds the tags mapped to those roles to the entry.

There is no editor-facing UI. The app's only screen is its configuration screen.

---

## Requirements

- **Node 20 or newer** (developed on Node 26)
- A **Contentful organization** where you can manage apps
- A **CMA personal access token** with app-management rights in that organization, for the setup
  scripts
- A space to install into, where:
  - you have the right to install apps
  - **Contentful Automations** are available. This app is triggered by an Automation, so check that
    your space's plan includes them before installing.
- Tags named in the form `Group: value`, for example `Region: EMEA` or `Team: Marketing`. Only
  grouped tags can be mapped, and an admin chooses which groups the app may apply.

---

## Install from scratch

Each step is idempotent, so re-running one is safe.

### 1. Install dependencies

```bash
cd role-auto-tagger
npm install
```

### 2. Configure credentials

```bash
cp .env.example .env
```

Fill in `CONTENTFUL_ORG_ID` and `CONTENTFUL_ACCESS_TOKEN`, and leave `CONTENTFUL_APP_DEF_ID` empty
for now. `.env.example` says where to find each value. `.env` is gitignored.

### 3. Create the app definition

```bash
npm run create-app-definition
```

This prints `CONTENTFUL_APP_DEF_ID=…`. Paste that line into `.env`.

### 4. Declare the installation parameters

```bash
npm run sync-parameters
```

**This must run before the config screen is saved for the first time.** Contentful validates an
installation's parameters strictly against the app definition, so saving an undeclared parameter
fails with `422 The property X is not expected`.

### 5. Build, upload, activate

```bash
npm run build:all
npm run upload-ci
npm run activate
npm run sync-action
```

- `build:all` builds the config screen and the App Function.
- `upload-ci` uploads the bundle without activating it.
- `activate` attaches the bundle and sets the app's locations in a single request.
- `sync-action` creates or updates the **Auto-tag by Role** App Action from
  `contentful-app-manifest.json`, including its parameters.

They are separate because a brand-new definition cannot be activated by `contentful-app-scripts`
itself (see [Deployment gotchas](#deployment-gotchas)).

The upload registers the function. The action is registered by `sync-action`, which matches the
existing action on the function it invokes rather than on its ID. `contentful-app-scripts
upsert-actions` matches on the ID, and an action's ID can be generated, so on an existing
definition it can create a duplicate action instead of updating the one your Automations call.

`npm run show-definition` confirms the result. It prints the locations, the declared parameters and
the registered action, and flags anything that differs from what this repo expects.

---

## Set it up in a space

### 1. Install the app and configure it

Install the app in the space and environment where you want tagging. App installations are per
space **and** environment, so repeat this in every environment that needs it.

On the config screen:

| Setting | What it does |
|---|---|
| **CMA token** | A personal access token used to read the space's members and roles. See [Why a token is needed](#why-a-token-is-needed). |
| **Tag groups** | Which `Group: value` groups the app may apply. Tags outside an enabled group are never applied, even if they are mapped. |
| **Role auto-tagging** | For each role in the space, the tags its members' entries receive. Searchable, 20 tags per page. |

Save.

### 2. Create the Automation

**Saving the config screen does not start tagging.** Create a Contentful Automation in the same
space and environment:

- **Trigger:** an entry is created
- **Action:** call an App Action. Choose this app's **Auto-tag by Role**.
- **Parameter:** set **Entry ID** to the ID of the entry that triggered the Automation. Leave
  **Dry run** and **Simulate role** unset. Those are for the Troubleshooting tab.

The app does not create the Automation itself, for two reasons:
- **Permissions:** the App SDK does not let an app create Automations.
- **Control:** a step the customer can see, scope and switch off is better than one an app sets up
  silently.

### 3. Check it

Open the config screen's **Troubleshooting** tab, choose a role and click **Run test**. It shows the
tags an entry created by someone with that role would receive. See [Troubleshooting](#troubleshooting).

Then create an entry as a user whose role has mapped tags. The tags appear on the entry shortly
after it is created.

---

## Troubleshooting

The function's own logs belong to the organization that owns the app definition. On an app
installed from the Marketplace, that is not you. So the config screen has a **Troubleshooting** tab
that shows you what the logs would say.

**How it works:** it calls the **same App Action your Automation calls**, as a *dry run*, with the
**saved** configuration:
- It works out the result and writes nothing.
- It uses the role you choose in place of the entry creator's roles.
- If you pick an entry, it also shows which tags the entry already has.
- If the screen has unsaved changes, the tab says so, because the run uses what is saved.

**What it shows:**
- the tags that would be added
- mapped tags that would not be, and why
- the function's log lines, word for word

When something fails, the message is the same one the function logs.

| The tab or the log says | What it means | What to do |
|---|---|---|
| `no cmaToken in installation parameters` | No token is saved | Add one on the Configuration tab and save |
| `401: The access token you sent could not be found or is invalid.` | The token is invalid, expired or revoked | Create a new token and save it |
| `nothing configured — roleTagMapping or enabledTagGroups is empty` | No group is enabled, or no role has tags | Enable a group and map at least one role |
| `no tags mapped for any of the user's roles` | The role has no tags mapped | Map tags to that role |
| `mapped but not applied: … its group is not enabled` | The tag's group is switched off | Enable the group, or unmap the tag |
| `mapped but not applied: … no longer exists` | The tag was deleted | Unmap it on the Configuration tab |
| `mapped tags are not in an enabled tag group` | None of the role's tags can be applied | As above |
| `all mapped tags already present` | Nothing to add | Nothing to fix |
| `user … is a space admin; admins hold no roles` | Space admins are never tagged | Expected. Test with a non-admin, or simulate a role here |
| `user … is not a member of space …` | The creator is not in the space | Check the creator's access, and that the token's user can see all members |
| `entry was created by a AppDefinition, not a user` | Another app created the entry | Expected. Only entries created by people are tagged |
| `role … does not exist in space …` | The role was deleted after it was mapped | Pick another role, and remove the old mapping |

**If the tab passes but entries are not tagged,** check that the Automation:
- exists in this space **and** environment
- runs on entry creation
- passes the new entry's ID as **Entry ID**

The Automation's own run history shows whether it ran and whether the call failed.

---

## How it behaves

- **The creator is read from the entry.** `sys.createdBy` decides whose roles count, not who or what
  called the action. An entry created by an app has an app, not a user, as its creator, so it is left alone. An entry created through the API with a personal access token counts as created by that token's user.
- **Direct and team members are treated the same.** Roles come from the space's effective
  membership, which combines roles granted directly and through Teams.
- **Space admins receive no tags.** In Contentful, admin is a flag rather than a role, so no mapping
  can match an admin.
- **Tags are only added, never removed.** Tags already on the entry are kept, including ones an
  editor added. If every mapped tag is already present, nothing is written.
- **Concurrent edits are retried.** The Automation fires while the editor may still be typing. On a
  version conflict, the action re-reads the entry and retries up to three times, keeping any tags
  added in the meantime.
- **Writes use the app's own identity.** The tag write goes through the App Function's app identity,
  so the entry's `sys.updatedBy` names the app.

---

## Why a token is needed

An app identity cannot read who is a member of a space or what roles exist in it. Both listings
return `401 AccessTokenInvalid` to an app token. So the only way to resolve the creator's roles is a
user's personal access token.

The token is used for **those two reads only**: `space_members` and `roles`. Reading the entry,
reading tags and writing tags all use the app identity.

To keep the token's reach small:
- **Create it for a dedicated user,** not a person's own account.
- **Make that user a member of only the spaces the app is installed in.** Reading members and roles
  needs no write access to content.
- **Set an expiry and rotate it.** A personal access token can do anything its user can.

---

## Scripts

### Setup and deployment

| Command | What it does |
|---|---|
| `npm run create-app-definition` | Creates the app definition and prints its ID |
| `npm run sync-parameters` | Declares the installation parameters. `-- --prune` also undeclares any this repo no longer uses |
| `npm run build:all` | Builds the config screen and the App Function (`npm run build` and `npm run build:functions` do one each) |
| `npm run upload-ci` | Uploads the bundle, without activating it |
| `npm run activate` | Attaches the newest bundle (or `-- <bundleId>`) and sets the locations |
| `npm run sync-action` | Creates or updates the App Action and its parameters from the manifest |

### Checks

| Command | What it does |
|---|---|
| `npm test` | Runs the Vitest suite against an in-memory fake CMA. No credentials, no network |
| `npm run test:coverage` | The same, with a coverage report in `coverage/` |
| `npm run typecheck` | `tsc --noEmit` over `src/` and `tools/` |
| `npm run lint` | ESLint, including the React hook-order rule |
| `npm run test:watch` / `npm run lint:fix` | Watch mode, and lint with autofix |

### Inspection

| Command | What it does |
|---|---|
| `npm run show-definition` | Prints the app definition and flags anything unexpected |
| `npm run check-members -- <spaceId> …` | Read-only. Checks that a space's effective membership agrees with its direct and team memberships, user by user |

### Local development

```bash
npm start        # or: npm run dev
```

This serves the config screen on `http://localhost:3002`. To use it, set the app definition's
**Frontend** to that URL in the web app. `npm run activate` switches the definition back to the
uploaded bundle.

---

## Deployment gotchas

- **Declare parameters before the first config save.** Run `npm run sync-parameters` first, or the
  first save fails with a 422.
- **`contentful-app-scripts activate` cannot activate a new definition.** It sends only a `dialog`
  location, which Contentful rejects. Upload with `--skip-activation` (which `upload-ci` does), then
  run `npm run activate`.
- **A definition has either a frontend URL or a bundle, never both.** `npm run activate` removes
  the URL.
- **axios is pinned to 1.19.0** (`overrides` in `package.json`). From 1.20, axios's fetch adapter
  sets a `cache` option on every request. The App Functions runtime rejects that option, so every
  CMA call fails with `The 'cache' field on 'RequestInitializerDict' is not implemented.` It passes
  locally, because Node accepts the option. So `npm audit` reports the axios advisory: lift the pin
  only once a release no longer sets `cache`, and test the App Action in a space before deploying.
- **A thrown error reaches the caller without its message.** When an App Function throws, the caller
  sees only `Invoking function … failed (code-error)`. So a **dry run** returns its failure as
  `{ failed: true, error, log }`, which is what lets the Troubleshooting tab show the real message.
  A real run still throws, so the Automation records it as failed.
- **Asset paths must be relative.** Contentful serves bundles from signed URLs, so
  `vite.config.mts` sets `base: './'`. Absolute paths 403 and every location renders blank.

---

## Known limitations

- **The role mapping is per space.** It is keyed by role ID, and role IDs differ between spaces even
  for roles with the same name. Configure each space in its own config screen.
- **Admins are never tagged,** because they hold no roles.
- **Only entry creation is handled,** and only for entries a user created. Changing a user's roles
  later does not retag their existing entries.
- **Only grouped tags can be mapped,** and only from enabled groups.
- **The trigger is manual.** Tagging does nothing until someone creates the Automation in each space
  and environment.

---

## Security

- **The token is a `Secret` installation parameter.** Contentful encrypts it and returns it only as
  a mask, and the config screen never overwrites the stored value with the mask.
- **The action's input is validated.** `entryId` and `simulateRoleId` must look like Contentful IDs
  before they are used in a request.
- **A simulated role cannot write.** `simulateRoleId` is refused unless `dryRun` is also set. This is
  checked when the input is read, and again in the engine, so the Troubleshooting tab cannot apply
  tags based on a role the creator does not hold.
- **Secrets stay out of the repo.** `.env` is gitignored, and so is `.claude/settings.local.json`,
  which can contain tokens from approved shell commands.

---

## Layout

```
src/
  lib/autoTag.ts              the engine: takes its clients, so it is testable
  lib/errors.ts               one readable line from a CMA failure
  lib/__tests__/              Vitest cases against a fake CMA (testkit.ts)
  functions/autoTagByRole.ts  the App Action: validates input, builds clients, calls the engine,
                              returns the log lines with the result
  locations/ConfigScreen.tsx  Configuration and Troubleshooting tabs
  components/Troubleshooting.tsx  dry run of the real action for a chosen role
  App.tsx, index.tsx          location router and entry point
tools/
  create-app-definition.ts  sync-parameters.ts  activate-bundle.ts  sync-action.ts  show-definition.ts
  parameters.ts             the installation parameters, in one place
  check-members.mts         read-only membership check
contentful-app-manifest.json  the function and the App Action
```

The engine in `src/lib/autoTag.ts` takes its CMA clients as arguments rather than creating them.
Keep it that way. That design is what lets the tests run against a fake, and it keeps the app
identity and the token visibly separate.
