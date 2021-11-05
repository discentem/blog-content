---
title: "Autopkg on Windows!"
date: 2021-10-21
draft: true
---

# What is Autopkg? 

From the [Autopkg](https://github.com/autopkg/autopkg/tree/dev#autopkg) readme: 

> AutoPkg is an automation framework for macOS software packaging and distribution, oriented towards the tasks one would normally perform manually to prepare third-party software for mass deployment to managed clients.

> ...

> With AutoPkg, we define these steps in a "Recipe" file in plist or yaml format, run automatically instead of by hand, and shared with others.


# Running Autopkg on Windows

1. Launch Powershell as Admin.
1. Inspect [bootstrap_autopkg.ps1](https://raw.githubusercontent.com/discentem/discentem-recipes/main/windows-recipes/bootstrap_autopkg.ps1).
1. If all seems well... 

    > `curl https://raw.githubusercontent.com/discentem/discentem-recipes/main/windows-recipes/bootstrap_autopkg.ps1 -outfile bootstrap_autopkg.ps1`
1. Launch a new Powershell session with a more lenient execution policy.

    > `powershell.exe -ExecutionPolicy Bypass`

1. Run bootstrap_autopkg.ps1

    > .\bootstrap_autopkg.ps1