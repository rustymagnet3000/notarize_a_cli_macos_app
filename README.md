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


### Verify the binary

#### Verify against apple system controls

```bash
spctl --assess --verbose ${FOO_APP}
spctl -a -v --raw ${FOO_APP}
```

#### Verify code signature

`codesign --verify ${FOO_APP} --verbose`

#### Find code signing certificate from KeyChain

`security find-identity -p basic -v`

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
             --file "build/hello-1.0.pkg"
```

### check status

```bash
xcrun altool --notarization-info "Your-Request-UUID" \
             --username "username@example.com" \                                    
             --password "@keychain:Developer-altool"   
```

### Remove apple's Security attribute on app ( BAD CHOICE )

`xattr -d com.apple.quarantine`

### Recovering from a failed log

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
      "path": "foo.zip/usr/local/bin/foo-cli",
      "message": "The binary is not signed with a valid Developer ID certificate.",
      "docUrl": null,
      "architecture": "x86_64"
    },
    {
      "severity": "error",
      "code": null,
      "path": "foo.zip/usr/local/bin/foo-cli",
      "message": "The signature does not include a secure timestamp.",
      "docUrl": null,
      "architecture": "x86_64"
    },
    {
      "severity": "error",
      "code": null,
      "path": "foo.zip/usr/local/bin/foo-cli",
      "message": "The executable does not have the hardened runtime enabled.",
      "docUrl": null,
      "architecture": "x86_64"
    }
  ]
}
```

### Fixing issue

```bash
codesign --force --options runtime --timestamp --sign <Developer ID> ${FOO_APP}
codesign --verify ${FOO_APP} --verbose

foo-cli: valid on disk
foo-cli: satisfies its Designated Requirement

pkgbuild --root ~/test-dir \
           --identifier "com.rusty.foo-cli" \
           --version "1.0" \
           --install-location "/" \
           --sign <Developer ID Installer> \
           foo-cli.zip

// check full Cert Chain
pkgutil --check-signature foo-cli.zip

spctl -vvv --assess --type exec test-dir/foo-cli 
test-dir/foo-cli: rejected

source=Unnotarized Developer ID
origin=Developer ID Application: <Developer + Team ID>


xcrun altool --notarization-info "xxxx" \
             --username ""username@example.com" \
             --password "@keychain:Developer-altool" 
             --output-format json

```

##### Result

>ITMS-90728: Invalid File Contents - The contents of the file foo-cli.zip do not match the extension. Verify that the contents of the file are valid for the extension and upload again.
