# Nix Pills 中文版

《Nix Pills》是由 **Luca Bruno**（aka Lethalman）于 2014–2015 年撰写的 Nix 入门系列博客，被视为学习 Nix 的经典教程。本仓库是其简体中文翻译版，原文托管于 [NixOS/nix-pills](https://github.com/NixOS/nix-pills)。

在线阅读：[多页 HTML](https://nixos.org/guides/nix-pills/)（英文原版）或 [EPUB 电子书](https://nixos.org/guides/nix-pills/nix-pills.epub)。

本地构建：

    nix-build release.nix -A html-split && firefox result/share/doc/nix-pills/index.html

如需构建 [EPUB](https://www.w3.org/publishing/epub32/) 版本：

    nix-build release.nix -A epub && foliate result/share/doc/nix-pills/nix-pills.epub
