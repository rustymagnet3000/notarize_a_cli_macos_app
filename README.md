# notarize_macos_app

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



Beginning in macOS 10.15, all software built after June 1, 2019, and distributed with Developer ID must be notarized.

Enabling the ‘Hardened Runtime’ will compile the binary in a way that makes it harder for external process to inject code. This will be requirement for successful notarization starting January 2020.


### Remove apple's Security attribute on app ( BAD CHOICE )

`xattr -d com.apple.quarantine`

https://scriptingosx.com/2019/09/notarize-a-command-line-tool/
https://appleid.apple.com/