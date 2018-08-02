openshift.withCluster() {
    // path of the template to use
    def templatePath = 'https://raw.githubusercontent.com/hashnao/openshift-tasks/master/openshift-tasks-build.yaml'
    // name of the template that will be created
    def templateName = 'openshift-tasks'
    // NOTE, the "pipeline" directive/closure from the declarative pipeline syntax needs to include, or be nested outside,
    // and "openshift" directive/closure from the OpenShift Client Plugin for Jenkins.  Otherwise, the declarative pipeline engine
    // will not be fully engaged.
    env.NAMESPACE = openshift.project()
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
        stage('preamble') {
            steps {
                script {
                    openshift.withCluster() {
                        openshift.withProject("$STAGE0") {
                            echo "Using project: ${openshift.project()}"
                        }
                    }
                }
            }
        }
        stage('cleanup') {
            steps {
                script {
                    openshift.withCluster() {
                        openshift.withProject() {
                            // delete everything with this template label
                            openshift.selector("all", [ template : templateName ]).delete()
                            // delete any secrets with this template label
                            if (openshift.selector("secrets", templateName).exists()) {
                                openshift.selector("secrets", templateName).delete()
                            }
                        }
                    }
                } // script
            } // steps
        } // stage
       stage('build') {
            steps {
                script {
                    openshift.withCluster() {
                        openshift.withProject() {
                            // output the build logs to the Jenkins console. logs()
                            // would run `oc logs bc/ruby-hello-world`, but that might only
                            // output a partial log if the build is in progress. Instead, we will
                            // pass '-f' to `oc logs` to follow the build until it terminates.
                            // Arguments to logs get passed Directly on to the oc command line.
                            def result = openshift.selector("bc", templateName).logs('-f')
                            // You can even see exactly what oc command was executed.
                            echo "Logs executed: ${result.actions[0].cmd}"
                            //
                            def builds = openshift.selector("bc", templateName).related('builds')
                            builds.untilEach(1) {
                                return (it.object().status.phase == "Complete")
                            }
                        }
                    }
                } // script
            } // steps
        } // stage
        stage('tag to dev') {
            steps {
                script {
                    openshift.withCluster() {
                        openshift.withProject() {
                            // if everything else succeeded, tag the ${templateName}:latest image as ${templateName}-prod:latest
                            // a pipeline build config for the development environment can watch for the ${templateName}-prod:latest
                            // image to change and then deploy it to development the environment
                            openshift.tag("${templateName}:latest", "${env.STAGE1}/${templateName}:latest")
                        }
                    }
                } // script
            } // steps
        } // stage
        stage('deploy to dev') {
            steps {
                script {
                    openshift.withCluster() {
                        openshift.withProject("${STAGE1}") {
                            def deploy = openshift.selector("dc", templateName)
                            deploy.rollout().latest()
                            deploy.rollout().status('-w')
                            deploy.related('pods').untilEach(1) {
                                return (it.object().status.phase == "Running")
                            }
                        }
                    }
                } // script
            } // steps
        } // stage
        stage('tag to prod') {
            steps {
                script {
                    openshift.withCluster() {
                        openshift.withProject() {
                            openshift.tag("${templateName}:latest", "${env.STAGE2}/${templateName}:latest")
                        }
                    }
                } // script
            } // steps
        } // stage
        stage('deploy to prod') {
            steps {
                script {
                    openshift.withCluster() {
                        openshift.withProject("${STAGE2}") {
                            def deploy = openshift.selector("dc", templateName)
                            deploy.rollout().latest()
                            deploy.rollout().status('-w')
                            deploy.related('pods').untilEach(1) {
                                return (it.object().status.phase == "Running")
                            }
                        }
                    }
                } // script
            } // steps
        } // stage
    } // stages
} // pipeline
