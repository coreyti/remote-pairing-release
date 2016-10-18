# Bootstrapping on GCP

A workflow for bootstrapping the BOSH Remote Pairing Release on `GCP`.

## Prerequisites

- [ ] [Sign up](https://cloud.google.com/compute/docs/signup) and activate a Google Cloud Platform (GCP) account
- [ ] Enable GCP API access [TODO](.)
- [ ] Ensure the following software are installed:
  - [ ] [`gcloud`](https://cloud.google.com/sdk/)
  - [ ] `git`
  - [ ] `ruby`
  - [ ] [`terraform`](https://www.terraform.io/downloads.html)

## Preparation

- [ ] Create a "workspace":

  ```bash
  #!/usr/bin/env bash
  set -euo pipefail

  : ${PROJECT_ID:?} # [required] GCP project ID.
  : ${WORKSPACE:?}  # [required] A directory path.
  : ${NAMESPACE:="vnc-$(date +%Y%m%d)"}
  : ${INSTALL_REGION:='us-west1'}
  : ${INSTALL_ZONE:='us-west1-a'}
  : ${INSTALL_CIDR:='10.9.8.0/24'}

  mkdir -p "${WORKSPACE}"
  ctx="$( cd "${WORKSPACE}" && pwd )"

  function outdent {
    awk 'NR==1 && match($0, /^ +/){n=RLENGTH} {print substr($0, n+1)}'
  }

  function envrc {
    outdent > "${WORKSPACE}/.envrc" << EOF
    #!/usr/bin/env bash

    : \${PROJECT_ID:='${PROJECT_ID}'}
    : \${WORKSPACE:='${WORKSPACE}'}
    : \${NAMESPACE:='${NAMESPACE}'}
    : \${INSTALL_REGION:='${INSTALL_REGION}'}
    : \${INSTALL_ZONE:='${INSTALL_ZONE}'}
    : \${RELEASE_GIT:='https://github.com/pivotal-cf-experimental/remote-pairing-release'}

    ctx='${ctx}'

    function name {
      echo "\${NAMESPACE}-\$1"
    }

    function acct {
      echo "\$(name account)@\${PROJECT_ID}.iam.gserviceaccount.com"
    }

    function outdent {
      awk 'NR==1 && match(\$0, /^ +/){n=RLENGTH} {print substr(\$0, n+1)}'
    }

    function require {
      printf "checking for '\$1' ... "

      if hash \$1 2>/dev/null ; then
        echo "found: \$(which \$1)"
        false
      else
        echo "installing:"
        true
      fi
    }
  EOF
  }

  function scaffold {
    pushd ${ctx} > /dev/null
      . .envrc

      [ ! -d './release' ] && \
        git clone ${RELEASE_GIT} release

      [ ! -d './install' ] && \
        mkdir -p install/{director,product}

      require bosh && gem install bosh_cli

      gcloud auth login
      gcloud config set project ${PROJECT_ID}
      gcloud config set compute/region ${INSTALL_REGION}
      gcloud config set compute/zone ${INSTALL_ZONE}
    popd > /dev/null
  }

  envrc && scaffold
  ```

- [ ] Bootstrap BOSH director:

  ```bash
  #!/usr/bin/env bash
  set -euo pipefail

  : ${WORKSPACE:?} # [required] A directory path.
  . ${WORKSPACE}/.envrc

  function generate-key {
    # Create key...
    gcloud iam service-accounts create $(name account)
    gcloud iam service-accounts keys create ./director-key.json \
      --iam-account $(acct)

    # Grant permissions (editor access)...
    gcloud projects add-iam-policy-binding ${PROJECT_ID} \
      --member serviceAccount:$(acct) \
      --role roles/editor
  }

  function generate-tf {
    outdent > './director-vars.tf' << EOF
    variable "namespace" {
      type = "string"
    }

    variable "project" {
      type = "string"
    }

    variable "region" {
      type = "string"
    }

    variable "cidr" {
      type = "string"
    }
EOF

    # Task: Generate BOSH director Terraform template
    # ----
    outdent > './director-tmpl.tf' << EOF
    provider "google" {
      project = "\${var.project}"
      region = "\${var.region}"
      credentials = "\${file("./director-key.json")}"
    }

    resource "google_compute_network" "director" {
      name       = "\${var.namespace}-director"
    }

    resource "google_compute_address" "director-ip" {
      name = "\${var.namespace}-director-ip"
    }

    resource "google_compute_subnetwork" "subnet" {
      name          = "\${var.namespace}-subnet"
      ip_cidr_range = "\${var.cidr}"
      network       = "\${google_compute_network.director.self_link}"
    }

    resource "google_compute_firewall" "director-internal" {
      name        = "\${var.namespace}-director-internal"
      network     = "\${google_compute_network.director.name}"
      source_tags = ["\${var.namespace}-director-internal"]
      target_tags = ["\${var.namespace}-director-internal"]

      allow {
        protocol = "icmp"
      }

      allow {
        protocol = "tcp"
      }

      allow {
        protocol = "udp"
      }
    }

    resource "google_compute_firewall" "director-external" {
      name        = "\${var.namespace}-director-external"
      network     = "\${google_compute_network.director.name}"
      target_tags = ["\${var.namespace}-director-external"]

      allow {
        protocol = "icmp"
      }

      allow {
        protocol = "tcp"
        ports    = ["22", "443", "4222", "6868", "25250", "25555", "25777", "8443"]
      }

      allow {
        protocol = "udp"
        ports    = ["53"]
      }
    }
EOF
  }

  function execute {
    terraform apply \
      -var namespace=${NAMESPACE} \
      -var project=${PROJECT_ID} \
      -var region=${INSTALL_REGION} \
      -var cidr=${INSTALL_CIDR}
  }

  pushd "${WORKSPACE}/install/director" > /dev/null
    generate-key
    generate-tf
    execute
  popd > /dev/null
  ```
