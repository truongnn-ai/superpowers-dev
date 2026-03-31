# Installing Superpowers (Dev)

This guide covers installing the marketplace and plugin from this local development folder.

## 1. Register the marketplace

From inside the repo directory:

```bash
/plugin marketplace add ./.claude-plugin/marketplace.json
```

Or using an absolute path from anywhere:

```bash
/plugin marketplace add ~/Documents/ws/superpowers-dev/.claude-plugin/marketplace.json
```

> The marketplace name (`superpowers-marketplace-dev`) is read from `marketplace.json` — do not use the `name@path` syntax for local paths, as Claude Code will treat `name` as a GitHub identifier.

## 2. Verify the plugin is available

Open the plugin browser and go to the **Discover** tab to see plugins from all configured marketplaces (Optional):

```bash
/plugin
```

Then press Tab to navigate to **Discover** and search for `superpowers`.

To confirm the marketplace was registered:

```bash
/plugin marketplace list
```

You should see `superpowers-marketplace-dev` in the output.

## 3. (Optional) Remove any existing superpowers installation

If you already have superpowers installed from another source, remove it first:

```bash
/plugin uninstall superpowers@<marketplace-name>
```

## 4. Install the plugin

```bash
/plugin install superpowers@superpowers-marketplace-dev
```

## 5. Verify

Start a new session and ask Claude to help plan a feature or debug an issue. The relevant skill should trigger automatically.

---

## Updating

After pulling new changes, update the plugin:

```bash
/plugin update superpowers@superpowers-marketplace-dev
```

## Uninstalling

```bash
/plugin uninstall superpowers@superpowers-marketplace-dev
/plugin marketplace remove superpowers-marketplace-dev
```
