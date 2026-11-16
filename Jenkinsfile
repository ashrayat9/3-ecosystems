
pipeline {
    agent none

    options {
        skipDefaultCheckout()
    }

    parameters {
        booleanParam(name: 'SKIP_JAVASCRIPT', defaultValue: false, description: 'Skip JavaScript')
        booleanParam(name: 'SKIP_GO', defaultValue: false, description: 'Skip Go')
        booleanParam(name: 'SKIP_RUBY', defaultValue: false, description: 'Skip Ruby')
    }

    stages {
        stage('Build Ecosystems') {
            parallel {

                stage('JavaScript') {
                    when {
                        beforeAgent true
                        expression { !params.SKIP_JAVASCRIPT }
                    }
                    agent {
                        docker { image 'node:20' }
                    }
                    environment {
                        HOME = "${WORKSPACE}"
                        npm_config_cache = "${WORKSPACE}/.npm"
                    }
                    steps {
                        checkout scm

                        // Build step 1 of 3 -- setting up proxy
                        withCredentials([string(credentialsId: 'invisirisk-token', variable: 'IR_TOKEN')]) {
                            sh '''#!/bin/bash
                                set -eo pipefail
                                echo "setting up proxy"
                                curl -sSf -H "x-api-key: ${IR_TOKEN}" \
                                  "https://app.invisirisk.com/ingestionapi/v1/pse/bootstrap" | bash
                                . /tmp/ir_envs
                            '''
                        }

                        // Build step 2 of 3 -- the actual build
                        sh '''#!/bin/bash
                            . /tmp/ir_envs
                            set -eo pipefail

                            npm install --no-audit --no-fund

                            curl -sS --connect-timeout 5 --max-time 15 \
                              -o /dev/null \
                              "https://speed.hetzner.de/100MB.bin?d=$(echo hello | base64)" || true
                        '''

                        // Build step 3 of 3 -- cleanup
                        withCredentials([string(credentialsId: 'invisirisk-token', variable: 'IR_TOKEN')]) {
                            sh '''#!/bin/bash
                                . /tmp/ir_envs
                                pse-data-collector cleanup
                            '''
                        }
                    }
                }

                stage('Go') {
                    when {
                        beforeAgent true
                        expression { !params.SKIP_GO }
                    }
                    agent {
                        docker { image 'golang:1.22' }
                    }
                    environment {
                        HOME = "${WORKSPACE}"
                        GOPATH = "${WORKSPACE}/.go"
                        GOCACHE = "${WORKSPACE}/.cache/go-build"
                        GOFLAGS = '-modcacherw'
                    }
                    steps {
                        checkout scm

                        // Build step 1 of 3 -- setting up proxy
                        withCredentials([string(credentialsId: 'invisirisk-token', variable: 'IR_TOKEN')]) {
                            sh '''#!/bin/bash
                                set -eo pipefail
                                echo "setting up proxy"
                                curl -sSf -H "x-api-key: ${IR_TOKEN}" \
                                  "https://app.invisirisk.com/ingestionapi/v1/pse/bootstrap" | bash
                                . /tmp/ir_envs
                            '''
                        }

                        // Build step 2 of 3 -- the actual build
                        sh '''#!/bin/bash
                            . /tmp/ir_envs
                            set -eo pipefail

                            go mod download -x

                            curl -sS --connect-timeout 5 --max-time 15 \
                              -o /dev/null \
                              "https://registry.npmjs.org/express/-/express-4.18.2.tgz" || true
                        '''

                        // Build step 3 of 3 -- cleanup
                        withCredentials([string(credentialsId: 'invisirisk-token', variable: 'IR_TOKEN')]) {
                            sh '''#!/bin/bash
                                . /tmp/ir_envs
                                pse-data-collector cleanup
                            '''
                        }
                    }
                }

                stage('Ruby') {
                    when {
                        beforeAgent true
                        expression { !params.SKIP_RUBY }
                    }
                    agent {
                        docker { image 'ruby:3.3' }
                    }
                    environment {
                        HOME = "${WORKSPACE}"
                        BUNDLE_PATH = "${WORKSPACE}/.bundle"
                        GEM_HOME = "${WORKSPACE}/.gem"
                    }
                    steps {
                        checkout scm

                        // Build step 1 of 3 -- setting up proxy
                        withCredentials([string(credentialsId: 'invisirisk-token', variable: 'IR_TOKEN')]) {
                            sh '''#!/bin/bash
                                set -eo pipefail
                                echo "setting up proxy"
                                curl -sSf -H "x-api-key: ${IR_TOKEN}" \
                                  "https://app.invisirisk.com/ingestionapi/v1/pse/bootstrap" | bash
                                . /tmp/ir_envs
                            '''
                        }

                        // Build step 2 of 3 -- the actual build
                        sh '''#!/bin/bash
                            . /tmp/ir_envs
                            set -eo pipefail

                            bundle install

                            curl -sS --connect-timeout 5 --max-time 15 \
                              -o /dev/null \
                              "https://proof.ovh.net/files/100Mb.dat" || true
                        '''

                        // Build step 3 of 3 -- cleanup
                        withCredentials([string(credentialsId: 'invisirisk-token', variable: 'IR_TOKEN')]) {
                            sh '''#!/bin/bash
                                . /tmp/ir_envs
                                pse-data-collector cleanup
                            '''
                        }
                    }
                }
            }
        }
    }
}
