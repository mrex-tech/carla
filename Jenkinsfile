#!/usr/bin/env groovy

pipeline
{
    agent none

    options
    {
        buildDiscarder(logRotator(numToKeepStr: '3', artifactNumToKeepStr: '3'))
    }

    stages
    {
        stage('Creating nodes')
        {
            agent { label "master" }
            steps
            {
                script
                {
                    JOB_ID = "${env.BUILD_TAG}"
                    jenkinsLib = load("/home/jenkins/jenkins.groovy")

                    jenkinsLib.CreateUbuntuBuildNode(JOB_ID)
                    jenkinsLib.CreateWindowsBuildNode(JOB_ID)
                }
            }
        }
        stage('Building CARLA')
        {
            parallel
            {
                stage('ubuntu')
                {
                    agent { label "ubuntu && build && ${JOB_ID}" }
                    environment
                    {
                        UE4_ROOT = '/home/jenkins/UnrealEngine_4.24'
                    }
                    stages
                    {
                        stage('ubuntu setup')
                        {
                            steps
                            {
                                sh 'make setup ARGS="--python3-version=3.7"'
                            }
                        }
                        stage('ubuntu build')
                        {
                            steps
                            {
                                sh 'make LibCarla'
                                sh 'make PythonAPI ARGS="--python3-version=3.7"'
                                sh 'make CarlaUE4Editor'
                                sh 'make examples'
                            }
                            post
                            {
                                always
                                {
                                    archiveArtifacts 'PythonAPI/carla/dist/*.egg'
                                    stash includes: 'PythonAPI/carla/dist/*.egg', name: 'ubuntu_eggs'
                                }
                            }
                        }
                        stage('ubuntu unit tests')
                        {
                            steps
                            {
                                sh 'make check ARGS="--all --xml --python3-version=3.7"'
                            }
                            post
                            {
                                always
                                {
                                    junit 'Build/test-results/*.xml'
                                    archiveArtifacts 'profiler.csv'
                                }
                            }
                        }
                        stage('ubuntu retrieve content')
                        {
                            steps
                            {
                                sh './Update.sh'
                            }
                        }
                        stage('ubuntu package')
                        {
                            steps
                            {
                                sh 'make package ARGS="--python3-version=3.7"'
                                sh 'make package ARGS="--packages=AdditionalMaps --clean-intermediate --python3-version=3.7"'
                                sh 'make examples ARGS="localhost 3654"'
                            }
                            post
                            {
                                always
                                {
                                    archiveArtifacts 'Dist/*.tar.gz'
                                    stash includes: 'Dist/CARLA*.tar.gz', name: 'ubuntu_package'
                                    stash includes: 'Examples/', name: 'ubuntu_examples'
                                }
                            }
                        }
                        stage('ubuntu deploy')
                        {
                            when { anyOf { branch "master"; buildingTag() } }
                            steps
                            {
                                sh 'git checkout .'
                                sh 'make deploy ARGS="--replace-latest --docker-push"'
                            }
                        }
                        stage('ubuntu Doxygen')
                        {
                            when { anyOf { branch "master"; buildingTag() } }
                            steps
                            {
                                sh 'rm -rf ~/carla-simulator.github.io/Doxygen'
                                sh '''
                                    cd ~/carla-simulator.github.io
                                    git fetch
                                    git checkout -B master origin/master
                                '''
                                sh 'make docs'
                                sh 'cp -rf ./Doxygen ~/carla-simulator.github.io/'
                                sh '''
                                    cd ~/carla-simulator.github.io
                                    git add Doxygen
                                    git commit -m "Updated c++ docs" || true
                                    git push
                                '''
                            }
                            post
                            {
                                always
                                {
                                    deleteDir()
                                }
                            }
                        }
                    }
                    post
                    {
                        always
                        {
                            deleteDir()

                            node('master')
                            {
                                script
                                {
                                    JOB_ID = "${env.BUILD_TAG}"
                                    jenkinsLib = load("/home/jenkins/jenkins.groovy")

                                    jenkinsLib.DeleteUbuntuBuildNode(JOB_ID)
                                }
                            }
                        }
                    }
                }
                stage('windows')
                {
                    agent { label "windows && build && ${JOB_ID}" }
                    environment
                    {
                        UE4_ROOT = 'C:\\Program Files\\Epic Games\\UE_4.24'
                    }
                    stages
                    {
                        stage('windows setup')
                        {
                            steps
                            {
                                bat """
                                    call ../setEnv64.bat
                                    make setup
                                """
                            }
                        }
                        stage('windows build')
                        {
                            steps
                            {
                                bat """
                                    call ../setEnv64.bat
                                    make LibCarla
                                """
                                bat """
                                    call ../setEnv64.bat
                                    make PythonAPI
                                """
                                bat """
                                    call ../setEnv64.bat
                                    make CarlaUE4Editor
                                """
                            }
                            post
                            {
                                always
                                {
                                    archiveArtifacts 'PythonAPI/carla/dist/*.egg'
                                    stash includes: 'PythonAPI/carla/dist/*.egg', name: 'windows_eggs'
                                }
                            }
                        }
                        stage('windows retrieve content')
                        {
                            steps
                            {
                                bat """
                                    call ../setEnv64.bat
                                    call Update.bat
                                """
                            }
                        }
                        stage('windows package')
                        {
                            steps
                            {
                                bat """
                                    call ../setEnv64.bat
                                    make package
                                """
                                bat """
                                    call ../setEnv64.bat
                                    make package ARGS="--packages=AdditionalMaps --clean-intermediate"
                                """
                            }
                            post {
                                always {
                                    archiveArtifacts 'Build/UE4Carla/*.zip'
                                }
                            }
                        }
                        stage('windows deploy')
                        {
                            when { anyOf { branch "master"; buildingTag() } }
                            steps {
                                bat """
                                    call ../setEnv64.bat
                                    git checkout .
                                    make deploy ARGS="--replace-latest"
                                """
                            }
                        }
                    }
                    post
                    {
                        always
                        {
                            deleteDir()

                            node('master')
                            {
                                script
                                {
                                    JOB_ID = "${env.BUILD_TAG}"
                                    jenkinsLib = load("/home/jenkins/jenkins.groovy")

                                    jenkinsLib.DeleteWindowsBuildNode(JOB_ID)
                                }
                            }
                        }
                    }
                }
            }
        }
    }
}
