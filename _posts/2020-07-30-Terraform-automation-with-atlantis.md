---
layout: post
title: Terraform automation with atlantis
subtitle:
tags: [automation]
---

Plan and apply terraform state from a central place with Atlantis.

![Atlantis Logo](https://www.runatlantis.io/hero.png)


Terraform is a great tool, even if there's no major version released yet (currently 0.12.27), it's a production ready tool.

Terraform 0.12 actually came with great improvements.

But using daily Terraform can come with some dowmside, depending on how you use it, for example:


#### Versionning

Not pinning your terraform module will inevitably end up with someone applying resources with a higher terraform version than yours.

A guy from your team just applied a state with a newer terraform version and changed the state version (which is now impossible to downgrade), and potentially your actual version might not work anymore.

#### Bad habits

So a guy might develop something, on his branch, apply resources and doesn't merge it, because he's lazy or simply forgot.

If you try to apply on the same resources (which is not merge into master yet), you might end up with changes/destroy to apply.

#### Not enough knowledge

Terraform is infrastructure as code, right. Everybody should understand how it works by reading a .tf file, *false* !

Depending on the complexity of the .tf files, it might be not obvious at all, so please, put a comment when the action you want to do is not evident.

Also, not everybody is a Terraform expert, having the team member constantly reviewing code help them to grow on their terraform's journey.

## Enter Atlantis

Atlantis has a complete documentation: [https://www.runatlantis.io/](https://www.runatlantis.io/)

To sum up:

- Main installation are: docker, k8s, binary
- Install terraform as well
- Works with github, gitlab, bitbucket
- Send a webhook to atlantis, which gonna perform terraform actions automatically and pre-defined comment on a pull request
- Improve team work and visibility


Atlantis automatically run a plan on a pull request. This one has failed because of syntax error.

![Atlantis Failed](https://github.com/ptran32/ptran32.github.io/blob/master/_posts/img/01-atlantis-failed.png?raw=true)


After fixing the syntax issue, verify that the plan output is ok, you or a team member can apply it by a comment: "atlantis apply" 

![Atlantis Succes](https://github.com/ptran32/ptran32.github.io/blob/master/_posts/img/02-atlantis-success.png?raw=true)

You can now merge your changes into master ;)


## Conclusion

This is the kind of tool you need if you want to have a central place where to manage your infrastructure.

Everybody is aware of the changes, who made it, and when.

It eliminates the "I work on my own stuff" and improve team visibility on your projects.