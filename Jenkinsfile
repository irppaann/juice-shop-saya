pipeline {
    agent any

    environment {
        // Nama kredensial yang Anda simpan di Jenkins (Manage Jenkins > Credentials)
        SONAR_TOKEN = credentials('sonar-token')
        // Sesuaikan dengan IP laptop Anda jika SonarQube jalan di Docker lokal
        SONAR_HOST_URL = 'http://172.16.15.29:2020' 
        SONAR_PROJECT_KEY = 'juice-shop-saya'
    }
	
	triggers {
        // Opsi 1: Jenkins cek ke GitHub setiap menit (paling mudah untuk lokal)
        pollSCM('* * * * *') 
    }

    stages {

        stage('Checkout Code') {
            steps {
                script {
                    // Mengganti 'checkout scm' standar dengan detail untuk shallow clone
                    echo 'Menjalankan checkout code'
                    checkout([$class: 'GitSCM', 
                        branches: [[name: '*/master']], // Pastikan sesuai branch Anda (master/main)
                        doGenerateSubmoduleConfigurations: false, 
                        extensions: [
                            [
                                $class: 'CloneOption', 
                                depth: 1,          // Hanya mengambil commit terakhir
                                noTags: false, 
                                reference: '', 
                                shallow: true,      // Mengaktifkan mode shallow
                                timeout: 30         // set timeout saat clone jadi 30 mnt
                            ],
                            // Tambahan untuk mengatasi error RPC/HTTP2 yang Anda alami sebelumnya
                            [$class: 'CheckoutOption', 
                                timeout: 30]  // set timeout saat checkout jadi 30 mnt
                        ], 
                        submoduleCfg: [], 
                        userRemoteConfigs: [[url: 'https://github.com/irppaann/juice-shop-saya.git']]
                    ])
                    echo 'Checkout code berhasil dijalankan'
                }
            }
        }    

        stage('SAST Analysis (SonarQube)') {
            steps {
                withSonarQubeEnv('SonarQube-Lokal') {
                    script{
                        docker.image('sonarsource/sonar-scanner-cli').inside('--network=host') {
                            sh """
                            sonar-scanner -X \
                                -Dsonar.projectKey=juice-shop-project \
                                -Dsonar.sources=. \
                                -Dsonar.javascript.lcov.reportPaths=coverage/lcov.info \
                                -Dsonar.host.url=${SONAR_HOST_URL} \
                                -Dsonar.token=sqp_1f2ca5d4d98dfeaac2094a18fb6b89721bc313c7 \
                                -Dsonar.exclusions=node_modules/**,test/**,spec/**
                            """
                        }
                    }
                }
                // script {
                //     // Kita gunakan image resmi sonar-scanner-cli agar Jenkins tetap bersih
                //     docker.image('sonarsource/sonar-scanner-cli').inside('--network=host') {
                //         sh """
                //         sonar-scanner \
                //           -Dsonar.projectKey=${SONAR_PROJECT_KEY} \
                //           -Dsonar.sources=. \
                //           -Dsonar.host.url=${SONAR_HOST_URL} \
                //           -Dsonar.token=${SONAR_TOKEN} \
                //           -Dsonar.javascript.lcov.reportPaths=coverage/lcov.info \
                //           -Dsonar.exclusions=node_modules/**,test/**,spec/**
                //         """
                //     }
                // }
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