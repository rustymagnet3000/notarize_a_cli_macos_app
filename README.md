# Notarize a macOS app

## Why?

Apple [notarize](https://developer.apple.com/documentation/xcode/notarizing_macos_software_before_distribution/:
):
> Beginning in macOS 10.15, all software built after June 1, 2019, and distributed with Developer ID must be notarized.

> Enabling the ‘Hardened Runtime’ will be requirement for successful notarization starting January 2020.

### What you need

- 2 x `Code Signing certificates` from <https://developer.apple.com>
- 1 x `App-specific-passwords` from <https://appleid.apple.com/>
[Help to notarize](https://scriptingosx.com/2019/09/notarize-a-command-line-tool/)
- Manual step-by-step [here](https://eclecticlight.co/2019/06/13/building-and-delivering-command-tools-for-catalina/)
- Automated [tool](https://github.com/electron/electron-notarize)

#### Get Certificates from Apple Developer account

Get two signing certificates from <https://developer.apple.com>.

- Get a `Developer ID Application` for signing the binary
- Get a `Developer ID Installer` for signing the package [that contains the binary].

Download and install into `keyChain`.

#### List the code signing certificate from KeyChain

```bash
security find-identity -p basic -v            
  5) 629......005A5 "Developer ID Installer: Mickey Mouse (U8.......)"
  6) 79F......C0126 "Developer ID Application: Mickey Mouse  (U8.......)"
```

Copy the 40 character identifiers for the downloaded certificates.  Also note the Team ID. The Team ID ( `U8...`) is a 10 character GUID.

### Set environment variables

```bash
export APP_DEV_ID=<Developer ID Application>
export PKG_DEV_ID=<Developer ID Installer>
export BUNDLE_ID="com.rusty.notarizetest"
export INSTALL_LOC="/usr/local/bin/foo-cli"
export APPLE_USERNAME=<email>
export TEAM_ID=<>
export FOO_APP="foo"
export FOO_PKG=${FOO_APP}.pkg
```

### Verify Notarization is required

If any of the following fail, you need to `notarize`.

```bash
codesign --verify ${FOO_APP} --verbose
spctl --assess --verbose ${FOO_APP}
spctl -a -v --raw ${FOO_APP}
```

### Add entitlements

Create an empty `macOS command line` app and add the entitlements required.  Copy that entitlements file.  <https://github.com/electron/electron-notarize> suggests you needd some `entitlements`:

```plist
    com.apple.security.cs.allow-jit
    com.apple.security.cs.allow-unsigned-executable-memory
```

### Sign the binary with Entitlements + App Hardening

The `runtime` flag is required for the `notarization` to pass.  

```bash
codesign --entitlements cli_empty_macos.entitlements --force --options runtime --timestamp --sign ${APP_DEV_ID} ${FOO_APP}

foo-cli: valid on disk
foo-cli: satisfies its Designated Requirement
```

### Add the code into a Package file

```bash
pkgbuild --root ~/test-dir \
           --identifier ${BUNDLE_ID} \
           --version "1.0" \
           --install-location ${INSTALL_LOC} \
           --sign ${PKG_DEV_ID} \
           ${FOO_PKG}
```

### Verify package was signed correctly

`pkgutil --check-signature ${FOO_PKG}`

### Verify package was signed correctly

The following always fails, even when `notarized`:

```bash
spctl -vvv --assess --type exec ${FOO_PKG}
foo: rejected
```

### Submit for notarization

```bash
xcrun altool --notarize-app \                 
             --primary-bundle-id ${BUNDLE_ID} \
             --username ${APPLE_USERNAME} \
             --password "@keychain:Developer-altool" \
             --asc-provider "U8MSEX235L" \
             --file foo-cli1.0.pkg
```

### check status

```bash
xcrun altool --notarization-info "Your-Request-UUID" \
             --username ${APPLE_USERNAME} \                                    
             --password "@keychain:Developer-altool"   
```

source=Unnotarized Developer ID
origin=Developer ID Application: <Developer + Team ID>



```

### Result from submission 2

>ITMS-90728: Invalid File Contents - The contents of the file foo.zip do not match the extension. Verify that the contents of the file are valid for the extension and upload again.

### Fix submission 2





#### Check local system

```bash
spctl -vvv --assess --type exec ${FOO_APP}
source=Unnotarized Developer ID
origin=Developer ID Application: <Developer + Team ID>
```

### Result from submission 3

The same error.

>ITMS-90728: Invalid File Contents - The contents of the file foo-cli.zip do not match the extension. Verify that the contents of the file are valid for the extension and upload again.

### Submission 4



### Verify success

```bash
xcrun altool --list-apps \
             --username ${APPLE_USERNAME} \
             --password "@keychain:Developer-altool"
```

### Final step - staple the ticket to package

```bash
xcrun stapler staple foo-cli1.0.pkg                   
Processing: foo-cli1.0.pkg
The staple and validate action worked!
```



### Remove apple's Security attribute on app ( BAD CHOICE )

`xattr -d com.apple.quarantine`

### Result from submission 1

I had forgotten to sign the zip package.

```json
{
  "logFormatVersion": 1,
  "jobId": ".....",
  "status": "Invalid",
  "statusSummary": "Archive contains critical validation errors",
  "statusCode": 4000,
  "archiveFilename": "foo.zip",
  "uploadDate": "2021-03-15T16:19:39Z",
  "sha256": "....",
  "ticketContents": null,
  "issues": [
    {
      "severity": "error",
      "code": null,
      "path": "foo.zip/usr/local/bin/foo",
      "message": "The binary is not signed with a valid Developer ID certificate.",
      "docUrl": null,
      "architecture": "x86_64"
    },
    {
      "severity": "error",
      "code": null,
      "path": "foo.zip/usr/local/bin/foo",
      "message": "The signature does not include a secure timestamp.",
      "docUrl": null,
      "architecture": "x86_64"
    },
    {
      "severity": "error",
      "code": null,
      "path": "foo.zip/usr/local/bin/foo",
      "message": "The executable does not have the hardened runtime enabled.",
      "docUrl": null,
      "architecture": "x86_64"
    }
  ]
}
```