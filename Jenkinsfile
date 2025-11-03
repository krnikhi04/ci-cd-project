pipeline {
    agent any

    // These variables will be used in the stages
    environment {
        // ---!!! YOU MUST EDIT THESE VALUES !!!---
        GCP_PROJECT_ID    = "tough-shelter-477103-k3"       // Find this on your GCP dashboard
        GCP_REGION        = "us-central1"               // The region of your Artifact Registry (e.g., us-central1)
        AR_REPO_NAME      = "my-web-app"                // The Artifact Registry repo name
        TARGET_VM_IP      = "34.72.97.216"  // The 'External IP' of your 'target-server' VM
        TARGET_VM_USER    = "ubuntu"                    // The user for the Ubuntu VM
        // ---!!! -----------------------------!!!---
        
        // These are calculated automatically
        IMAGE_NAME_BASE   = "${GCP_REGION}-docker.pkg.dev/${GCP_PROJECT_ID}/${AR_REPO_NAME}/my-web-app"
        IMAGE_NAME_BUILD  = "${IMAGE_NAME_BASE}:${env.BUILD_NUMBER}"
        IMAGE_NAME_LATEST = "${IMAGE_NAME_BASE}:latest"
    }

    stages {
        stage('1. Checkout Code') {
            steps {
                echo 'Checking out code from GitHub...'
                checkout scm
            }
        }

        stage('2. Build Application') {
            steps {
                echo 'Installing Node.js dependencies...'
                sh 'npm install'
            }
        }

        stage('3. Run Tests') {
            steps {
                echo 'Running tests...'
                sh 'npm test'
            }
        }

        stage('4. Build Docker Image') {
            steps {
                echo "Building Docker image: ${IMAGE_NAME_BUILD}"
                sh "docker build -t ${IMAGE_NAME_BUILD} ."
                sh "docker tag ${IMAGE_NAME_BUILD} ${IMAGE_NAME_LATEST}"
            }
        }

        stage('5. Push to Artifact Registry') {
            // 'jenkins-gcp-key' is the Credential ID we will create in Jenkins
            withCredentials([file(credentialsId: 'jenkins-gcp-key', variable: 'GCP_KEY_FILE')]) {
                steps {
                    echo 'Authenticating to GCP...'
                    // 1. Authenticate gcloud CLI using the downloaded JSON key
                    sh "gcloud auth activate-service-account --key-file=${GCP_KEY_FILE}"
                    
                    // 2. Configure Docker to use gcloud credentials for the specific region
                    sh "gcloud auth configure-docker ${GCP_REGION}-docker.pkg.dev"

                    echo "Pushing image ${IMAGE_NAME_BUILD}..."
                    sh "docker push ${IMAGE_NAME_BUILD}"
                    
                    echo "Pushing image ${IMAGE_NAME_LATEST}..."
                    sh "docker push ${IMAGE_NAME_LATEST}"
                }
            }
        }

        stage('6. Deploy to Compute Engine') {
            steps {
                echo "Deploying application to ${TARGET_VM_IP}..."
                // 'target-gcp-ssh' is the Credential ID we will create in Jenkins
                sshagent(credentials: ['target-gcp-ssh']) {
                    sh """
                        ssh -o StrictHostKeyChecking=no ${TARGET_VM_USER}@${TARGET_VM_IP} '
                        
                            # The target-server uses its attached service account (target-sa) to auth
                            # We just need to configure Docker to use it
                            gcloud auth configure-docker ${GCP_REGION}-docker.pkg.dev -q
                            
                            # Stop and remove the old container, if it exists
                            docker stop my-web-app || true
                            docker rm my-web-app || true
                            
                            # Pull the new 'latest' image from Artifact Registry
                            docker pull ${IMAGE_NAME_LATEST}
                            
                            # Run the new container, mapping port 80 (public) to 8080 (app)
                            docker run -d -p 80:8080 --name my-web-app ${IMAGE_NAME_LATEST}
                        '
                    """
                }
            }
        }
    }
    
    post {
        always {
            echo 'Cleaning up the Jenkins workspace...'
            // Removes the built docker images from the Jenkins server to save space
            sh "docker rmi ${IMAGE_NAME_BUILD} || true"
            sh "docker rmi ${IMAGE_NAME_LATEST} || true"
        }
        success {
            echo "Pipeline Succeeded! App is live at http://${TARGET_VM_IP}"
        }
        failure {
            echo 'Pipeline Failed.'
        }
    }
}