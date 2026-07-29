pipeline {
    agent any

    options {
        timestamps()
        disableConcurrentBuilds()
    }

    environment {
        PATH = "/opt/kolla-venv/bin:${env.PATH}"
        KOLLA_INVENTORY = "${WORKSPACE}/inventories/lab/multinode"
        KOLLA_CONFIG_PATH = "${WORKSPACE}/etc/kolla"
        ANSIBLE_HOST_KEY_CHECKING = "True"
    }

    stages {
        stage('Validate Files') {
            steps {
                sh '''#!/usr/bin/env bash
set -euxo pipefail

echo "Bash version: ${BASH_VERSION}"

test -f "${KOLLA_INVENTORY}"
test -f "${KOLLA_CONFIG_PATH}/globals.yml"

ansible-inventory \
  -i "${KOLLA_INVENTORY}" \
  --graph

python - <<'PY'
import yaml

with open("etc/kolla/globals.yml", encoding="utf-8") as stream:
    yaml.safe_load(stream)

print("globals.yml syntax is valid")
PY
'''
            }
        }

        stage('Test SSH') {
            steps {
                sshagent(credentials: ['openstack-aio-ssh-key']) {
                    sh '''#!/usr/bin/env bash
set -euxo pipefail

ssh \
  -o BatchMode=yes \
  -o StrictHostKeyChecking=yes \
  vagrant@192.168.56.10 \
  'hostname && sudo -n id'
'''
                }
            }
        }

        stage('Test Ansible') {
            steps {
                sshagent(credentials: ['openstack-aio-ssh-key']) {
                    sh '''#!/usr/bin/env bash
set -euxo pipefail

ansible \
  -i "${KOLLA_INVENTORY}" \
  all \
  -m ping
'''
                }
            }
        }

        stage('Test Remote Sudo') {
            steps {
                sshagent(credentials: ['openstack-aio-ssh-key']) {
                    sh '''#!/usr/bin/env bash
set -euxo pipefail

ansible \
  -i "${KOLLA_INVENTORY}" \
  openstack-aio \
  -b \
  -m command \
  -a 'id'
'''
                }
            }
        }
    }

    post {
        always {
            cleanWs()
        }
    }
}

