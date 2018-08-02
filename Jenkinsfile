openshift.withCluster() {
    env.NAMESPACE = openshift.project()
    env.POM_FILE = env.BUILD_CONTEXT_DIR ? "${env.BUILD_CONTEXT_DIR}/pom.xml" : "pom.xml"
    env.APP_NAME = "${env.JOB_NAME}".replaceAll(/-?pipeline-?/, '').replaceAll(/-?${env.NAMESPACE}-?/, '').replaceAll("/", '')
    echo "Starting Pipeline for ${APP_NAME}..."
    def projectBase = "${env.NAMESPACE}".replaceAll(/-dev/, '')
    env.STAGE0 = "${projectBase}-dev"
    env.STAGE1 = "${projectBase}-build"
    env.STAGE2 = "${projectBase}-prod"
}
pipeline {
    agent {
        node {
            // spin up a slave pod to run this build on
            label 'maven'
        }
    }
    options {
        // set a timeout of 20 minutes for this pipeline
        timeout(time: 20, unit: 'MINUTES')
    }
    stages {
        stage('Git Checkout') {
            steps {
                // Turn off Git's SSL cert check, uncomment if needed
                // sh 'git config --global http.sslVerify false'
                git url: "${SOURCE_URL}"
            }
        }
        // Run Maven build, skipping tests
        stage('Build'){
            steps {
                sh "mvn clean install -DskipTests=true -f ${POM_FILE}"
            }
        }
        // Run Maven unit tests
        stage('Unit Test'){
            steps {
                sh "mvn test -f ${POM_FILE}"
            }
        }
        stage('Build Container Image'){
            steps {
                // Copy the resulting artifacts into common directory
                sh """
                  ls target/*
                  rm -rf oc-build && mkdir -p oc-build/deployments
                  for t in \$(echo "jar;war;ear" | tr ";" "\\n"); do
                  cp -rfv ./target/*.\$t oc-build/deployments/ 2> /dev/null || echo "No \$t files"
                  done
                """
                script {
                    openshift.withCluster() {
                        openshift.withProject("$STAGE0") {
                            echo "Using project: ${openshift.project()}"
                            openshift.selector("bc", "${APP_NAME}").startBuild("--from-dir=oc-build").logs("-f")
                            def builds = openshift.selector("bc", "${APP_NAME}").related('builds')
                            builds.untilEach(1) {
                                return (it.object().status.phase == "Complete")
                            }
                        }
                    }
                } // script
            } // steps
        } // stage
        stage('Promote from Build to Dev') {
            steps {
                script {
                    openshift.withCluster() {
                        openshift.withProject() {
                            openshift.tag("${env.APP_NAME}:latest", "${env.STAGE1}/${env.APP_NAME}:latest")
                        }
                    }
                } // script
            } // steps
        } // stage
        stage ('Verify Deployment to Dev') {
            steps {
                script {
                    openshift.withCluster() {
                        openshift.withProject("${STAGE1}") {
                            def dcObj = openshift.selector("dc", env.APP_NAME).object()
                            def podSelector = openshift.selector('pod', [deployment: "${APP_NAME}-${dcObj.status.latestVersion}"])
                            podSelector.untilEach {
                                echo "pod: ${it.name()}"
                                return it.object().status.containerStatuses[0].ready
                            }
                        }
                    }
                } // script
            } // steps
        } // stage
        stage('Promote from Dev to Prod') {
            steps {
                script {
                    openshift.withCluster() {
                        openshift.withProject() {
                            openshift.tag("${env.APP_NAME}:latest", "${env.STAGE2}/${env.APP_NAME}:latest")
                        }
                    }
                } // script
            } // steps
        } // stage
        stage ('Verify Deployment to Prod') {
            steps {
                script {
                    openshift.withCluster() {
                        openshift.withProject("${STAGE2}") {
                            def dcObj = openshift.selector("dc", env.APP_NAME).object()
                            def podSelector = openshift.selector('pod', [deployment: "${APP_NAME}-${dcObj.status.latestVersion}"])
                            podSelector.untilEach {
                                echo "pod: ${it.name()}"
                                return it.object().status.containerStatuses[0].ready
                            }
                        }
                    }
                } // script
            } // steps
        } // stage
    } // stages
} // pipeline
