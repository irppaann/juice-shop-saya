pipeline {
    agent any

    environment {
        // Nama kredensial yang Anda simpan di Jenkins (Manage Jenkins > Credentials)
        SONAR_TOKEN = credentials('sonar-token')
        // Sesuaikan dengan IP laptop Anda jika SonarQube jalan di Docker lokal
        SONAR_HOST_URL = 'http://172.16.15.35:2020' 
        SONAR_PROJECT_KEY = 'juice-shop-saya'
    }
	
	triggers {
        // Opsi 1: Jenkins cek ke GitHub setiap menit (paling mudah untuk lokal)
        pollSCM('* * * * *') 
    }

    stages {

        stage('SAST Analysis (SonarQube)') {
            steps {
                script {
                    // Kita gunakan image resmi sonar-scanner-cli agar Jenkins tetap bersih
                    docker.image('sonarsource/sonar-scanner-cli').inside('--network=host') {
                        sh """
                        sonar-scanner \
                          -Dsonar.projectKey=${SONAR_PROJECT_KEY} \
                          -Dsonar.sources=. \
                          -Dsonar.host.url=${SONAR_HOST_URL} \
                          -Dsonar.token=${SONAR_TOKEN} \
                          -Dsonar.javascript.lcov.reportPaths=coverage/lcov.info \
                          -Dsonar.exclusions=node_modules/**,test/**,spec/**
                        """
                    }
                }
            }
        }

        stage('Quality Gate') {
            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    script {
                        // Jenkins akan menunggu respon dari SonarQube apakah project ini "Passed" atau "Failed"
                        def qg = waitForQualityGate()
                        if (qg.status != 'OK') {
                            error "Pipeline berhenti karena Quality Gate SonarQube gagal: ${qg.status}"
                        }
                    }
                }
            }
        }

        stage('Build & Deploy (Docker)') {
            steps {
                echo 'Hanya berjalan jika Quality Gate lulus...'
                sh 'docker build -t juice-shop-app .'
                sh 'docker rm -f juice-shop-container || true'
                sh 'docker run -d --name juice-shop-container -p 3000:3000 juice-shop-app'
            }
        }
    }

    post {
        always {
            echo 'Membersihkan workspace...'
            cleanWs()
        }
    }
}