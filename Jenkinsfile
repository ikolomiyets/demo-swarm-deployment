version="1.0.0"
default_tag="latest"
def result

podTemplate(label: 'demo-test-deployment-pod', cloud: 'kubernetes', serviceAccount: 'jenkins',
  containers: [
    containerTemplate(name: 'docker', image: 'docker:dind', ttyEnabled: true, command: 'cat', privileged: true,
        envVars: [secretEnvVar(key: 'DOCKER_USERNAME', secretName: 'ikolomiyets-docker-hub-credentials', secretKey: 'username'),
        envVar(key: 'DOCKER_HOST', value: 'tcp://sonarqube.iktech.io:2376'),
        envVar(key: 'DOCKER_TLS_VERIFY', value: '1'),
    ])
  ],
  volumes: [
    secretVolume(mountPath: '/etc/.ssh', secretName: 'ssh-home'),
    secretVolume(mountPath: '/etc/docker/certificates', secretName: 'sonarqube-docker-certificates'),
    secretVolume(secretName: 'ikolomiyets-docker-hub-credentials', mountPath: '/etc/.secret'),
    hostPathVolume(hostPath: '/var/run/docker.sock', mountPath: '/var/run/docker.sock')
  ]) {
    node('demo-test-deployment-pod') {
        stage('Prepare') {
            checkout scm

            // Set up private key to access BitBucket
            sh """
                cat /etc/.ssh/id_rsa > ~/.ssh/id_rsa
                chmod 400 ~/.ssh/id_rsa
                mkdir -p ~/.docker
                ls -l /etc/docker/certificates/
                cp /etc/docker/certificates/ca.pem ~/.docker/ca.pem
                cp /etc/docker/certificates/cert.pem ~/.docker/cert.pem
                cp /etc/docker/certificates/key.pem ~/.docker/key.pem
                chmod 600 ~/.docker/key.pem
            """
        }

        stage('Retrieve docker images versions') {
			result = retrieveArtifacts stage: 'Production', names: ['demo-customer', 'demo-policy', 'demo-frontend']
        }

        stage('Deploy Latest') {
            container('docker') {
                demo_customer_version = result['demo_customer']
                demo_policy_version = result['demo_policy']
                demo_frontend_version = result['demo_frontend']

                if (demo_customer_version.isEmpty()) {
                    demo_customer_version = 'latest';
                }

                if (demo_policy_version.isEmpty()) {
                    demo_policy_version = 'latest';
                }

                if (demo_frontend_version.isEmpty()) {
                    demo_frontend_version = 'latest';
                }

                sh """
                    echo "Preparing docker-compose file"
                    echo "Deploying stack to a docker instance at ${env.DOCKER_HOST}"
                    cat docker-compose.template| sed s/__DEMO_POLICY_TAG__/${demo_policy_version}/g|sed s/__DEMO_CUSTOMERS_TAG__/${demo_customer_version}/g|sed s/__DEMO_FRONTEND_TAG__/${demo_frontend_version}/g|docker stack deploy --compose-file docker-compose.prod.yaml --compose-file - demo-prod
                """
            }
        }
    }
}

properties([[
    $class: 'BuildDiscarderProperty',
    strategy: [
        $class: 'LogRotator',
        artifactDaysToKeepStr: '', artifactNumToKeepStr: '', daysToKeepStr: '', numToKeepStr: '10']
    ]
]);
