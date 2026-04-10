import groovy.json.JsonSlurperClassic

pipeline {
    agent any

    environment {
        ANDROID_HOME = "C:\\Users\\Nisrina\\AppData\\Local\\Android\\Sdk"
        JAVA_HOME    = "C:\\Program Files\\Eclipse Adoptium\\jdk-17.0.17.10-hotspot"

        PATH = "${JAVA_HOME}\\bin;${ANDROID_HOME}\\platform-tools;${ANDROID_HOME}\\emulator;${env.PATH}"

        AVD_NAME     = "Pixel_4_XL"

        APK_PATH     = "test-apk/newcity-testing.apk"
        APP_PACKAGE  = "com.dhilla.newcity"

        MOBSF_URL    = "http://localhost:8000"
        MOBSF_TOKEN  = "67f8dcdbaf63751750653685407053c3e1762a3394c5833de1d00379ca06c0fe"
    }

    stages {

        // ================= CHECKOUT =================
        stage('Checkout Code') {
            steps {
                deleteDir()              // clean workspace
                checkout scm             // 🔥 WAJIB
            }
        }

        // ================= DEBUG FILE =================
        stage('Verify Files') {
            steps {
                bat 'echo === WORKSPACE ==='
                bat 'cd'
                bat 'dir /s /b'
            }
        }

        // ================= VALIDATE APK =================
        stage('Validate APK') {
            steps {
                script {
                    if (!fileExists(env.APK_PATH)) {
                        error "❌ APK not found: ${env.APK_PATH}"
                    }
                    echo "✅ APK found!"
                }
            }
        }

        // ================= PREPARE =================
        stage('Prepare Workspace') {
            steps {
                bat 'if not exist apk-outputs mkdir apk-outputs'
            }
        }

        // ================= SAST =================
        stage('SAST - MobSF') {
            steps {
                script {
                    echo "Uploading APK..."

                    def uploadResponse = bat(
                        script: """
                        @curl -s ^
                        -H "Authorization: ${env.MOBSF_TOKEN}" ^
                        -F "file=@${env.APK_PATH}" ^
                        ${env.MOBSF_URL}/api/v1/upload
                        """,
                        returnStdout: true
                    ).trim()

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

        // ================= EMULATOR =================
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
            }
        }

        // ================= INSTALL APK =================
        stage('Install APK') {
            steps {
                script {
                    def timestamp = new Date().format("dd-MM-yyyy_HH-mm-ss")
                    def destPath  = "apk-outputs\\newcity-${timestamp}.apk"

                    bat "copy \"${env.APK_PATH}\" \"${destPath}\""

                    bat(script: "adb uninstall ${env.APP_PACKAGE}", returnStatus: true)
                    bat "adb install -r \"${destPath}\""

                    echo "✅ APK Installed"
                }
            }
        }

        // ================= DAST =================
        stage('DAST') {
            steps {
                script {
                    bat """
                    @curl -s -X POST ^
                    -H "Authorization: ${env.MOBSF_TOKEN}" ^
                    --data "hash=${env.APK_HASH}" ^
                    ${env.MOBSF_URL}/api/v1/dynamic/start_analysis
                    """

                    sleep 30

                    bat "adb shell monkey -p ${env.APP_PACKAGE} -v 500"

                    sleep 30

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
                bat """
                taskkill /F /IM emulator.exe /T || echo emulator stopped
                adb kill-server || echo adb stopped
                """
            }
        }
    }
}