// CI/CD for travellerhub (krishnaacharyaa/wanderlust app, ESM + TypeScript).
// checkout -> (optional) test -> SonarQube -> OWASP -> Trivy FS -> build+push images
//          -> bump tags in kubernetes/*.yaml -> push -> ArgoCD auto-syncs to EKS.
// Runs on the Jenkins worker agent (Docker + Trivy live there).
//
// >>> VERIFY: credential IDs (dockerHubCreds, github, sonar-token), tool names
//     (sonar-scanner, OWASP-DepCheck) and SonarQube server name ('sonar') match Jenkins config.

pipeline {
    agent { label 'worker-node' }

    environment {
        DOCKERHUB_USER  = 'sahilsahu6246'                          // <-- your Docker Hub username
        BACKEND_IMAGE   = "${DOCKERHUB_USER}/wanderlust-backend"
        FRONTEND_IMAGE  = "${DOCKERHUB_USER}/wanderlust-frontend"
        IMAGE_TAG       = "${BUILD_NUMBER}"
        // Browser-facing backend URL (backend NodePort). Baked into the frontend bundle.
        VITE_API_PATH   = 'http://65.0.185.152:31100'
        // Default OFF: the repo's backend "test" script uses `jest --watchAll`, which never
        // exits and would hang the pipeline. Set to 'true' only with the CI command below.
        RUN_TESTS       = 'false'
    }

    stages {

        stage('Checkout') {
            steps {
                git branch: 'devops-end-to-end',
                    url: 'https://github.com/sahilsahu246/travellerhub.git'
            }
        }

        stage('Install & Test') {
            when { expression { return env.RUN_TESTS == 'true' } }
            steps {
                // No committed lockfile -> npm install. Frontend needs --ignore-scripts to skip
                // its monorepo "prepare" hook. Backend test is forced into non-watch CI mode.
                dir('backend')  { sh 'npm install && npx jest --ci --runInBand --detectOpenHandles --coverage --passWithNoTests' }
                dir('frontend') { sh 'npm install --ignore-scripts && npm test' }
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonar') {
                    script {
                        def scannerHome = tool 'sonar-scanner'
                        sh "${scannerHome}/bin/sonar-scanner -Dsonar.projectKey=travellerhub -Dsonar.projectName=travellerhub -Dsonar.sources=backend,frontend"
                    }
                }
            }
        }

        stage('OWASP Dependency-Check') {
            steps {
                dependencyCheck additionalArguments: '--scan ./ --format HTML --format XML',
                                odcInstallation: 'OWASP-DepCheck'
                dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
            }
        }

        stage('Trivy Filesystem Scan') {
            steps {
                sh 'trivy fs --severity HIGH,CRITICAL --no-progress .'
            }
        }

        stage('Build Images') {
            steps {
                sh "docker build -t ${BACKEND_IMAGE}:${IMAGE_TAG} ./backend"
                sh "docker build --build-arg VITE_API_PATH=${VITE_API_PATH} -t ${FRONTEND_IMAGE}:${IMAGE_TAG} ./frontend"
            }
        }

        stage('Push Images') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'dockerHubCreds',
                                                  usernameVariable: 'DH_USER',
                                                  passwordVariable: 'DH_PASS')]) {
                    sh 'echo "$DH_PASS" | docker login -u "$DH_USER" --password-stdin'
                    sh "docker push ${BACKEND_IMAGE}:${IMAGE_TAG}"
                    sh "docker push ${FRONTEND_IMAGE}:${IMAGE_TAG}"
                }
            }
        }

        stage('Update Manifests (GitOps)') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'github',
                                                  usernameVariable: 'GH_USER',
                                                  passwordVariable: 'GH_TOKEN')]) {
                    sh '''
                        sed -i "s#${BACKEND_IMAGE}:.*#${BACKEND_IMAGE}:${IMAGE_TAG}#"   kubernetes/backend.yaml
                        sed -i "s#${FRONTEND_IMAGE}:.*#${FRONTEND_IMAGE}:${IMAGE_TAG}#" kubernetes/frontend.yaml

                        git config user.email "ci@travellerhub.local"
                        git config user.name  "jenkins-ci"
                        git add kubernetes/backend.yaml kubernetes/frontend.yaml
                        git commit -m "ci: bump images to ${IMAGE_TAG} [skip ci]" || echo "nothing to commit"
                        git push https://${GH_USER}:${GH_TOKEN}@github.com/sahilsahu246/travellerhub.git HEAD:devops-end-to-end
                    '''
                }
            }
        }
    }

    post {
        always {
            sh 'docker logout || true'
            cleanWs()
        }
    }
}