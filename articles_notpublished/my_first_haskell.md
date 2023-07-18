---
title: "My First Haskell"
emoji: "📌"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: []
published: false
---

メモです

インストールする:

https://www.haskell.org/downloads/

```shell-session:
C:\> Ubuntu
$ sudo apt-get install ghc cabal-install haskell-stack
$ ghc --version
The Glorious Glasgow Haskell Compilation System, version 8.6.5
$ stack upgrade
$ stack setup
$ echo 'main = putStrLn "Hello, World!"' > main.hs
$ ghc hello.hs
```

読む:

https://www.haskell.org/documentation/

