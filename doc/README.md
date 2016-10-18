# Bootstrapping on GCP

A workflow for bootstrapping the BOSH Remote Pairing Release on `GCP`.

## Prerequisites

The following software must be available:

- [ ] `git`
- [ ] `ruby`

## Preparation

- [ ] Create a "workspace":

  ```bash
  : ${PROVIDER:?}   # [required] One of: [GCP].
  : ${WORKSPACE:?}  # [required] A directory path.

  mkdir -p "${WORKSPACE}"
  ctx="$( cd "${WORKSPACE}" && pwd )"

  function outdent {
    awk 'NR==1 && match($0, /^ +/){n=RLENGTH} {print substr($0, n+1)}'
  }

  outdent > "${WORKSPACE}/.envrc" << EOF
    #!/usr/bin/env bash

    : \${PROVIDER:='${PROVIDER}'}
    : \${WORKSPACE:='${ctx}'}
    : \${RELEASE_GIT:='https://github.com/pivotal-cf-experimental/remote-pairing-release'}

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
  ```