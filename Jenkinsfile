pipeline {
    agent any

    options {
        timestamps()
        disableConcurrentBuilds()
    }

    environment {
        KOLLA_VENV = '/opt/kolla-venv'
        KOLLA_INVENTORY = "${WORKSPACE}/inventories/lab/multinode"
        KOLLA_CONFIG_PATH = "${WORKSPACE}/etc/kolla"
        PATH = "/opt/kolla-venv/bin:${env.PATH}"
        ANSIBLE_HOST_KEY_CHECKING = 'True'
    }

    stages {
        stage('Validate Deployer Dependencies') {
            steps {
                sh '''#!/usr/bin/env bash
set -euxo pipefail

echo "Running as: $(whoami)"
echo "HOME=${HOME}"

python - <<'PY'
import docker
import dbus
import netaddr
import yaml

print("docker:", docker.__version__)
print("dbus: available")
print("netaddr:", netaddr.__version__)
print("yaml: available")
PY

ansible-galaxy collection list |
grep -E 'ansible.utils|community.general'

ansible-doc \
  -t filter \
  ansible.utils.ipaddr >/dev/null

echo "Deployer dependencies are valid"
'''
            }
        }

        stage('Validate Files') {
            steps {
                sh '''#!/usr/bin/env bash
set -euxo pipefail

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

        stage('Prepare Secrets') {
            steps {
                withCredentials([
                    file(
                        credentialsId: 'kolla-passwords-yml',
                        variable: 'KOLLA_PASSWORDS_FILE'
                    )
                ]) {
                    sh '''#!/usr/bin/env bash
set -euxo pipefail

install -m 600 \
  "${KOLLA_PASSWORDS_FILE}" \
  "${KOLLA_CONFIG_PATH}/passwords.yml"
'''
                }
            }
        }

        stage('Connectivity Test') {
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

        stage('Prechecks') {
            steps {
                sshagent(credentials: ['openstack-aio-ssh-key']) {
                    sh '''#!/usr/bin/env bash
set -euxo pipefail

kolla-ansible prechecks \
  -i "${KOLLA_INVENTORY}" \
  --configdir "${KOLLA_CONFIG_PATH}" \
  --use-test-images
'''
                }
            }
        }

        stage('Deploy') {
            when {
                branch 'main'
            }

            steps {
                input message: 'Deploy OpenStack configuration?'

                sshagent(credentials: ['openstack-aio-ssh-key']) {
                    sh '''#!/usr/bin/env bash
set -euxo pipefail

kolla-ansible deploy \
  -i "${KOLLA_INVENTORY}" \
  --configdir "${KOLLA_CONFIG_PATH}" \
  --use-test-images
'''
                }
            }
        }

        stage('Post Deploy') {
            when {
                branch 'main'
            }

            steps {
                sshagent(credentials: ['openstack-aio-ssh-key']) {
                    sh '''#!/usr/bin/env bash
set -euxo pipefail

kolla-ansible post-deploy \
  -i "${KOLLA_INVENTORY}" \
  --configdir "${KOLLA_CONFIG_PATH}"
'''
                }
            }
        }

        stage('Smoke Tests') {
            when {
                branch 'main'
            }

            steps {
                sh '''#!/usr/bin/env bash
set -euxo pipefail

export OS_CLIENT_CONFIG_FILE="${KOLLA_CONFIG_PATH}/clouds.yaml"
export OS_CLOUD=kolla-admin

openstack token issue
openstack service list
openstack compute service list
openstack network agent list
openstack hypervisor list
'''
            }
        }
    }

    post {
        always {
            sh '''#!/usr/bin/env bash
rm -f "${KOLLA_CONFIG_PATH}/passwords.yml"
rm -f "${KOLLA_CONFIG_PATH}/clouds.yaml"
'''

            cleanWs()
        }
    }
}
