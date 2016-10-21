# Bootstrapping on GCP

A workflow for bootstrapping the BOSH Remote Pairing Release on `GCP`.

## Prerequisites

- [ ] [Sign up](https://cloud.google.com/compute/docs/signup) and activate a Google Cloud Platform (GCP) account
- [ ] Enable GCP API access [TODO](.)
- [ ] Ensure the following software are installed:
  - [ ] [`bosh-init`](https://bosh.io/docs/install-bosh-init.html)
  - [ ] [`gcloud`](https://cloud.google.com/sdk/)
  - [ ] [`git`](https://git-scm.com/)
  - [ ] [`ruby`](https://www.ruby-lang.org/)
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
    : \${INSTALL_CIDR:='${INSTALL_CIDR}'}
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

- [ ] Bootstrap BOSH director (i.e., set up IaaS):

  ```bash
  #!/usr/bin/env bash
  set -euo pipefail

  : ${WORKSPACE:?} # [required] A directory path.
  . ${WORKSPACE}/.envrc

  function generate-keys {
    # Create GCP service account...
    gcloud iam service-accounts create $(name account)

    # Grant permissions (editor access)...
    gcloud projects add-iam-policy-binding ${PROJECT_ID} \
      --member serviceAccount:$(acct) \
      --role roles/editor

    # Create service account JSON key...
    gcloud iam service-accounts keys create ./director-acct.json \
      --iam-account $(acct)

    # Create BOSH director SSH key...
    ssh-keygen -b 2048 -t rsa -C vcap@director -f director.key -P ''
  }

  function generate-tf {
    outdent > './director-out.tf' << EOF
    output "ip" {
      value = "\${google_compute_address.director-ip.address}"
    }
  EOF

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
      credentials = "\${file("./director-acct.json")}"
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
    generate-keys
    generate-tf
    execute
  popd > /dev/null
  ```

- [ ] Deploy BOSH director:

  ```bash
  #!/usr/bin/env bash
  set -euo pipefail

  : ${WORKSPACE:?} # [required] A directory path.
  . ${WORKSPACE}/.envrc

  addr_base=$(echo ${INSTALL_CIDR} | cut -d . -f1-3)
  function addr {
    echo "${addr_base}.$1"
  }

  function dump {
    cat $1 | tr "\n" " "
  }

  function generate-yml {
    public_ip=$(terraform output ip)
    outdent > './director.yml' << EOF
    ---
    name: bosh

    releases:
    - name: bosh
      url: https://bosh.io/d/github.com/cloudfoundry/bosh?v=257.15
      sha1: f4cf3579bfac994cd3bde4a9d8cbee3ad189c8b2
    - name: bosh-google-cpi
      url: https://bosh.io/d/github.com/cloudfoundry-incubator/bosh-google-cpi-release?v=25.5.0
      sha1: 974c097f08920b7d9f3a7a2f6fe64495c387a644

    # platform
    # ----------------------------------------------------------
    disk_pools:
    - name: disks
      disk_size: 20000
      cloud_properties: { type: pd-standard }

    networks:
    - name: private
      type: manual
      subnets:
      - range: ${INSTALL_CIDR}
        gateway: $(addr 1)
        cloud_properties:
          network_name:     ${NAMESPACE}-director
          subnetwork_name:  ${NAMESPACE}-subnet
          tags:
            - ${NAMESPACE}-director-internal
            - ${NAMESPACE}-director-external
    - name: public
      type: vip

    resource_pools:
    - name: vms
      network: private
      stemcell:
        url: https://bosh.io/d/stemcells/bosh-google-kvm-ubuntu-trusty-go_agent?v=3263.7
        sha1: a09ce8b4acfa9876f52ee7b4869b4b23f27d5ace
      cloud_properties:
        machine_type: n1-standard-2
        root_disk_size_gb: 40
        root_disk_type: pd-standard
        service_scopes:
          - compute
          - devstorage.full_control
        zone: ${INSTALL_ZONE}

    jobs:
    - name: bosh
      instances: 1

      templates:
      - {name: nats, release: bosh}
      - {name: postgres, release: bosh}
      - {name: blobstore, release: bosh}
      - {name: director, release: bosh}
      - {name: health_monitor, release: bosh}
      - {name: registry, release: bosh}
      - {name: google_cpi, release: bosh-google-cpi}

      resource_pool: vms
      persistent_disk_pool: disks

      networks:
      - name: private
        static_ips: [$(addr 6)]
        default: [dns, gateway]
      - name: public
        static_ips: [${public_ip}]

      properties:
        nats:
          address: 127.0.0.1
          user: nats
          password: nats-password

        postgres: &db
          listen_address: 127.0.0.1
          host: 127.0.0.1
          user: postgres
          password: postgres-password
          database: bosh
          adapter: postgres

        registry:
          address: $(addr 6)
          host: $(addr 6)
          db: *db
          http: {user: admin, password: admin, port: 25777}
          username: admin
          password: admin
          port: 25777

        blobstore:
          address: $(addr 6)
          port: 25250
          provider: dav
          director: {user: director, password: director-password}
          agent: {user: agent, password: agent-password}

        director:
          address: 127.0.0.1
          name: bosh
          db: *db
          cpi_job: google_cpi
          max_threads: 10
          user_management:
            provider: local
            local:
              users:
              - {name: admin, password: admin}
              - {name: hm, password: hm-password}

        hm:
          director_account: {user: hm, password: hm-password}
          resurrector_enabled: true

        google: &google
          project: ${PROJECT_ID}
          json_key: '$(dump 'director-acct.json')'

        agent: {mbus: "nats://nats:nats-password@$(addr 6):4222"}

        ntp: &ntp [0.pool.ntp.org, 1.pool.ntp.org]

    cloud_provider:
      template: {name: google_cpi, release: bosh-google-cpi}

      ssh_tunnel:
        host: ${public_ip}
        port: 22
        user: vcap
        private_key: ./director.key

      mbus: "https://mbus:mbus-password@${public_ip}:6868"

      properties:
        google: *google
        agent: {mbus: "https://mbus:mbus-password@0.0.0.0:6868"}
        blobstore: {provider: local, path: /var/vcap/micro_bosh/data/cache}
        ntp: *ntp
  EOF
  }

  function execute {
    bosh-init deploy director.yml
  }

  pushd "${WORKSPACE}/install/director" > /dev/null
    generate-yml
    execute
  popd > /dev/null
  ```
