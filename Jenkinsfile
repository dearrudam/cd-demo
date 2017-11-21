  env.DOCKERHUB_USERNAME = 'dearrudam'

  node("docker-test") {
    checkout scm

    stage("Unit Test") {
      sh "docker run --rm -v ${WORKSPACE}:/go/src/cd-demo golang go test cd-demo -v --run Unit"
    }
    stage("Integration Test") {
      try {
        sh "docker build -t it/cd-demo:${BUILD_NUMBER} ."
        sh "docker rm -f it_cd-demo_${BUILD_NUMBER} || true"
        sh "docker run -d --name=it_cd-demo_${BUILD_NUMBER} it/cd-demo:${BUILD_NUMBER}"
        // env variable is used to set the server where go test will connect to run the test
        sh "docker run --rm -v ${WORKSPACE}:/go/src/cd-demo --link=it_cd-demo_${BUILD_NUMBER} -e SERVER=it_cd-demo_${BUILD_NUMBER} golang go test cd-demo -v --run Integration"
      }
      catch(e) {
        error "Integration Test failed"
      }finally {
        sh "docker rm -f it_cd-demo_${BUILD_NUMBER} || true"
        sh "docker ps -aq | xargs docker rm || true"
        sh "docker images -aq -f dangling=true | xargs docker rmi || true"
        sh '''
            for img in $(docker images it/cd-demo | awk '{OFS=":"}{print \$1, \$2}' | grep -E '^it/cd-demo');
            do 
             docker rmi $img 
            done
           '''
      }
    }
    stage("Build Dev version") {
      sh "docker build -t ${DOCKERHUB_USERNAME}/cd-demo:dev.${BUILD_NUMBER} ."
      sh '''
            for img in $(docker images ${DOCKERHUB_USERNAME}/cd-demo | awk '{OFS=":"}{print \$1, \$2}' | grep -E ':dev.[0-9]{1,}$');
            do 
              if [[ "$img" != "${DOCKERHUB_USERNAME}/cd-demo:dev.${BUILD_NUMBER}" ]]; then
                docker rmi $img
              fi
            done 
         '''
    }
    //stage("Publish") {
    //  withDockerRegistry([credentialsId: 'DockerHub']) {
    //    sh "docker push ${DOCKERHUB_USERNAME}/cd-demo:${BUILD_NUMBER}"
    //  }
    //}
  }

  node("docker-stage") {
    checkout scm

    stage("Staging") {
      try {
        sh "docker rm -f st_cd-demo_${BUILD_NUMBER} || true"
        sh "docker run -d --name=st_cd-demo_${BUILD_NUMBER} ${DOCKERHUB_USERNAME}/cd-demo:dev.${BUILD_NUMBER}"
        sh "docker run --rm -v ${WORKSPACE}:/go/src/cd-demo --link=st_cd-demo_${BUILD_NUMBER} -e SERVER=st_cd-demo_${BUILD_NUMBER} golang go test cd-demo -v"

      } catch(e) {
        error "Staging failed"
      } finally {
        sh "docker rm -f st_cd-demo_${BUILD_NUMBER} || true"
        sh "docker ps -aq | xargs docker rm || true"
        sh "docker images -aq -f dangling=true | xargs docker rmi || true"
      }
    }
    stage("Build Prod Version") {
      sh "docker build -t ${DOCKERHUB_USERNAME}/cd-demo:${BUILD_NUMBER} ."
    }
  }

  node("docker-prod") {
    stage("Production") {
      try {
        // Create the service if it doesn't exist otherwise just update the image
        sh '''
          SERVICES=$(docker service ls --filter name=cd-demo --quiet | wc -l)
          if [[ "$SERVICES" -eq 0 ]]; then
            docker network rm cd-demo || true
            docker network create --driver overlay --attachable cd-demo
            docker service create --replicas 3 --network cd-demo --name cd-demo -p 8080:8080 ${DOCKERHUB_USERNAME}/cd-demo:${BUILD_NUMBER}
          else
            docker service update --image ${DOCKERHUB_USERNAME}/cd-demo:${BUILD_NUMBER} cd-demo
          fi
          '''
        // run some final tests in production
        checkout scm
        sh '''
          sleep 30s 
          for i in `seq 1 20`;
          do
            STATUS=$(docker service inspect --format '{{ .UpdateStatus.State }}' cd-demo)
            if [[ "$STATUS" != "updating" ]]; then
              docker run --rm -v ${WORKSPACE}:/go/src/cd-demo --network cd-demo -e SERVER=cd-demo golang go test cd-demo -v --run Integration
              break
            fi
            sleep 10s
          done
          
        '''
        sh '''
            for img in $(docker images ${DOCKERHUB_USERNAME}/cd-demo | awk '{OFS=":"}{print \$1, \$2}' | grep -E ':[0-9]{1,}$');
            do 
              if [[ "$img" != "${DOCKERHUB_USERNAME}/cd-demo:dev.${BUILD_NUMBER}" ]]; then
                docker rmi $img
              fi
            done
          '''
      }catch(e) {
        sh "docker service update --rollback  cd-demo"
        error "Service update failed in production"
      }finally {
        sh "docker ps -aq | xargs docker rm || true"
        
      }
    }
  }
