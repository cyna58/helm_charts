pipeline {
    agent any

    environment {
        DOCKER_HUB_USERNAME = credentials('docker-hub-username')
        DOCKER_HUB_PASSWORD = credentials('docker-hub-password')
        KUBECONFIG = credentials('kubeconfig')
    }

    triggers {
        pollSCM('H/1 * * * *') // sprawdzanie zmian co 1 minutę
    }

    stages {
        stage('Checkout Repos') {
            steps {
                // Checkout aplikacji
                git url: 'https://github.com/mcyliowski/kubernetes-dockery.git', branch: 'main'

                // Checkout Helm charts
                dir('helm') {
                    git url: 'https://github.com/mcyliowski/helm_charts.git', branch: 'main'
                }
            }
        }

        stage('Build & Push Docker Images') {
            steps {
                script {
                    // Frontend
                    if (fileExists('frontend/Dockerfile')) {
                        env.FRONTEND_IMAGE = "docker.io/cyna58/nauka-kubernetes:frontend-${env.BUILD_NUMBER}"
                        echo "Building Frontend image: ${env.FRONTEND_IMAGE}"
                        sh """
                            /kaniko/executor \\
                                --dockerfile frontend/Dockerfile \\
                                --context frontend/ \\
                                --destination ${env.FRONTEND_IMAGE} \\
                                --skip-tls-verify \\
                                --oci-layout-path /kaniko/oci || exit 1
                        """
                    }

                    // Backend
                    if (fileExists('backend/Dockerfile')) {
                        env.BACKEND_IMAGE = "docker.io/cyna58/nauka-kubernetes:backend-${env.BUILD_NUMBER}"
                        echo "Building Backend image: ${env.BACKEND_IMAGE}"
                        sh """
                            /kaniko/executor \\
                                --dockerfile backend/Dockerfile \\
                                --context backend/ \\
                                --destination ${env.BACKEND_IMAGE} \\
                                --skip-tls-verify \\
                                --oci-layout-path /kaniko/oci || exit 1
                        """
                    }
                }
            }
        }

        stage('Deploy All Helm Charts') {
            steps {
                script {
                    // Szukamy wszystkich Chart.yaml w katalogu helm/charts/*
                    def charts = findFiles(glob: 'helm/charts/*/Chart.yaml')

                    for (chart in charts) {
                        def chartDir = chart.path.replace('/Chart.yaml','')
                        def releaseName = chartDir.tokenize('/').last()   // np. "poe_wheel_of_choice"
                        
                        // Pobieramy namespace z values.yaml jeśli istnieje, inaczej default
                        def namespace = "default"
                        def valuesFile = "${chartDir}/values.yaml"
                        if (fileExists(valuesFile)) {
                            def valuesYaml = readYaml file: valuesFile
                            if (valuesYaml?.service?.namespace) {
                                namespace = valuesYaml.service.namespace
                            }
                        }

                        echo "Deploying chart: ${releaseName} in namespace: ${namespace}"

                        dir(chartDir) {
                            // Budujemy parametr --set dla obrazów tylko jeśli katalog istnieje
                            def setValues = ""
                            if (fileExists("../../frontend/Dockerfile")) {
                                setValues += "frontend.image=${env.FRONTEND_IMAGE},"
                            }
                            if (fileExists("../../backend/Dockerfile")) {
                                setValues += "backend.image=${env.BACKEND_IMAGE},"
                            }
                            if (setValues.endsWith(",")) setValues = setValues[0..-2]  // usuwamy ostatni przecinek

                            sh """
                                # Tworzymy namespace jeśli nie istnieje
                                kubectl get ns ${namespace} || kubectl create ns ${namespace}

                                # Deploy Helm
                                helm upgrade --install ${releaseName} . \\
                                    --namespace ${namespace} \\
                                    --values values.yaml \\
                                    ${setValues ? "--set ${setValues}" : ""} \\
                                    --kubeconfig ${KUBECONFIG}
                            """
                        }
                    }
                }
            }
        }
    }

    post {
        success {
            echo "✅ All subcharts deployed successfully!"
        }
        failure {
            echo "❌ Deployment failed!"
        }
    }
}
