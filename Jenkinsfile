// Unplugged build pipeline for up_openvpn_xor (UNP-8203).
//
// Builds the (amd64) OpenVPN server image — OpenVPN + Tunnelblick XOR/scramble
// patch — and publishes it to ghcr.io/eranunplugged/up_openvpn_xor under a
// date-based, arch-prefixed tag.
//
// IMPORTANT: this pipeline intentionally NEVER pushes or overwrites the
// `latest` tag — production provisioning still references it; version is
// controlled via OVPN_IMAGE_VERSION in Vault.
pipeline {
  agent { label 'bs1' }

  options {
    buildDiscarder(logRotator(numToKeepStr: '20', daysToKeepStr: '60'))
    timestamps()
  }

  parameters {
    booleanParam(
      name: 'PUSH_IMAGE',
      defaultValue: false,
      description: 'Push the built image to ghcr.io/eranunplugged/up_openvpn_xor. Auto-pushes on master. Never modifies the "latest" tag.'
    )
  }

  environment {
    REGISTRY = 'ghcr.io'
    IMAGE    = 'eranunplugged/up_openvpn_xor'
  }

  stages {
    stage('Prepare') {
      steps {
        script {
          env.BUILD_DATE_LABEL = sh(script: "date -u +%Y-%m-%dT%H:%M:%S%:z", returnStdout: true).trim()
          env.TAG_DATE  = sh(script: "date +%Y.%m.%d", returnStdout: true).trim()
          env.IMAGE_TAG = "amd64-${env.TAG_DATE}"
          env.IMAGE_REF = "${env.REGISTRY}/${env.IMAGE}:${env.IMAGE_TAG}"
          echo "Building ${env.IMAGE_REF}"
        }
      }
    }

    stage('Build') {
      steps {
        sh '''#!/bin/bash
          set -euo pipefail
          DOCKER_BUILDKIT=1 docker build \
            -f Dockerfile \
            -t "${IMAGE_REF}" .
        '''
      }
    }

    stage('Verify') {
      steps {
        sh '''#!/bin/bash
          set -euo pipefail
          echo "version: $(docker run --rm --entrypoint openvpn "${IMAGE_REF}" --version 2>&1 | head -1 || true)"
          # XOR/scramble patch must be compiled in
          docker run --rm --entrypoint sh "${IMAGE_REF}" -c 'strings /usr/local/sbin/openvpn | grep -q "^scramble$"' \
            && echo "scramble XOR patch: present" \
            || { echo "scramble XOR patch: MISSING"; exit 1; }
          # EasyRSA baked in
          docker run --rm --entrypoint sh "${IMAGE_REF}" -c '/opt/easyrsa/easyrsa version 2>/dev/null | grep -i version | head -1'
        '''
      }
    }

    stage('Push') {
      when {
        anyOf {
          branch 'master'
          expression { return params.PUSH_IMAGE }
        }
      }
      steps {
        withCredentials([usernamePassword(credentialsId: 'github_registry', usernameVariable: 'DOCKER_USERNAME', passwordVariable: 'DOCKER_PASSWORD')]) {
          sh '''#!/bin/bash
            set -euo pipefail
            echo "${DOCKER_PASSWORD}" | docker login ghcr.io -u "${DOCKER_USERNAME}" --password-stdin
            docker push "${IMAGE_REF}"
          '''
        }
        echo "Pushed ${IMAGE_REF}. The 'latest' tag was intentionally NOT modified."
      }
    }
  }

  post {
    always {
      sh 'docker image rm -f "${IMAGE_REF}" || true'
    }
  }
}
