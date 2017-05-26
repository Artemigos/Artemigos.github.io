---
layout: post
title: "Assembly versioning with MSBuild"
# date: 2017-05-26 12:00:00 +0200
categories: coding
tags: msbuild .net
---

Ah, the problem of assembly versioning - always there to cause headaches. Maybe it's time to solve it once and for all?

# Requirements
This is what **I** want from a versioning system:

- vanilla MSBuild only - I don't want to depend on any external stuff for every single project
- partial SemVer is possible but not required - store a pre-release label in assembly
- easily automated versioning - it should be as easy as passing an argument to MSBuild, enabling versioning debug builds and automated releases
- no changes in the code repository - when I run a build locally I don't want any new files or modifications to existing ones
- extra: no need to modify every project in solution
