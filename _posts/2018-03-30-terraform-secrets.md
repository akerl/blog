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

```
// Serve up the plugin
func Serve() {
	plugin.Serve(&plugin.ServeOpts{
		ProviderFunc: provider,
	})
}

func provider() terraform.ResourceProvider {
	return &schema.Provider{
		Schema: awsProvider().Schema,
		ResourcesMap: map[string]*schema.Resource{
			"awscreds_iam_access_key": resourceIamAccessKey(),
		},
		ConfigureFunc: configure,
	}
}

func configure(d *schema.ResourceData) (interface{}, error) {
    var w wrapper
    if err := w.init(awsProvider(), d); err != nil {
        return nil, err
    }
    return w, nil
}
```

I'm only defining the one resource, but I'm copying the schema of the upstream AWS provider, and I've written up some wrapper code to instantiate a copy of that provider so I can use its `ConfigureFunc`:

```
type wrapper struct {
    provider *schema.Provider
    config   interface{}
}

func (w *wrapper) init(p *schema.Provider, d *schema.ResourceData) error {
    w.provider = p
    config, err := p.ConfigureFunc(d)
    if err != nil {
        return err
    }
    w.config = config
    return nil
}

func (w wrapper) resource(name string) *schema.Resource {
    return w.provider.ResourcesMap[name]
}

func resolveProvider(provider terraform.ResourceProvider) *schema.Provider {
    return provider.(*schema.Provider)
}
```

So that gets me a working provider that can call the actual AWS provider. For most of the methods on my new resource, I just pass through directly to the real resource. I'm only concerned with the "Create" method, which I want to wrap so that it writes the access key to a local file and then deletes it from the resource so that it doesn't end up in the statefile:

```
func createIamAccessKey(d *schema.ResourceData, m interface{}) error {
	if err := realResource(m).Create(d, realConfig(m)); err != nil {
		return err
	}

	access := d.Id()
	secret := d.Get("secret").(string)

	for _, key := range keysToSuppress {
		if err := d.Set(key, ""); err != nil {
			return err
		}
	}

	contents := access + "\n" + secret + "\n"
	file := d.Get("file").(string)
	return ioutil.WriteFile(file, []byte(contents), 0600)
}
```

Using my new provider
=======

Now that I've got my provider, using it in Terraform is straightforward:

```
provider "awscreds" {
  region = "us-east-1"
}

resource "awscreds_iam_access_key" "admin" {
  user       = "admin_user"
  file       = "creds/admin_user"
}
```

This drops my new secret key into the `creds/admin_user` file. Success!

