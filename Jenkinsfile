library 'pipeline-library'
pipeline {
  agent none
  options { 
    timeout(time: 10, unit: 'MINUTES')
    skipDefaultCheckout true
  }
  stages {
    stage ('Fix Changelog') {
      // only do this if there is no prior build
      when { expression { return !currentBuild.previousBuild } }
      steps {
        checkout([
            $class: 'GitSCM',
            branches: scm.branches,
            userRemoteConfigs: scm.userRemoteConfigs,
            browser: scm.browser,
            // this extension builds the changesets from the compareTarget branch
            // Using a variable here, but do what's appropriate for your env
            extensions: [[$class: 'ChangelogToBranch', options: [compareRemote: 'origin', compareTarget: 'main']]]
        ])
      }
    }
    stage("Checkout") {
      steps {
        dir('checkout') {
          checkout scm
        }
      }
    } 
    stage('Provision Managed Controller') {
      agent {
        kubernetes {
          yaml libraryResource ('podtemplates/kubectl.yml')
        }
      }
      environment {
        ADMIN_CLI_TOKEN = credentials('admin-cli-token')
      }
      when {
        branch 'main'
        anyOf {
          changeset "controller.yaml"
        }
      }
      steps {
        container("kubectl") {
          gitHubParseOriginUrl()
          sh '''
            rm -rf ./checkout || true
            mkdir -p checkout
          '''
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
            http://cjoc/cjoc/casc-items/create-items?path=/cloudbees-ci-previews-demo \
            --data-binary @./checkout/controller.yaml -H 'Content-Type:text/yaml'
        '''
      }
    }
  }
}
