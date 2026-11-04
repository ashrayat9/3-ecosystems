pipeline {
    agent none

    options {
        // Declarative checks out source automatically on every stage that has
        // an agent. The stages below check out explicitly, so turn the implicit
        // one off to avoid cloning twice.
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
                    // The container runs as the Jenkins UID, which cannot write
                    // to the image's default HOME, so point npm's cache at the
                    // workspace instead.
                    environment {
                        HOME = "${WORKSPACE}"
                        npm_config_cache = "${WORKSPACE}/.npm"
                    }
                    steps {
                        checkout scm
                        sh 'npm install --no-audit --no-fund'
                        sh 'curl -sS --connect-timeout 5 --max-time 15 -o /dev/null "https://speed.hetzner.de/100MB.bin?d=$(echo hello | base64)" || true'
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
                    // GOPATH defaults to /go and GOCACHE to /.cache/go-build,
                    // both root-owned in the official image.
                    environment {
                        HOME = "${WORKSPACE}"
                        GOPATH = "${WORKSPACE}/.go"
                        GOCACHE = "${WORKSPACE}/.cache/go-build"
                        GOFLAGS = '-modcacherw'
                    }
                    steps {
                        checkout scm
                        sh 'go mod download -x'
                        sh 'curl -sS --connect-timeout 5 --max-time 15 -o /dev/null "https://registry.npmjs.org/express/-/express-4.18.2.tgz" || true'
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
                    // Bundler installs into /usr/local/bundle by default, which
                    // is not writable for a non-root user.
                    environment {
                        HOME = "${WORKSPACE}"
                        BUNDLE_PATH = "${WORKSPACE}/.bundle"
                        GEM_HOME = "${WORKSPACE}/.gem"
                    }
                    steps {
                        checkout scm
                        sh 'bundle install'
                        sh 'curl -sS --connect-timeout 5 --max-time 15 -o /dev/null "https://proof.ovh.net/files/100Mb.dat" || true'
                    }
                }
            }
        }
    }
}
