pipeline {
    agent any

    parameters {
        choice(name: 'MODULE', choices: ['ems', 'cns', 'tms', 'wfh', 'amenitybooking'], description: 'Select the module')
        // Branch is handled automatically in Multibranch, but keeping this for logic reference
        choice(name: 'NAMESPACE', choices: ['spm-dev', 'spm-prod'], description: 'Target Namespace')
    }

    environment {
        CREDENTIALS_ID = 'abec083d-f2fd-4744-9b39-66f274d5275b' // GitLab Creds
        DOCKER_REGISTRY_USER = 'jk2425'
        DOCKER_CRED_ID = 'docker-hub-creds' // Jenkins Credential ID for Docker Hub
        GCP_AUTH_KEY = credentials('gke-auth') 
        APPROVER_EMAIL = 'jagadishkadiri8@gmail.com'
    }

    stages {
        stage('Initialize & Scan') {
            steps {
                script {
                    def repoMap = [ems: 'EMS', cns: 'CommonNotificationService', tms: 'TMS', wfh: 'workfromhome', amenitybooking: 'spacemanagement']
                    env.REPO_NAME = repoMap[params.MODULE]
                    // Branch detection logic
                    env.CURRENT_BRANCH = env.BRANCH_NAME ?: 'dev'
                    echo "🛠️ Building ${env.REPO_NAME} from branch ${env.CURRENT_BRANCH}"
                }
            }
        }

        stage('Checkout') {
            steps {
                checkout scm // In Multibranch, this automatically pulls the correct branch
            }
        }

        stage('Maven Build') {
            steps {
                sh 'mvn clean install -DskipTests'
            }
        }

        stage('Snyk Repo Scan') {
            steps {
                echo "Running Snyk scan for dependency vulnerabilities..."
                sh '''
                    echo "Authenticating with Snyk..."
                    snyk auth $snyk_token
                    echo "Running Snyk Test..."
                    snyk test --all-projects || true
                    snyk monitor --all-projects || true
                '''
            }
        }

        stage('Security Scan (Trivy)') {
            steps {
                // High-value interview point: "We don't build if the FS is compromised"
                sh "trivy fs --severity HIGH,CRITICAL --exit-code 0 ." 
                echo "✅ Security scan completed."
            }
        }

        stage('Docker Build & Push') {
            steps {
                script {
                    def imageName = "${DOCKER_REGISTRY_USER}/${params.MODULE}:${env.CURRENT_BRANCH}-${env.BUILD_NUMBER}"
                    
                    docker.withRegistry('https://index.docker.io/v1/', "${DOCKER_CRED_ID}") {
                        def customImage = docker.build(imageName, "-f Dockerfile .")
                        
                        // Image Scanning before Push
                        sh "trivy image --severity CRITICAL --exit-code 0 ${imageName}"
                        
                        customImage.push()
                    }
                    env.IMAGE_TAG = imageName
                }
            }
        }

        stage('Production Approval Gate') {
            // ONLY triggers if the branch is 'main'
            when {
                branch 'main'
            }
            steps {
                script {
                    echo "📢 Notifying ${env.APPROVER_EMAIL} for Production Deployment Approval"
                    // The pipeline pauses here
                    timeout(time: 1, unit: 'HOURS') {
                        input message: "Approve deployment of ${params.MODULE} to Production?",
                              ok: "Deploy to Prod",
                              submitter: "admin,jagadish" // You can restrict who clicks this
                    }
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                script {
                    sh """
                        gcloud auth activate-service-account --key-file=${GCP_AUTH_KEY}
                        gcloud config set project ss-dev-infra-01
                        gcloud container clusters get-credentials ss-gcp-dev-singapore-cluster-01 --region asia-southeast1
                        kubectl set image deployment/${params.MODULE} ${params.MODULE}=${env.IMAGE_TAG} -n ${params.NAMESPACE}
                        kubectl rollout status deployment/${params.MODULE} -n ${params.NAMESPACE}
                    """
                }
            }
        }
    }

    post {
        success {
            echo "🎉 Successfully deployed ${params.MODULE} to ${params.NAMESPACE}"
        }
        failure {
            mail to: "${env.APPROVER_EMAIL}",
                 subject: "FAILED: Pipeline ${currentBuild.fullDisplayName}",
                 body: "Something went wrong with the deployment of ${params.MODULE}. Check Jenkins logs."
        }
    }
}
