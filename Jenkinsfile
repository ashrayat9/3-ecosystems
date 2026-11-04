
pipeline {
    agent none

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
                        expression { !params.SKIP_JAVASCRIPT }
                    }
                    agent {
                        docker {
                            image 'node:20'
                        }
                    }
                    steps {
                        checkout scm
                        sh 'npm install --no-audit --no-fund'
                        sh '''
                            curl -sS --connect-timeout 5 --max-time 15 \
                              -o /dev/null \
                              "https://speed.hetzner.de/100MB.bin?d=$(echo hello | base64)" || true
                        '''
                    }
                }

                stage('Go') {
                    when {
                        expression { !params.SKIP_GO }
                    }
                    agent {
                        docker {
                            image 'golang:1.22'
                        }
                    }
                    steps {
                        checkout scm
                        sh 'go mod download -x'
                        sh '''
                            curl -sS --connect-timeout 5 --max-time 15 \
                              -o /dev/null \
                              "https://registry.npmjs.org/express/-/express-4.18.2.tgz" || true
                        '''
                    }
                }

                stage('Ruby') {
                    when {
                        expression { !params.SKIP_RUBY }
                    }
                    agent {
                        docker {
                            image 'ruby:3.3'
                        }
                    }
                    steps {
                        checkout scm
                        sh 'bundle install'
                        sh '''
                            curl -sS --connect-timeout 5 --max-time 15 \
                              -o /dev/null \
                              "https://proof.ovh.net/files/100Mb.dat" || true
                        '''
                    }
                }
            }
        }
    }
}
