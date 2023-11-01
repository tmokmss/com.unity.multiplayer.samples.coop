pipeline {
    agent none

    stages {
        stage('build Unity project on spot') {
            agent {
                docker {
                    image 'unityci/editor:2021.3.14f1-ios-1.0'
                    args '-u root:root'
                    label 'linux'
                }
            }

            steps {
                // https://plugins.jenkins.io/jobcacher/
                sh 'rm -rf Logs'
                cache(maxCacheSize: 1000, caches: [arbitraryFileCache(path: './Logs', compressionMethod: 'ZIP')]) {
                    sh '''
                    # キャッシュ処理のデモ
                    ls Logs || true
                    '''
                }
                sh '''#!/bin/bash
                set -xe
                printenv
                ls -la
                echo "===Installing stuff for unity"
                apt-get update
                apt-get install -y curl unzip zip jq

                curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
                unzip -q -o awscliv2.zip
                ./aws/install

                # S3にファイルを配置する例
                echo 'File sharing via S3 example' > /tmp/sample.txt
                aws s3 cp /tmp/sample.txt s3://${ARTIFACT_BUCKET_NAME}/s3_sample.txt

                # Unity Build Serverからライセンスを取得:
                mkdir -p /usr/share/unity3d/config/
                echo '{
                "licensingServiceBaseUrl": "'"$UNITY_BUILD_SERVER_URL"'",
                "enableEntitlementLicensing": true,
                "enableFloatingApi": true,
                "clientConnectTimeoutSec": 5,
                "clientHandshakeTimeoutSec": 10}' > /usr/share/unity3d/config/services-config.json
                mkdir -p ./iOSProj
                mkdir -p ./Build/iosBuild
                unity-editor \
                    -quit \
                    -batchmode \
                    -nographics \
                    -executeMethod ExportTool.ExportXcodeProject \
                    -buildTarget iOS \
                    -customBuildTarget iOS \
                    -customBuildName iosBuild \
                    -customBuildPath ./Build/iosBuild \
                    -projectPath "./"
                echo "===Zipping Xcode project"
                zip -q -r -0 iOSProj iOSProj
                '''
                // pick up archive xcode project
                dir('') {
                    stash includes: 'iOSProj.zip', name: 'xcode-project'
                }
            }
            post {
                always {
                    // Unity Build Server利用時は明示的なライセンス返却処理は不要
                    // sh 'unity-editor -quit -returnlicense'
                    sh 'chmod -R 777 .'
                }
            }
        }
    }
    post {
        success {
            echo 'Success ^_^'
        }
        failure {
            echo 'Failed :('
        }
    }
}
