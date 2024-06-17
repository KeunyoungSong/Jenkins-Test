pipeline { // 파이프라인의 시작
    // 스테이지 별로 다른 거 
    agent any // 아무 노예를 써라

    triggers {  // 얼마 주기로 실행될 건가
        pollSCM('*/3 * * * *')  // 크론 신텍스, 3분 주기 트리거 설정
    }

    environment {  // 이 파이프 라인 안에서 쓸 환경변수들 추가, 현재 github access token 은 어드민 페이지를 통해 등록했지만
    // AWS 의 ECS, ECR 을 사용하기 위해 추가 환경변수 등록이 필요함 
      AWS_ACCESS_KEY_ID = credentials('awsAccessKeyId') // AWS 엑세스키
      AWS_SECRET_ACCESS_KEY = credentials('awsSecretAccessKey') // AWS 비밀 엑세스 키
      AWS_DEFAULT_REGION = 'ap-northeast-2'
      HOME = '.' // Avoid npm root owned
    }

    stages {  // 내부에 여러 stage 를 가지고 각 스테이지에서는 준비, 설정, 작업, 배포 등 여러가지 일을 수행한다.
        // 레포지토리를 다운로드 받음
        stage('Prepare') {
            agent any
            
            steps {  // 소스코드 다운
                echo 'Clonning Repository'

                git url: 'https://github.com/KeunyoungSong/Jenkins-Test.git',
                    branch: 'master',
                    credentialsId: 'git-access'
            }

            post {
                // If Maven was able to run the tests, even if some of the test
                // failed, record the test results and archive the jar file.
                success {  // 성공 시 
                    echo 'Successfully pulled Repository'
                }

                always { //  항상 하는 작업
                  echo "i tried..."
                }

                cleanup {  // 모든 작업이 끝난 후
                  echo "after all other post condition"
                }
            }
        }

        // when 분기 사용법
        // stage('Only for Production'){
        //   when {
        //     branch 'production'
        //     environment name: 'APP_ENV', value = 'prod'
        //     anyOf{
        //       environment name: 'DEPLOY_TO', value: 'production'
        //       environment name: 'DEPLOY_TO', value: 'staging'
        //     }
        //   }
        // }
        
        // aws s3 에 파일을 올림, 실제로는 이 단계에서 다양한 프론트엔드 작업이 이뤄짐(lint 등)
        stage('Deploy Frontend') {
          steps {
            echo 'Deploying Frontend'
            // 프론트엔드 디렉토리의 정적파일들을 S3 에 올림, 이 전에 반드시 EC2 instance profile 을 등록해야함.
            dir ('./website'){
                sh '''
                aws s3 sync ./ s3://song-jenkins-test
                '''
            }
          }

          post {
              // If Maven was able to run the tests, even if some of the test
              // failed, record the test results and archive the jar file.
              success {  // 성공 시 이메일 보냄, 메일을 보내기 위해서도 추가 설정 필요
                  echo 'Successfully Cloned Repository'

                  mail  to: 'rmsdud623@gmail.com',
                        subject: "Deploy Frontend Success",
                        body: "Successfully deployed frontend!"

              }

              failure {
                  echo 'I failed :('

                  mail  to: 'rmsdud623@gmail.com',
                        subject: "Failed Pipelinee",
                        body: "Something is wrong with deploy frontend"
              }
          }
        }

        stage('Lint Backend') {
            // Docker plugin and Docker Pipeline 두개를 깔아야 사용가능!
            agent {
              docker { // 퍼블릭 node 이미지를 받아와 도커에서 node 를 돌림
                image 'node:latest'
              }
            }
            
            steps {
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

            dir ('./server'){ // 테스트 코드 실행(index.test.js)
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
            // 서버는 도커를 만들어서 배포할 것 docker build
            // 이 명령어를 쓰려면 도커를 젠킨스 노드에 직접 설치해줘야 함? 거기서 도커 컨테이너를 실행시켜 놔야함?
            // 아래선 환경변수에  PROD 를 사용하는데 각각의 설정 값을 개발, 운용 등으로 추상화 하고 실제 키관리는 AWS 서비스를
            // 통해서 관리하는 게 좋은 설계라고 생각
            dir ('./server'){
                sh """
                docker build . -t server --build-arg env=${PROD}
                """
            }
          }

          post { // post 는 step 이 끝난 이후 실행됨
            failure {  // 서버 빌드 실패 시 나머지 파이프라인 종료 시켜
              error 'This pipeline stops here...'
            }
          }
        }
        
        // 실 환경에서는 ECS나 쿠버네티스 업데이트를 함
        // 여기선 원래 떠있던 도커 이미지 (지우고) 새로 실행시킴
        //docker rm -f $(docker ps -aq)
        stage('Deploy Backend') {
          agent any
                
          steps {
            echo 'Build Backend'

            dir ('./server'){
                sh '''
                docker run -p 80:80 -d server
                '''
            }
          }

          post {
            success {
              mail  to: 'rmsdud623@gmail.com',
                    subject: "Deploy Success",
                    body: "Successfully deployed!"
                  
            }
          }
        }
    }
}
