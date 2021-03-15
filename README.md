# notarize_macos_app


openssl x509 -inform der -in ~/Desktop/development.cer -noout -text

codesign --verify ${FOO_APP}

/usr/local/bin/FOO_APP-cli: code object is not signed at all

codesign --verify ${FOO_APP} --verbose

spctl --assess --verbose ${FOO_APP} 
codesign -dvv ${FOO_APP}

spctl -a -v --raw ${FOO_APP}


codesign --force --verify --verbose --sign < developer ID > ${FOO_APP} --entitlements cli_empty_macos.entitlements

Beginning in macOS 10.15, all software built after June 1, 2019, and distributed with Developer ID must be notarized.

xattr -d com.apple.quarantine

https://scriptingosx.com/2019/09/notarize-a-command-line-tool/


Much longer life!

Developer ID Certificates ( for package installer )

￼


Developer ID certificates
￼


security find-identity -p basic -v


https://appleid.apple.com/