pipeline {
    // 스테이지 별로 다른 거
    agent any

    triggers {  // 몇 분 주기로 파이프라인을 구동할 지
        pollSCM('*/3 * * * *')  // 3분 주기
    }

    // AWS key를 입력하여 접속 환경 구성
    environment {
      // 젠킨스의 시스템 환경 변수로 등록
      AWS_ACCESS_KEY_ID = credentials('awsAccessKeyId')
      AWS_SECRET_ACCESS_KEY = credentials('awsSecretAccessKey')
      AWS_DEFAULT_REGION = 'ap-northeast-2'
      HOME = '.' // Avoid npm root owned
    }

    // 파이프라인 정의 시작
    stages {
        // 레포지토리를 다운로드 받음
        stage('Prepare') {
            agent any
            
            steps {
                echo 'Clonning Repository'

                git url: 'https://github.com/seoul-chan/JenkinsTest.git',
                    branch: 'master',
                    credentialsId: 'jenkinsGit'   // credentials에서 GitHub 토큰 정보 입력한 ID
            }

            post {  // post : stage 끝난 후 실행
                // If Maven was able to run the tests, even if some of the test
                // failed, record the test results and archive the jar file.
                success { // 성공 시 출력
                    echo 'Successfully Cloned Repository'
                }

                always {  // 항상 출력
                  echo "i tried..."
                }

                cleanup { // post 영역의 모든 작업 처리 후 출력
                  echo "after all other post condition"
                }
            }
        }

        // stage('Only for production') {
        //   // when 사용예시
        //     when {  // production브랜치이고 APP_ENV가 prod일 경우 실행
        //       branch 'production'
        //       environment name: 'APP_ENV', value: 'prod'
        //       anyOf {
        //         environment name: 'DEPLOY_TO', value: 'production'
        //         environment name: 'DEPLOY_TO', value: 'staging'
        //       }
        //     }
        // }
        
        // aws s3 에 파일을 올림
        stage('Deploy Frontend') {
          steps {
            echo 'Deploying Frontend'
            // 프론트엔드 디렉토리의 정적파일들을 S3 에 올림, 이 전에 반드시 EC2 instance profile 을 등록해야함.
            dir ('./website'){  //디렉토리
            // website 폴더에 있는 파일을 S3에 업로드, 생성한 S3 Name을 입력
                sh '''
                aws s3 sync . s3://jenkinstest-20230104
                '''
            }
          }

          post {
              // If Maven was able to run the tests, even if some of the test
              // failed, record the test results and archive the jar file.
              success {
                  echo 'Successfully Cloned Repository'

                  // mail  to: 'kkchan9210@gmail.com',
                  //       subject: "Deploy Frontend Success",
                  //       body: "Successfully deployed frontend!"

              }

              // failure {
              //     echo 'I failed :('

              //     mail  to: 'kkchan9210@gmail.com',
              //           subject: "Failed Pipelinee",
              //           body: "Something is wrong with deploy frontend"
              // }
          }
        }
        
        stage('Lint Backend') {
            // Docker plugin and Docker Pipeline 두개를 깔아야 사용가능!
            agent {       // docker로 일을 진행하며 node 최신 버전을 사용
              docker {
                image 'node:latest'
              }
            }
            
            steps {   // npm 설치 및 lint 단계
              dir ('./server'){
                  sh '''
                  npm install&&
                  npm run lint
                  '''
              }
            }
        }
        
        stage('Test Backend') {
          agent {
            docker {
              image 'node:latest'
            }
          }
          steps {
            echo 'Test Backend'

            dir ('./server'){
                sh '''
                npm install
                npm run test
                '''
            }
          }
        }
        
        stage('Bulid Backend') {
          agent any
          steps {
            echo 'Build Backend'

            dir ('./server'){
              // -t : 태그 - server로 지정
                sh """
                docker build . -t server --build-arg env=${PROD}
                """
            }
          }

          post {
            failure {   // 실패 시 error 전달 후 pipeline 종료
              error 'This pipeline stops here...'
            }
          }
        }
        
        stage('Deploy Backend') {
          agent any

          steps {
            echo 'Build Backend'

            dir ('./server'){
              // 태그로 지정한 server를 실행
                sh '''
                docker rm -f $(docker ps -aq)
                docker run -p 80:80 -d server
                '''
            }
          }

          post {
            success {
              mail  to: 'frontalnh@gmail.com',
                    subject: "Deploy Success",
                    body: "Successfully deployed!"
                  
            }
          }
        }
    }
}
