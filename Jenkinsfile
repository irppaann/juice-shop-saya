pipeline {
    agent any

    environment {
        // Definisikan variabel agar mudah diatur di satu tempat
        SCANNER_IMAGE = 'sonarsource/sonar-scanner-cli'
        SONAR_HOST    = 'http://172.16.15.29:2020'
        SONAR_TOKEN   = 'sqa_fe9b3301c18595d96409b2b7311a3a4bd65e8be3'
        PROJECT_KEY   = 'juice-shop-saya'
        // Memaksa path agar konsisten dan menghindari folder @2 / @tmp
        WS_PATH       = "/var/jenkins_home/workspace/${env.JOB_NAME}"
    }
	
	triggers {
        // Opsi 1: Jenkins cek ke GitHub setiap menit (paling mudah untuk lokal)
        pollSCM('* * * * *') 
    }

    stages {

        stage('Checkout Code') {
            steps {
                ws("${WS_PATH}") {
                    echo "Pulling code from GitHub..."
                    checkout([$class: 'GitSCM', 
                        branches: [[name: '*/master']], 
                        doGenerateSubmoduleConfigurations: false, 
                        extensions: [
                            [$class: 'CloneOption', depth: 1, shallow: true, timeout: 10],
                            [$class: 'CleanupBeforeCheckout'] // Bersihkan folder sebelum tarik baru
                        ], 
                        userRemoteConfigs: [[url: 'https://github.com/irppaann/juice-shop-saya.git']]
                    ])
                    sh "ls -la" // Verifikasi file ada (package.json, dll)
                }

                // script {
                //     // Mengganti 'checkout scm' standar dengan detail untuk shallow clone
                //     echo 'START check
                //     out code'
                //     sh 'pwd' //untuk print path workspace yang sedang digunakan
                //     checkout([$class: 'GitSCM', 
                //         branches: [[name: '*/master']], // Pastikan sesuai branch Anda (master/main)
                //         doGenerateSubmoduleConfigurations: false, 
                //         extensions: [
                //             [
                //                 $class: 'CloneOption', 
                //                 depth: 1,          // Hanya mengambil commit terakhir
                //                 noTags: false, 
                //                 reference: '', 
                //                 shallow: true,      // Mengaktifkan mode shallow
                //                 timeout: 30         // set timeout saat clone jadi 30 mnt
                //             ],
                //             // Tambahan untuk mengatasi error RPC/HTTP2 yang Anda alami sebelumnya
                //             [$class: 'CheckoutOption', 
                //                 timeout: 30]  // set timeout saat checkout jadi 30 mnt
                //         ], 
                //         submoduleCfg: [], 
                //         userRemoteConfigs: [[url: 'https://github.com/irppaann/juice-shop-saya.git']]
                //     ])
                //     echo 'END:SUCCESS Checkout code berhasil dijalankan'
                // }
            }
        }    

        stage('SAST Analysis (SonarQube)') {
            steps {
                ws("${WS_PATH}") {
                    script {
                        echo 'Starting SonarQube Scan...'
                        // Menggunakan docker run secara langsung agar lebih fleksibel
                        // Kita mount ${WS_PATH} ke /usr/src milik container scanner
                        sh """
                        docker run --rm --network=host \
                            -v "${WS_PATH}:/usr/src" \
                            -u 0:0 \
                            ${SCANNER_IMAGE} \
                            -Dsonar.projectKey=${PROJECT_KEY} \
                            -Dsonar.sources=. \
                            -Dsonar.host.url=${SONAR_HOST} \
                            -Dsonar.token=${SONAR_TOKEN} \
                            -Dsonar.exclusions=node_modules/**,test/**,spec/** \
                            -Dsonar.loglevel=INFO
                        """
                    }
                }

                // script {
                //     sh """
                //     docker run --rm --network=host \
                //     -v "${WORKSPACE}:/usr/src" \
                //     sonarsource/sonar-scanner-cli \
                //     -Dsonar.projectKey=juice-shop-saya \
                //     -Dsonar.sources=. \
                //     -Dsonar.host.url=http://172.16.15.29:2020 \
                //     -Dsonar.token=sqa_fe9b3301c18595d96409b2b7311a3a4bd65e8be3 \
                //     -Dsonar.exclusions=node_modules/**,test/**,spec/** \
                //     -Dsonar.loglevel=DEBUG
                //     """
                // }

                // withSonarQubeEnv('SonarQube-Lokal') {
                //     script{
                //         echo 'START SAST Analysis di Workspace: ${env.WORKSPACE}'
                //         docker.image('sonarsource/sonar-scanner-cli').inside('--network=host -u 0:0') {
                //             sh """
                //             sonar-scanner -X \
                //                 -Dsonar.projectKey=juice-shop-saya \
                //                 -Dsonar.sources=. \
                //                 -Dsonar.javascript.lcov.reportPaths=coverage/lcov.info \
                //                 -Dsonar.host.url=http://172.16.15.29:2020 \
                //                 -Dsonar.token=sqa_fe9b3301c18595d96409b2b7311a3a4bd65e8be3 \
                //                 -Dsonar.exclusions=node_modules/**,test/**,spec/**
                //             """
                //         }
                //         echo 'END:SUCCESS SAST Analysis berhasil dijalankan'
                //     }
                // }

                // script {
                //     echo 'START SAST Analysis di Workspace'
                //     // Kita gunakan image resmi sonar-scanner-cli agar Jenkins tetap bersih
                //     docker.image('sonarsource/sonar-scanner-cli').inside('--network=host -u 0:0') {
                //         sh """
                //         sonar-scanner \
                //           -Dsonar.projectKey=juice-shop-saya \
                //           -Dsonar.sources=. \
                //           -Dsonar.host.url=http://172.16.15.29:2020 \
                //           -Dsonar.token=sqa_fe9b3301c18595d96409b2b7311a3a4bd65e8be3 \
                //           -Dsonar.javascript.lcov.reportPaths=coverage/lcov.info \
                //           -Dsonar.exclusions=node_modules/**,test/**,spec/**
                //         """
                //     }
                // }
            }
        }

        stage('Quality Gate') {
            steps {
                // Gunakan timeout agar pipeline tidak 'gantung' selamanya jika Webhook gagal
                timeout(time: 5, unit: 'MINUTES') {
                    script {
                        // Perhatikan: withSonarQubeEnv harus sesuai dengan nama di Jenkins System Config
                        withSonarQubeEnv('SonarQube-Lokal') {
                            def qg = waitForQualityGate()
                            if (qg.status != 'OK') {
                                error "Pipeline failed due to Quality Gate: ${qg.status}"
                            }
                        }
                    }
                }

                // timeout(time: 5, unit: 'MINUTES') {
                //     script {
                //         echo 'START Quality Gate'
                //         // Jenkins akan menunggu respon dari SonarQube apakah project ini "Passed" atau "Failed"
                //         def qg = waitForQualityGate()
                //         if (qg.status != 'OK') {
                //             error "Pipeline berhenti karena Quality Gate SonarQube gagal: ${qg.status}"
                //         }
                //         echo 'END:SUCCESS Quality Gate berhasil dijalankan'
                //     }
                // }
            }
        }

        stage('Build & Deploy (Docker)') {
            steps {
                echo 'START Build image & deploy container'
                sh 'docker build -t juice-shop-app .'
                sh 'docker rm -f juice-shop-container || true'
                sh 'docker run -d --name juice-shop-container -p 3000:3000 juice-shop-app'
                echo 'END:SUCCESS Build & Deploy berhasil dijalankan'
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