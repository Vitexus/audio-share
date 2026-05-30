#!groovy

// Part of https://github.com/VitexSoftware/BuildImages

String[] distributions = ['debian:trixie', 'ubuntu:resolute']

String vendor = 'vitexsoftware'

properties([
    copyArtifactPermission('*'),
    buildBlocker(
        useBuildBlocker: true,
        blockLevel: 'GLOBAL',
        scanQueueFor: 'ALL',
        blockingJobs: 'RebulidDEBRepoByAnsible'
    )
])

node() {
    ansiColor('xterm') {
        stage('SCM Checkout') {
            checkout scm
        }
    }
}

def branches = [:]
distributions.each { distro ->
    branches[distro] = {
        def distroName = distro

        def dist = distroName.split(':')
        def distroFamily = dist[0]
        def distroCode = dist[1]
        def buildImage = ''
        def buildVer = ''

        node {
            ansiColor('xterm') {
                stage('Checkout ' + distroName) {
                    checkout scm
                    buildImage = docker.image(vendor + '/' + distro)
                    sh 'git checkout debian/changelog'
                    def VERSION = sh(
                        script: 'dpkg-parsechangelog --show-field Version',
                        returnStdout: true
                    ).trim()
                    buildVer = VERSION + '.' + env.BUILD_NUMBER + '~' + distroCode
                }
                stage('Build ' + distroName) {
                    buildImage.inside {
                        sh 'dch -b -v ' + buildVer + ' "' + env.BUILD_TAG + '"'
                        sh 'sudo apt-get update --allow-releaseinfo-change'
                        sh 'sudo chown jenkins:jenkins ..'
                        sh 'debuild-pbuilder -i -us -uc -b'
                        sh 'mkdir -p $WORKSPACE/dist/debian/ ; rm -rf $WORKSPACE/dist/debian/* ; for deb in $(cat debian/files | awk \'{print $1}\'); do mv "../$deb" $WORKSPACE/dist/debian/; done'
                    }
                }
                stage('Test ' + distroName) {
                    buildImage.inside {
                        sh 'cd $WORKSPACE/dist/debian/ ; dpkg-scanpackages . /dev/null | gzip -9c > Packages.gz; cd $WORKSPACE'
                        sh 'echo "deb [trusted=yes] file://///$WORKSPACE/dist/debian/ ./" | sudo tee /etc/apt/sources.list.d/local.list'
                        sh 'sudo apt-get update --allow-releaseinfo-change'
                        sh 'IFS="\n\b"; for package in `ls $WORKSPACE/dist/debian/ | grep .deb | grep -v dbgsym | awk -F_ \'{print $1}\'` ; do sudo DEBIAN_FRONTEND=noninteractive apt-get -y install $package ; done'
                    }
                }
                stage('Copy artifacts ' + distroName) {
                    buildImage.inside {
                        sh 'mv $WORKSPACE/dist/debian/*.deb $WORKSPACE'
                    }
                }
            }
        }
    }
}

parallel branches

node {
    stage('Publish to Aptly') {
        publishDebToAptly()
    }
}
