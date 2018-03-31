---
layout: post
title: "Avoiding AWS secrets in Terraform statefiles"
---

I've been using [Terraform](https://terraform.io) for managing my AWS account for a while. It's pretty snazzy, but there are still a couple of things that Terraform doesn't fully handle. For example, making an [IAM access key](https://www.terraform.io/docs/providers/aws/r/iam_access_key.html) in Terraform stores the secret key in the statefile. They've added support to store the secret key encrypted with a GPG key, but I'd much prefer to not have it end up in the statefile at all.

<!--more-->

Why do they even do this?
==============

When I first realized that the iam_access_key resource dropped the secret key into the statefile, I was puzzled. Terraform statefiles need to be accessible to anybody looking to update the Terraform, so if your use case is generating admin IAM keys, having them be stored there isn't ideal.

The other use case for IAM keys is why it's set up this way. If you're generating the IAM keys for services/systems instead of humans, storing them in the statefile is necessary. Terraform needs to know how to figure out the full desired state of resources on each run, so anything you're providing as input to other resources has to be written down. So in this case, storing the secret key in the statefile is what enables handing that secret key to other terraform resources.

Time for some monkeypatching
================

I looked into ways to get around this for my use case. One thing I wanted to avoid was rewriting the code for IAM key management. Based on that, I decided to shim the existing [terraform-provider-aws](https://github.com/terraform-providers/terraform-provider-aws) codebase and add just what I needed.

After reading Terraform's official provider docs, and skimming the terraform-provider-aws code, this turns out to be pretty easy. Making a provider takes some initial boilerplate:

{% gist akerl/eb0cc7ee46a2007550fa73d1e1c9384c %}

I'm only defining the one resource, but I'm copying the schema of the upstream AWS provider, and I've written up some wrapper code to instantiate a copy of that provider so I can use its `ConfigureFunc`:

{% gist akerl/11f1d50ab00c919c15b0ee224991b7c1 %}

So that gets me a working provider that can call the actual AWS provider. For most of the methods on my new resource, I just pass through directly to the real resource. I'm only concerned with the "Create" method, which I want to wrap so that it writes the access key to a local file and then deletes it from the resource so that it doesn't end up in the statefile:

{% gist akerl/c6fab286cd3d5ef9ae6b62549690d929 %}

Using my new provider
=======

Now that I've got my provider, using it in Terraform is straightforward:

{% gist akerl/197f0f85d6bbfa329c64490b89dcb4d1 %}

This drops my new secret key into the `creds/admin_user` file. Success!

The full codebase for this is is [available on GitHub](https://github.com/akerl/terraform-provider-awscreds).
