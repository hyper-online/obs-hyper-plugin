# OpenSSL static library for macOS

## Build commands

### 1. Build OpenSSL for x86_64

```zsh
$ cd <PROJECT_ROOT>/..
$ git clone git@github.com:openssl/openssl.git -b OpenSSL_1_1_1s --depth 1 --recurse-submodules
$ cp -r openssl openssl_x86
$ mv openssl openssl_arm64
$ cd openssl_x86
$ perl ./Configure --prefix=/usr/local --openssldir=/usr/local/openssl no-ssl3 no-ssl3-method no-zlib darwin64-x86_64-cc enable-ec_nistp_64_gcc_128
$ make
```

### 2. Build OpenSSL for arm64

```zsh
$ cd <PROJECT_ROOT>/../openssl_arm64
$ perl ./Configure --prefix=/usr/local --openssldir=/usr/local/openssl no-ssl3 no-ssl3-method no-zlib darwin64-arm64-cc enable-ec_nistp_64_gcc_128
$ make
```

### 3. Link 2 libraries

```zsh
$ cd <PROJECT_ROOT>
$ lipo -create ../openssl_x86/libcrypto.a ../openssl_arm64/libcrypto.a -output libcrypto.a
$ lipo -create ../openssl_x86/libssl.a ../openssl_arm64/libssl.a -output libssl.a
```
