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

Note the 40 character identifiers for the downloaded certificates.  You don't need the string.

`security find-identity -p basic -v`

### Set environment variables

```bash
export APP_DEV_ID=<Developer ID Application>
export PKG_DEV_ID=<Developer ID Installer>
export BUNDLE_ID="com.rusty.notarizetest"
export APPLE_USERNAME=<email>
export FOO_APP="foo"
export FOO_PKG=${FOO_APP}.pkg
```

### Verify the binary

#### Verify against apple system controls

```bash
spctl --assess --verbose ${FOO_APP}
spctl -a -v --raw ${FOO_APP}
```

#### Verify code signature

`codesign --verify ${FOO_APP} --verbose`


#### Read certificate ( if outside of KeyChain )

`openssl x509 -inform der -in ~/development.cer -noout -text`

### Code sign

```bash
codesign -dvv ${FOO_APP}
codesign --force --verify --verbose --sign < developer ID > ${FOO_APP} --entitlements cli_empty_macos.entitlements
```

### uploade zip to Apple’s Notarization Servers

```bash
xcrun altool --notarize-app \
             --primary-bundle-id "com.example.com" \
             --username "username@example.com" \
             --password "@keychain:Developer-altool" \
             --asc-provider "ABCD123456" \
             --file foo-cli.zip
```

### check status

```bash
xcrun altool --notarization-info "Your-Request-UUID" \
             --username "username@example.com" \                                    
             --password "@keychain:Developer-altool"   
```

### Fix submission 1


codesign --force --options runtime --timestamp --sign ${APP_DEV_ID} ${FOO_APP}
codesign --verify ${FOO_APP} --verbose

foo: valid on disk
foo: satisfies its Designated Requirement

pkgbuild --root ~/test-dir \
           --identifier ${BUNDLE_ID} \
           --version "1.0" \
           --install-location "/" \
           --sign ${PKG_DEV_ID} \
           foo.zip

xcrun altool --notarize-app \
             --primary-bundle-id ${BUNDLE_ID} \
             --username ${APPLE_USERNAME} \
             --password "@keychain:Developer-altool" \
             --asc-provider "ABCD123456" \
             --file foo.zip

// check full Cert Chain
pkgutil --check-signature foo.zip

spctl -vvv --assess --type exec ${FOO_APP}
foo: rejected

source=Unnotarized Developer ID
origin=Developer ID Application: <Developer + Team ID>


xcrun altool --notarization-info "xxxx" \
             --username ""username@example.com" \
             --password "@keychain:Developer-altool" 
             --output-format json

```

### Result from submission 2

>ITMS-90728: Invalid File Contents - The contents of the file foo.zip do not match the extension. Verify that the contents of the file are valid for the extension and upload again.

### Fix submission 2

<https://github.com/electron/electron-notarize> suggests I needed to add some `entitlements`:

```plist
    com.apple.security.cs.allow-jit
    com.apple.security.cs.allow-unsigned-executable-memory
```

#### Add entitlements

Create an empty app and add the entitlements required.  Copy that entitlements file.

```bash
codesign --entitlements cli_empty_macos.entitlements --force --options runtime --timestamp --sign ${APP_DEV_ID} ${FOO_APP}

foo-cli: valid on disk
foo-cli: satisfies its Designated Requirement
```

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

```bash
pkgbuild --root ~/test-dir \
           --identifier ${BUNDLE_ID} \
           --version "1.0" \
           --install-location "/" \
           --sign ${PKG_DEV_ID} \
           ${FOO_APP}1.0.pkg
```

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