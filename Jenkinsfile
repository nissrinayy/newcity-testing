import groovy.json.JsonSlurperClassic

pipeline {
    agent any

    environment {
        ANDROID_HOME = "C:\\Users\\Nisrina\\AppData\\Local\\Android\\Sdk"
        JAVA_HOME    = "C:\\Program Files\\Eclipse Adoptium\\jdk-17.0.17.10-hotspot"

        PATH = "${JAVA_HOME}\\bin;${ANDROID_HOME}\\platform-tools;${ANDROID_HOME}\\emulator;${env.PATH}"

        AVD_NAME     = "Pixel_4_XL"

        // 🔥 APK dari repo (bukan build)
        APK_PATH     = "test-apk/newcity-testing.apk"
        APP_PACKAGE  = "com.dhilla.newcity"

        MOBSF_URL    = "http://localhost:8000"
        MOBSF_TOKEN  = "67f8dcdbaf63751750653685407053c3e1762a3394c5833de1d00379ca06c0fe"
    }

    stages {
        stage('CHECK VERSION') {
            steps {
                echo "USING NEW PIPELINE VERSION"
            }
        }
        // ================= CHECKOUT =================
        stage('Checkout') {
            steps {
                deleteDir()
                git branch: 'main', url: 'https://github.com/nissrinayy/newcity-testing'
            }
        }

        // ================= DEBUG (WAJIB biar gak halu) =================
        stage('Debug APK Path') {
            steps {
                bat 'echo === SEARCH APK ==='
                bat 'dir /s /b *.apk'
            }
        }

        // ================= VALIDATE =================
        stage('Validate APK') {
            steps {
                script {
                    if (!fileExists(env.APK_PATH)) {
                        error "❌ APK NOT FOUND: ${env.APK_PATH}"
                    }
                    echo "✅ APK found: ${env.APK_PATH}"
                }
            }
        }

        // ================= SAST =================
        stage('SAST - MobSF') {
            steps {
                script {
                    echo "Uploading APK..."

                    def uploadResponse = bat(
                        script: """
                        @echo === CURL RESPONSE ===
                        curl ^
                        -H "Authorization: ${env.MOBSF_TOKEN}" ^
                        -F "file=@${env.APK_PATH}" ^
                        ${env.MOBSF_URL}/api/v1/upload
                        """,
                        returnStdout: true
                    ).trim()

                    echo "UPLOAD RESPONSE: ${uploadResponse}"

                    def matcher = uploadResponse =~ /"hash"\\s*:\\s*"([^"]+)"/
                    if (!matcher.find()) error "❌ Upload failed"

                    env.APK_HASH = matcher.group(1)
                    echo "APK HASH: ${env.APK_HASH}"

                    bat """
                    @curl -s -X POST ^
                    -H "Authorization: ${env.MOBSF_TOKEN}" ^
                    --data "hash=${env.APK_HASH}" ^
                    ${env.MOBSF_URL}/api/v1/scan
                    """
                }
            }
        }

        // ================= START EMULATOR =================
        stage('Start Emulator') {
            steps {
                bat """
                start /b "" "${env.ANDROID_HOME}\\emulator\\emulator.exe" ^
                -avd "${env.AVD_NAME}" ^
                -no-window -no-audio -gpu swiftshader_indirect -wipe-data
                """

                sleep 60
                bat "adb wait-for-device"
                bat "adb shell getprop sys.boot_completed"

                echo "✅ Emulator ready"
            }
        }

        // ================= INSTALL APK =================
        stage('Install APK') {
            steps {
                bat "adb uninstall ${env.APP_PACKAGE} || echo not installed"
                bat "adb install -r \"${env.APK_PATH}\""
                echo "✅ APK installed"
            }
        }

        // ================= DAST =================
        stage('DAST - MobSF') {
            steps {
                script {
                    bat """
                    @curl -s -X POST ^
                    -H "Authorization: ${env.MOBSF_TOKEN}" ^
                    --data "hash=${env.APK_HASH}" ^
                    ${env.MOBSF_URL}/api/v1/dynamic/start_analysis
                    """

                    sleep 20

                    bat "adb shell monkey -p ${env.APP_PACKAGE} --throttle 500 -v 300"

                    sleep 20

                    bat """
                    @curl -s -X POST ^
                    -H "Authorization: ${env.MOBSF_TOKEN}" ^
                    --data "hash=${env.APK_HASH}" ^
                    ${env.MOBSF_URL}/api/v1/dynamic/stop_analysis
                    """
                }
            }
        }

        // ================= CLEANUP =================
        stage('Cleanup') {
            steps {
                bat 'taskkill /F /IM emulator.exe /T || echo emulator stopped'
                bat 'adb kill-server || echo adb stopped'
            }
        }
    }
}