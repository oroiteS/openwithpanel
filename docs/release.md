# Releasing

Pushing a version tag runs the Release workflow in GitHub Actions. It builds OpenWithPanel, packages it as a DMG, and creates a draft release with the DMG and a Homebrew cask file whose version and checksum are already filled in. Review the draft on GitHub, then publish it.

The workflow is defined in `.github/workflows/release.yml` and packages with `scripts/package.sh`, the same script used for local builds.

Whether a release is signed depends on the signing secrets configured in the repository:

- **None configured:** the app is signed ad hoc and isn't notarized by Apple. macOS blocks it the first time a user opens it, and they have to click **Open Anyway** in **System Settings › Privacy & Security**. The DMG includes `First Launch.txt` with those steps, and the draft release notes include them too. The **First launch** section of the README (and its translations) describes this case.
- **All 6 configured:** the app is signed with a Developer ID and notarized, so users can open it directly. Once releases are signed, remove the **First launch** section from the README and every translation.

If only some of the secrets are configured, the workflow stops before building, so a release meant to be signed is never published unsigned.

## Setting up signing and notarization (optional)

1. Join the Apple Developer Program.
2. Create a **Developer ID Application** certificate on the developer website, install it in your keychain, then export it together with its private key as a `.p12` file from Keychain Access, setting a password.
3. In App Store Connect, under **Users and Access › Integrations › App Store Connect API**, create an API key with the **Developer** role. Download the `.p8` file (it can only be downloaded once) and note the key ID and issuer ID.
4. In the GitHub repository, open **Settings › Environments** and create an environment named `release`:
   - under **Deployment branches and tags**, choose **Selected branches and tags** and add only the tag rule `v*`;
   - turn on **Required reviewers** and add yourself, so every release waits for your approval before the secrets are used.
5. Add these 6 secrets under **Environment secrets** of the `release` environment. Don't add them as repository secrets, which other workflows and branches could read.

| Name | Value |
|---|---|
| `DEVELOPER_ID_P12_BASE64` | The `.p12` file in base64: `base64 -i certificate.p12 \| pbcopy` |
| `DEVELOPER_ID_P12_PASSWORD` | The password set when exporting the `.p12` |
| `DEVELOPER_ID` | The certificate name, such as `Developer ID Application: Name (TEAMID)`; find it with `security find-identity -v -p codesigning` |
| `NOTARY_KEY_BASE64` | The `.p8` file in base64: `base64 -i AuthKey_XXXX.p8 \| pbcopy` |
| `NOTARY_KEY_ID` | The API key ID |
| `NOTARY_ISSUER_ID` | The issuer ID |

## Publishing a release

1. Create and push a tag, for example `git tag v0.1.1 && git push origin v0.1.1`. Any new `v*` tag can be published: the workflow passes the tag without its `v` to `scripts/package.sh` as `VERSION`, which overrides `MARKETING_VERSION` for that build, so the tag, the DMG name and the version inside the app always agree. You do not need to bump `MARKETING_VERSION` in the project first. You can also run the workflow manually from **Actions › Release › Run workflow** and enter the tag name.
2. When the workflow finishes, find the draft on the **Releases** page, check the notes and assets, and click **Publish release**.

## Homebrew

This fork publishes from `oroiteS/openwithpanel`, and the cask lives in `oroiteS/homebrew-tap`:

```sh
brew install --cask oroiteS/tap/openwithpanel
```

`script/sync.sh` in that tap reads the latest release every 30 minutes and bumps `version` and `sha256` on its own, so a new release needs no manual step there. It expects the asset name `OpenWithPanel-<version>.dmg` and the single `sha256` line, both of which this project produces. The `openwithpanel.rb` attached to the draft can still be copied in by hand if a bump is needed immediately.

The tap's cask carries a `caveats` block with the `xattr -dr com.apple.quarantine` command, because releases from this fork are ad-hoc signed and not notarized: Homebrew installs them fine but macOS blocks the first launch. The same caveat applies to users who download the DMG directly, so keep the **First launch** section in the README as long as releases stay unnotarized.
