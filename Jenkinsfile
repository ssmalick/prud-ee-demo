/* Requires the Docker Pipeline plugin */
pipeline {
    // Use docker agent and `ansible-builder-ee` container image built previously
    agent {
        docker { 
            image 'ansible-builder-ee'
            args '--privileged --user 0'
        } 
    }

    stages {
        stage('Login to RH Registry') {
            steps {
                sh 'podman login -u RH_REG_USER -p RH_REG_PASSWORD registry.redhat.io'
            }
        }

        stage('Pull 24/ee-minimal') {
            steps {
                sh 'podman pull registry.redhat.io/ansible-automation-platform-24/ee-minimal-rhel8:latest'
            }
        }

        stage('Build EE') {
            steps {
                dir('ee/build-definition-v1') {
                    sh 'ansible-builder build -t ansible-builder-ee'
                }
            }
        }

        stage('Push EE to PAH') {
            steps {
                sh 'podman tag localhost/ansible-builder-ee:latest pah.lan/ansible-builder-ee'
                sh 'podman login -u PAH_USER -p PAH_PASSWORD PAH_URL --tls-verify=false'
                sh 'podman push pah.lan/ansible-builder-ee:latest --tls-verify=false'
            }
        }
    }
}


