def event = currentBuild.getBuildCauses()[0].event
library 'pipeline-library'
pipeline {
  agent none
  options { timeout(time: 10, unit: 'MINUTES') }
  triggers {
    eventTrigger jmespathQuery("ref=='refs/heads/main' && commits[0].added[?contains(@, 'controller.yaml')]")
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
        GITHUB_ORG = event.organization.login.toString().replaceAll(" ", "-")
        GITHUB_ORG_LOWER = GITHUB_ORG.toLowerCase()
        GITHUB_REPO = event.repository.name.toString()
        BUNDLE_ID = "${GITHUB_ORG_LOWER}-${GITHUB_REPO}"
      }
      steps {
        container("kubectl") {
          sh '''
            rm -rf ./${BUNDLE_ID} || true
            rm -rf ./checkout || true
            mkdir -p ${BUNDLE_ID}
            mkdir -p checkout
            git clone https://github.com/${GITHUB_ORG}/${GITHUB_REPO}.git checkout
            sed -i "s/REPLACE_REPO/$GITHUB_REPO/g" checkout/controller.yaml
            sed -i "s/REPLACE_REPO/$GITHUB_REPO/g" checkout/bundle/bundle.yaml
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
            http://cjoc/cjoc/load-casc-bundles/checkout
            
          curl --user "$ADMIN_CLI_TOKEN_USR:$ADMIN_CLI_TOKEN_PSW" -XPOST \
            http://cjoc/cjoc/casc-items/create-items?path=/cloudbees-ci-previews-demo \
            --data-binary @./checkout/controller.yaml -H 'Content-Type:text/yaml'
        '''
      }
    }
  }
}
