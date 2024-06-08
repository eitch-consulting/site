---
title: ACME SSL certificates on terraform
date: 2024-06-08T13:10:15-03:00
draft: false
language: en
featured_image: ../assets/images/featured/featured-ssl-automation.jpg
summary:
  While automating your infrastructure, keeping SSL certificates updated is important and sometimes
  overlooked (since I do this one time per year, why automate?), and can lead to some bad scenarios. Automating
  the renew of them can give us peace of mind, so let's do this with the ACME protocol and terraform.
categories: Tech-Blog
tags:
- automation
- terraform
- security
- ssl
- certificates
---

While automating your infrastructure, keeping SSL certificates updated is important and sometimes
overlooked (since I do this one time per year, why automate?), and can lead to some bad scenarios. Automating
the renew of them can give us peace of mind, so let's do this with the ACME protocol and terraform.

When interacting with an ACME protocol, you first create an account, then you request a certificate
to be signed by the authority behind the protocol. These steps are automated so you don't have to actually
pay attention on what you're doing, and thus it can be performed on a regular basis using terraform.

In this tutorial, we'll use Let's Encrypt and an acme terraform provider to generate our valid certificate
and keep renewing it automatically.

## Setting up terraform and acme provider

We'll use the third-party acme terraform provider
[vancluever/acme](https://registry.terraform.io/providers/vancluever/acme/latest). This provider uses the
[LEGO](https://go-acme.github.io/lego/) library to talk with ACME and DNS providers to do the work.

Also, I'll be using examples with Google Cloud and AWS Route53 DNS providers, but there's dozens of other
providers that you check the configuration in the [LEGO DNS Providers](https://go-acme.github.io/lego/dns/) page.

First, configure terraform to use the required providers and their versions by beginning to write code on a
file with the `.tf` extension (maybe `main.tf`?)

```hcl
terraform {
  required_version = "~> 1.8.0"

  required_providers {
    tls = {
      source  = "hashicorp/tls"
      version = "~> 4.0.0"
    }

    acme = {
      source  = "vancluever/acme"
      version = "~> 2.2.0"
    }

    google = {
      source  = "hashicorp/google"
      version = "~> 5.32.0"
    }

    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.53.0"
    }
  }
}
```

Now you'll have to configure which ACME server you'll use. You do this by configuring the provider:

```hcl
provider "acme" {
  # let's encrypt staging
  server_url = "https://acme-staging-v02.api.letsencrypt.org/directory"

  # let's encrypt production
  #server_url = "https://acme-v02.api.letsencrypt.org/directory"

  # zerossl production
  #server_url = "https://acme.zerossl.com/v2/DV90"
}
```

In the example, only the `server_url` with Let's Encrypt staging environment is uncommented. This means
that all requests will be for testing purposes only, so you can mess around without worrying getting locked out
due to flooding, etc.

When you succeed with your tests, you can change the `server_url` to the production environment.

Also note that I'm not putting any provider configuration about Google Cloud or AWS because this is not the
scope of this tutorial. But if the command line you're running terraform has access to these clouds, you should
be OK.

## Creating the resources

In order to create the certificate, we must do these steps:

1. Create an account private key
2. Sign Up for an account using the created private key and a contact email address
3. Request a certificate using this newly created account.

Everything is automated, so you will need only this terraform code:

```hcl
# ------------------------------------
# ACME ACCOUNT
# ------------------------------------
resource "tls_private_key" "account_private_key" {
  algorithm = "RSA"
}

resource "acme_registration" "reg" {
  account_key_pem = tls_private_key.account_private_key.private_key_pem
  email_address   = "eitch@example.com"
}

# -------------------------------------
# CERTIFICATE
# -------------------------------------
resource "acme_certificate" "certificate" {
  account_key_pem           = acme_registration.reg.account_key_pem
  min_days_remaining        = "60"
  common_name               = "example.com"
  subject_alternative_names = [
    "example.com",
    "*.example.com"
  ]

  dns_challenge {
    provider = "gcloud"
    config   = {
      GCE_PROJECT = "my-example-project-id"
    }
  }
}
```

Each of the these resources corresponds to a step that we must do to generate a certificate.

- Change the `email_address`. The server doesn't accept any `@example.com` :)

- Replace `common_name` with the main domain you want to create the certificate to

- In `subject_alternative_names`, you can put a list of domain aliases which the certificate
  would also be valid. Notice that I'm using a wildcard certificate in the example, and this works!

- I'm using the Google Cloud DNS provider, but I can also use AWS Route53:

    ```hcl
    dns_challenge {
      provider = "route53"
      config   = {
        AWS_HOSTED_ZONE_ID = "Z2U8GN29DN0921B23OCS"
      }
    }
    ```

    ...or any other DNS provider. There's lots of them on LEGO!

## Defining the outputs

As a bonus and in order to use this certificate on external modules or when using as a dependency
with terragrunt, you could also create these outputs:

```hcl
# ------------------------------------
# ACME ACCOUNT
# ------------------------------------
output "account_id" {
  description = "The original full URL of the account."
  value       = acme_registration.reg.id
}

output "account_registration_url" {
  description = "The current full URL of the account."
  value       = acme_registration.reg.registration_url
}

output "account_key_pem" {
  description = "The private key used to identify the account."
  value       = acme_registration.reg.account_key_pem
  sensitive   = true
}

output "account_email_address" {
  description = "The contact email address for the account."
  value       = acme_registration.reg.email_address
}

# -------------------------------------
# CERTIFICATE
# -------------------------------------
output "certificate_url" {
  description = "The full URL of the certificate within the ACME CA."
  value       = acme_certificate.certificate.certificate_url
}

output "certificate_domain" {
  description = "The common name of the certificate."
  value       = acme_certificate.certificate.certificate_domain
}

output "certificate_private_key_pem" {
  description = "The certificate's private key, in PEM format."
  value       = acme_certificate.certificate.private_key_pem
  sensitive   = true
}

output "certificate_pem" {
  description = "The certificate in PEM format."
  value       = acme_certificate.certificate.certificate_pem
}

output "certificate_issuer_pem" {
  description = "The intermediate certificates of the issuer. Multiple certificates are concatenated in this field when there is more than one intermediate certificate in the chain."
  value       = acme_certificate.certificate.issuer_pem
}
```

## Running

The same and out 3 commands should do the trick, if you didn't run them yet:

```bash
terraform init
terraform plan
terraform apply
```

To get the certificate after it's created:

```bash
terraform output -raw certificate_pem
```

And the private key for the certificate:

```bash
terraform output -raw certificate_private_key_pem
```

## Conclusion and references

You should be able to run this within a crontab or in a pipeline inside a CI/CD and when run regularly, it will
renew the certificate when there's 30- days remaining to expire, and update it on other systems automatically. You
probably won't need to remember about when to renew.

ANYWAY, even if automated, don't forget to set up some monitoring and alarm to warn you if this automation doesn't
work... Nothing is infallible.

References:

- [Provider vancluever/acme documentation](https://registry.terraform.io/providers/vancluever/acme/latest/docs)
- [LEGO](https://go-acme.github.io/lego/)
- [terraform-acme-certificate](https://github.com/nephosolutions/terraform-acme-certificate) -
  example of a terraform module using the provider

## Example outputs

This is an extra: raw output on running terraform in this tutorial.

```plain
$ terraform init

Initializing the backend...

Initializing provider plugins...
- Finding hashicorp/google versions matching "~> 5.32.0"...
- Finding hashicorp/tls versions matching "~> 4.0.0"...
- Finding vancluever/acme versions matching "~> 2.2.0"...
- Installing hashicorp/google v5.32.0...
- Installed hashicorp/google v5.32.0 (signed by HashiCorp)
- Installing hashicorp/tls v4.0.5...
- Installed hashicorp/tls v4.0.5 (signed by HashiCorp)
- Installing vancluever/acme v2.2.0...
- Installed vancluever/acme v2.2.0 (self-signed, key ID AD79F2BFFFD5B46C)

Partner and community providers are signed by their developers.
If you'd like to know more about provider signing, you can read about it here:
https://www.terraform.io/docs/cli/plugins/signing.html

Terraform has created a lock file .terraform.lock.hcl to record the provider
selections it made above. Include this file in your version control repository
so that Terraform can guarantee to make the same selections by default when
you run "terraform init" in the future.

Terraform has been successfully initialized!

You may now begin working with Terraform. Try running "terraform plan" to see
any changes that are required for your infrastructure. All Terraform commands
should now work.

If you ever set or change modules or backend configuration for Terraform,
rerun this command to reinitialize your working directory. If you forget, other
commands will detect it and remind you to do so if necessary.
```

```plain
$ terraform plan

Terraform used the selected providers to generate the following execution plan. Resource actions are indicated with the following symbols:
  + create

Terraform will perform the following actions:

  # acme_certificate.certificate will be created
  + resource "acme_certificate" "certificate" {
      + account_key_pem              = (sensitive value)
      + certificate_domain           = (known after apply)
      + certificate_p12              = (sensitive value)
      + certificate_pem              = (known after apply)
      + certificate_url              = (known after apply)
      + common_name                  = "example.com"
      + disable_complete_propagation = false
      + id                           = (known after apply)
      + issuer_pem                   = (known after apply)
      + key_type                     = "2048"
      + min_days_remaining           = 60
      + must_staple                  = false
      + pre_check_delay              = 0
      + private_key_pem              = (sensitive value)
      + subject_alternative_names    = [
          + "*.example.com",
          + "example.com",
        ]

      + dns_challenge {
          + config   = (sensitive value)
          + provider = "gcloud"
        }
    }

  # acme_registration.reg will be created
  + resource "acme_registration" "reg" {
      + account_key_pem  = (sensitive value)
      + email_address    = "eitch@example.com"
      + id               = (known after apply)
      + registration_url = (known after apply)
    }

  # tls_private_key.account_private_key will be created
  + resource "tls_private_key" "account_private_key" {
      + algorithm                     = "RSA"
      + ecdsa_curve                   = "P224"
      + id                            = (known after apply)
      + private_key_openssh           = (sensitive value)
      + private_key_pem               = (sensitive value)
      + private_key_pem_pkcs8         = (sensitive value)
      + public_key_fingerprint_md5    = (known after apply)
      + public_key_fingerprint_sha256 = (known after apply)
      + public_key_openssh            = (known after apply)
      + public_key_pem                = (known after apply)
      + rsa_bits                      = 2048
    }

Plan: 3 to add, 0 to change, 0 to destroy.

Changes to Outputs:
  + account_email_address       = "eitch@example.com"
  + account_id                  = (known after apply)
  + account_key_pem             = (sensitive value)
  + account_registration_url    = (known after apply)
  + certificate_domain          = (known after apply)
  + certificate_issuer_pem      = (known after apply)
  + certificate_pem             = (known after apply)
  + certificate_private_key_pem = (sensitive value)
  + certificate_url             = (known after apply)

─────────────────────────────────────────────────────────────────────────────────────────────────────────────────

Note: You didn't use the -out option to save this plan, so Terraform can't guarantee to take exactly these actions if you run "terraform apply" now.
```

```plain
$ terraform apply
[...]

tls_private_key.account_private_key: Creating...
tls_private_key.account_private_key: Creation complete after 0s [id=584c9e45c9894857957e9aecaf36357c]
acme_registration.reg: Creating...
acme_registration.reg: Creation complete after 2s [id=https://acme-staging-v02.api.letsencrypt.org/acme/acct/1234567890]
acme_certificate.certificate: Creating...
acme_certificate.certificate: Still creating... [10s elapsed]
acme_certificate.certificate: Still creating... [20s elapsed]
acme_certificate.certificate: Still creating... [30s elapsed]
acme_certificate.certificate: Still creating... [40s elapsed]
acme_certificate.certificate: Still creating... [50s elapsed]
acme_certificate.certificate: Creation complete after 57s [id=eb1cc575-4427-4f72-856f-d1c137bf7edd]

Apply complete! Resources: 3 added, 0 changed, 0 destroyed.

Outputs:

account_email_address = "eitch@example.com"
account_id = "https://acme-staging-v02.api.letsencrypt.org/acme/acct/1234567890"
account_key_pem = <sensitive>
account_registration_url = "https://acme-staging-v02.api.letsencrypt.org/acme/acct/1234567890"
certificate_domain = "example.com"
certificate_issuer_pem = <<EOT
-----BEGIN CERTIFICATE-----
...
-----END CERTIFICATE-----

EOT
certificate_pem = <<EOT
-----BEGIN CERTIFICATE-----
...
-----END CERTIFICATE-----

EOT
certificate_private_key_pem = <sensitive>
certificate_url = "https://acme-staging-v02.api.letsencrypt.org/acme/cert/6f8dbdc61221460f9c1295755677b787"
```
