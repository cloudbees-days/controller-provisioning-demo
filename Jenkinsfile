library 'pipeline-library'
pipeline {
  agent none
  options { timeout(time: 10, unit: 'MINUTES') }
  triggers {
    eventTrigger jmespathQuery("controller.action=='provision'")
  }
  stages {
    stage('Provision Managed Controller') {
      agent {
        kubernetes {
          yaml libraryResource ('podtemplates/kubectl.yml')
        }
      }
      environment {
        ADMIN_CLI_TOKEN = credentials('admin-cli-token')
      }
      steps {
        gitHubParseOriginUrl()
        container("kubectl") {
          sh '''
            rm -rf ./${BUNDLE_ID} || true
            rm -rf ./checkout || true
            mkdir -p ${BUNDLE_ID}
            mkdir -p checkout
            git clone https://github.com/${GITHUB_ORG}/${GITHUB_REPO}.git checkout
          '''
          dir('checkout/bundle') {
            sh "cp --parents `find -name \\*.yaml*` ../../${BUNDLE_ID}/"
          }
          sh '''
            ls -la ${BUNDLE_ID}"
            kubectl cp --namespace cbci ${BUNDLE_ID} cjoc-0:/var/jenkins_home/jcasc-bundles-store/ -c jenkins
          '''
        }
        
        sh '''
          curl --user "$ADMIN_CLI_TOKEN_USR:$ADMIN_CLI_TOKEN_PSW" -XPOST \
            http://cjoc/cjoc/load-casc-bundles/checkout
            
          curl --user "$ADMIN_CLI_TOKEN_USR:$ADMIN_CLI_TOKEN_PSW" -XPOST \
            http://cjoc/cjoc/casc-items/create-items?path=/cloudbees-ci-casc-workshop \
            --data-binary @./controller.yaml -H 'Content-Type:text/yaml'
        '''
      }
    }
  }
}
