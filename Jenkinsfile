library 'pipeline-library'
pipeline {
  agent none
  options { 
    timeout(time: 10, unit: 'MINUTES')
    skipDefaultCheckout true
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
        container("kubectl") {
          sh '''
            rm -rf ./checkout || true
            mkdir -p checkout
          '''
          dir('checkout') {
            checkout scm
            gitHubParseOriginUrl()
          }
          sh '''
            rm -rf ./${BUNDLE_ID} || true
            mkdir -p ${BUNDLE_ID}
            sed -i "s/REPLACE_REPO/$GITHUB_REPO/g" checkout/controller.yaml
            sed -i "s/REPLACE_REPO/$GITHUB_REPO/g" checkout/bundle/bundle.yaml
            sed -i "s/REPLACE_REPO/$GITHUB_REPO/g" checkout/bundle/variables.yaml
          '''
          dir('checkout/bundle') {
            sh "cp --parents `find -name \\*.yaml*` ../../${BUNDLE_ID}/"
          }
          sh '''
            ls -la ${BUNDLE_ID}
            kubectl cp --namespace cbci ${BUNDLE_ID} cjoc-0:/var/jenkins_config/jcasc-bundles-store/ -c jenkins
          '''
        }
        
        sh '''
          curl --user "$ADMIN_CLI_TOKEN_USR:$ADMIN_CLI_TOKEN_PSW" -XPOST \
            -H "Accept: application/json" \
            http://cjoc/cjoc/load-casc-bundles/checkout
            
          curl --user "$ADMIN_CLI_TOKEN_USR:$ADMIN_CLI_TOKEN_PSW" -XPOST \
            -H "Accept: application/json" \
            http://cjoc/cjoc/casc-items/create-items?path=/controllers \
            --data-binary @./checkout/controller.yaml -H 'Content-Type:text/yaml'
        '''
      }
    }
  }
}
