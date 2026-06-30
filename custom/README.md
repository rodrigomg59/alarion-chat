# custom/ — Alarion Chat backend customizations

This directory is Chatwoot's built-in extension point for white-label products
(`ChatwootApp.custom?` / `ChatwootApp.extensions`), wired up the same way the
`enterprise/` directory is. Code placed here is autoloaded and merges into the
app without editing core Chatwoot files, which is what keeps future
`git merge <chatwoot-tag>` upgrades low-conflict.

## Layout

Mirror the path of the file you're extending, under the `Custom` namespace:

```
custom/app/models/custom/account.rb       -> module Custom::Account
custom/app/controllers/custom/...
custom/app/views/...                      -> overrides/adds views, same as enterprise/app/views
custom/lib/...
custom/config/initializers/...
```

## Hooking into a core class

Core Chatwoot classes already call `prepend_mod_with`, `include_mod_with`, or
`extend_mod_with` at the bottom of the file (see `app/models/account.rb` for an
example). Those calls look up `Enterprise::X` *and* `Custom::X` automatically —
nothing else to wire up. If a class you need to extend doesn't have one of
these calls yet, that's the one line you add to core (small, low-conflict diff)
instead of editing the method bodies directly.

## Branding

Don't put branding (name, logo, colors, URLs) here — that's runtime
configuration, not code. See `config/installation_config.yml`
(`INSTALLATION_NAME`, `LOGO`, `LOGO_DARK`, `BRAND_NAME`, `BRAND_URL`, etc.),
settable via environment variables or the Super Admin UI. Zero merge-conflict
risk, since we're not touching upstream files at all.
