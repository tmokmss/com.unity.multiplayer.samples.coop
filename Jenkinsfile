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
                cache(maxCacheSize: 5000, caches: [arbitraryFileCache(path: './Library', compressionMethod: 'TARGZ')]) {
                    // install stuff for Unity, build xcode project, archive the result
                    sh '''#!/bin/bash
                    set -xe
                    printenv
                    ls -la
                    echo "===Installing stuff for unity"
                    apt-get update
                    apt-get install -y curl unzip zip jq

                    # Unity Build Serverを使う場合:
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
                        -projectPath "./" \
                    echo "===Zipping Xcode project"
                    zip -q -r -0 iOSProj iOSProj
                    '''
                }
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
        stage('build and sign iOS app on mac') {
            // we don't need the source code for this stage
            options {
                skipDefaultCheckout()
            }
            agent {
                label 'mac'
            }
            environment {
                PROJECT_FOLDER = 'iOSProj'
                CERT_PRIVATE = credentials('priv')
                CERT_SIGNATURE = credentials('development')
                BUILD_SECRET_JSON = credentials('ios-build-secret')
            }
            steps {
                unstash 'xcode-project'
                sh '''#!/bin/zsh
                set -xe
                printenv
                ls -l
                # Remove old project and unpack a new one
                sudo rm -rf ${PROJECT_FOLDER}
                unzip -q iOSProj.zip
                '''

                // create export options file
                writeFile file: "${env.PROJECT_FOLDER}/ExportOptions.plist", text: """
                <?xml version="1.0" encoding="utf-8"?>
                <!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
                <plist version="1.0">
                <dict>
                    <key>signingStyle</key>
                    <string>manual</string>
                </dict>
                </plist>
                """

                sh '''#!/bin/zsh
                set -xe
                # 必要なパスを通す
                export PATH=/usr/local/bin:/opt/homebrew/bin:\${PATH}
                cd ${PROJECT_FOLDER}
                TEAM_ID=$(echo $BUILD_SECRET_JSON | jq -r '.TEAM_ID')
                BUNDLE_ID=$(echo $BUILD_SECRET_JSON | jq -r '.BUNDLE_ID')
                # extra backslash for groovy
                sed -i "" "s/DEVELOPMENT_TEAM = \\"\\"/DEVELOPMENT_TEAM = $TEAM_ID/g" Unity-iPhone.xcodeproj/project.pbxproj
                #############################################
                # setup certificates in a temporary keychain
                #############################################
                echo "===Setting up a temporary keychain"
                pwd
                # security コマンドがec2-userではpermission errorになることがあるので、sudoで実行する
                # Unique keychain ID
                MY_KEYCHAIN="temp.keychain.`uuidgen`"
                MY_KEYCHAIN_PASSWORD="secret"
                sudo security create-keychain -p "$MY_KEYCHAIN_PASSWORD" "$MY_KEYCHAIN"
                # Append the temporary keychain to the user search list
                # double backslash for groovy
                sudo security list-keychains -d user -s "$MY_KEYCHAIN" $(security list-keychains -d user | sed s/\\"//g)
                # Output user keychain search list for debug
                sudo security list-keychains -d user
                # Disable lock timeout (set to "no timeout")
                sudo security set-keychain-settings "$MY_KEYCHAIN"
                # Unlock keychain
                sudo security unlock-keychain -p "$MY_KEYCHAIN_PASSWORD" "$MY_KEYCHAIN"
                echo "===Importing certs"
                # Import certs to a keychain; bash process substitution doesn't work with security for some reason
                sudo security -v import $CERT_SIGNATURE -k "$MY_KEYCHAIN" -T "/usr/bin/codesign"
                #rm /tmp/cert
                PASSPHRASE=""
                sudo security -v import $CERT_PRIVATE -k "$MY_KEYCHAIN" -P "$PASSPHRASE" -t priv -T "/usr/bin/codesign"
                # Dump keychain for debug
                sudo security dump-keychain "$MY_KEYCHAIN"
                # Set partition list (ACL) for a key
                sudo security set-key-partition-list -S apple-tool:,apple:,codesign: -s -k $MY_KEYCHAIN_PASSWORD $MY_KEYCHAIN
                # Get signing identity for xcodebuild command
                sudo security find-identity -v -p codesigning $MY_KEYCHAIN
                # double backslash for groovy
                CODE_SIGN_IDENTITY=`security find-identity -v -p codesigning $MY_KEYCHAIN | awk '/ *1\\)/ {print $2}'`
                echo code signing identity is $CODE_SIGN_IDENTITY
                sudo security default-keychain -s $MY_KEYCHAIN

                #############################################
                # Build
                #############################################
                echo ===Building
                pwd

                sudo xcodebuild -scheme Unity-iPhone -sdk iphoneos -configuration AppStoreDistribution archive -archivePath "$PWD/build/Unity-iPhone.xcarchive" CODE_SIGN_STYLE="Manual" CODE_SIGN_IDENTITY=$CODE_SIGN_IDENTITY OTHER_CODE_SIGN_FLAGS="--keychain=$MY_KEYCHAIN" -UseModernBuildSystem=0 CODE_SIGNING_REQUIRED=NO CODE_SIGNING_ALLOWED=NO

                # Generate ipa
                echo ===Exporting ipa
                pwd
                # xcodebuild -exportArchive -archivePath "$PWD/build/Unity-iPhone.xcarchive" -exportOptionsPlist ExportOptions.plist -exportPath "$PWD/build"

                sudo chown -R ec2-user "$PWD/build"

                #############################################
                # Upload
                #############################################
                # Upload to S3
                # /usr/local/bin/aws s3 cp ./build/*.ipa s3://${S3_BUCKET}/
                #############################################
                # Cleanup
                #############################################
                # Delete keychain - should be moved to a post step, but this would require a global variable or smth
                sudo security delete-keychain "$MY_KEYCHAIN"
                '''
            }
            post {
                always {
                    sh '''
                    #############################################
                    # cleanup
                    #############################################
                    zip -r iOSProj/build/Unity-iPhone.zip iOSProj/build/Unity-iPhone.xcarchive
                    '''
                    archiveArtifacts artifacts: '**/Unity-iPhone.zip', onlyIfSuccessful: true, caseSensitive: false
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
