---
title: "BigSur(macOS 11.0)에서 pyenv 상의 python 설치 오류 해결"
date: 2020-01-03
categories:
  - env
tags:
  - macOS
  - pyenv
---

최근에 BigSur로 macOS 버전을 업데이트 한 이후, pyenv에서 새로운 파이썬 버전을 설치하는 경우 아래와 같은 오류가 발생하였습니다.

```bash
pyenv install 3.7.6
python-build: use openssl@1.1 from homebrew
python-build: use readline from homebrew
Downloading Python-3.7.6.tar.xz...
-> https://www.python.org/ftp/python/3.7.6/Python-3.7.6.tar.xz
error: failed to download Python-3.7.6.tar.xz

BUILD FAILED (OS X 11.0.1 using python-build 20180424)
```

Reference [1], [2]를 참고한 결과, brew를 이용하여 zlib, bzip2를 설치하고 아래와 같은 명령어로 설치할 수 있었습니다.
(\--patch 뒤의 버전을 바꾸어 다른 버전 또한 설치 가능합니다: 3.6.6, 3.8.0)

```bash
brew install zlib bzip2
```

```bash
CFLAGS="-I$(brew --prefix openssl)/include -I$(brew --prefix bzip2)/include -I$(brew --prefix readline)/include -I$(xcrun --show-sdk-path)/usr/include" LDFLAGS="-L$(brew --prefix openssl)/lib -L$(brew --prefix readline)/lib -L$(brew --prefix zlib)/lib -L$(brew --prefix bzip2)/lib" pyenv install --patch 3.7.5 < <(curl -sSL https://github.com/python/cpython/commit/8ea6353.patch\?full_index\=1) 
```

## Reference
  1. <https://github.com/pyenv/pyenv/issues/1643>
  2. <https://koji-kanao.medium.com/install-python-3-8-0-via-pyenv-on-bigsur-b4246987a548>

[1]: https://github.com/pyenv/pyenv/issues/1643
[2]: https://koji-kanao.medium.com/install-python-3-8-0-via-pyenv-on-bigsur-b4246987a548
