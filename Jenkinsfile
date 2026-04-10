import groovy.json.JsonSlurperClassic

// ================= HELPER =================
@NonCPS
def cleanJsonString(String rawOutput) {
    int firstBrace = rawOutput.indexOf('{')
    int lastBrace  = rawOutput.lastIndexOf('}')
    if (firstBrace == -1 || lastBrace == -1) return null
    return rawOutput.substring(firstBrace, lastBrace + 1).trim()
}

// ================= PIPELINE =================
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

        // ================= VERSION CHECK =================
        stage('CHECK VERSION') {
            steps {
                echo "🚀 FINAL PIPELINE VERSION - JSON PARSING FIX"
            }
        }

        // ================= CHECKOUT =================
        stage('Checkout') {
            steps {
                deleteDir()
                git branch: 'main', url: 'https://github.com/nissrinayy/newcity-testing'
            }
        }

        // ================= DEBUG =================
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
                    echo "📤 Uploading APK to MobSF..."

                    def uploadResponse = bat(
                        script: """
                        @curl -s ^
                        -H "Authorization: ${env.MOBSF_TOKEN}" ^
                        -F "file=@${env.APK_PATH}" ^
                        ${env.MOBSF_URL}/api/v1/upload
                        """,
                        returnStdout: true
                    ).trim()

                    echo "RAW RESPONSE: ${uploadResponse}"

                    // ================= FIX: NO JsonSlurper =================
                    def clean = uploadResponse.replaceAll("(?s).*?(\\{.*\\}).*", "\$1")

                    def hashValue = clean.replaceAll(/.*"hash"\s*:\s*"([^"]+)".*/, "\$1")

                    if (hashValue == clean) {
                        error "❌ Upload failed"
                    }

                    env.APK_HASH = hashValue
                    echo "✅ APK HASH: ${env.APK_HASH}"

                    // ================= SCAN =================
                    bat """
                    @curl -s -X POST ^
                    -H "Authorization: ${env.MOBSF_TOKEN}" ^
                    --data "hash=${env.APK_HASH}" ^
                    ${env.MOBSF_URL}/api/v1/scan
                    """

                    // ================= GET REPORT =================
                    def raw = bat(
                        script: """
                        @curl -s -X POST ^
                        -H "Authorization: ${env.MOBSF_TOKEN}" ^
                        --data "hash=${env.APK_HASH}" ^
                        ${env.MOBSF_URL}/api/v1/report_json
                        """,
                        returnStdout: true
                    ).trim()

                    def json = cleanJsonString(raw)

                    if (json) {
                        writeFile file: 'sast_report_newcity.json', text: json
                        archiveArtifacts artifacts: 'sast_report_newcity.json'
                        echo "✅ SAST done"
                    }
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
                    echo "🚀 Starting Dynamic Analysis..."

                    bat """
                    curl -X POST ^
                    -H "Authorization: ${env.MOBSF_TOKEN}" ^
                    --data "hash=${env.APK_HASH}" ^
                    ${env.MOBSF_URL}/api/v1/dynamic/start_analysis
                    """

                    sleep 20

                    bat "adb shell monkey -p ${env.APP_PACKAGE} --throttle 500 -v 300"

                    sleep 20

                    bat """
                    curl -X POST ^
                    -H "Authorization: ${env.MOBSF_TOKEN}" ^
                    --data "hash=${env.APK_HASH}" ^
                    ${env.MOBSF_URL}/api/v1/dynamic/stop_analysis
                    """

                    echo "✅ DAST completed"
                }
            }
        }

        // ================= CLEANUP =================
        stage('Cleanup') {
            steps {
                bat 'taskkill /F /IM emulator.exe /T || echo emulator stopped'
                bat 'adb kill-server || echo adb stopped'
                echo "🧹 Cleanup done"
            }
        }
    }
}