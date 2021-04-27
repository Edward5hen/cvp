/**
 * This file is to implement the CVP testing in the "extras" way.
 * Listening build UMB messages -> trigger the test job -> running test -> send results to zelda -> post test result via UMB message.
 */

def ciMessage = params.CI_MESSAGE // UMB message which triggered the build
def testBranch = params.TEST_BRANCH // the branch for test project containers-ansible
def buildMetadata = [:]           // image build metadata; parsed from the UMB message
def result_flag = 0               // flag for testing result, used for the post sending UMB
def status = ''                   // testing result for post sending UMB
def namespace = ''                // namespace for sending result


// Load the contra-int-lib library which will be used for UMB message parsing
library identifier: "contra-int-lib@master",
        retriever: modernSCM([$class: 'GitSCMSource',
                              remote: "https://gitlab.sat.engineering.redhat.com/contra/contra-int-lib.git"])

pipeline {
  agent {
    // Ideally, jobs should _not_ run directly on a Jenkins master, but rather on a different agent (slave)
    // This sample is simple enough though and running on master requires less configuration.
    label params.AGENT
  }

  stages {
    stage("Parse 'redhat-container-image.pipeline.running' message") {
      steps {
        script {
          echo "Raw message:\n${ciMessage}"

          buildMetadata = extractCVPPipelineRunningMessageData(ciMessage)

          def metadataStr = buildMetadata
              .collect { meta -> "\t${meta.key} -> ${meta.value}"}
              .join("\n")
          echo "Build metadata:\n${metadataStr}"
        }
      }
    }

    stage("Run image tests") {
      steps {
        script {
          echo "---------------------- TEST START ---------------------"

          def img_fn = buildMetadata['full_name']
          def test_nvr = buildMetadata['nvr']

          try {
            sh """
                if [ ! -d zelda-backend ]
                then
                  git clone git@gitlab.cee.redhat.com:atomic-qe/zelda-backend.git
                fi
                cd zelda-backend/ansible-client
                git pull
                bash run_test.sh ${test_nvr} ${img_fn} ${testBranch}
            """
          }
          catch (exc) {
            result_flag = 1
          }
        }
      }
      post {
        always {
          script {
            // report test results to ResultsDB
            def provider = "Red Hat UMB" // change the provider to "Red Hat UMB Stage" for development purposes

            // the following three values need to match the configuration in gating.yaml
            def type = "default"
            def testName = "cvetest"
            if (result_flag == 0) {
              status = "PASSED"
            } else {
              status = "FAILED"
            }

            def brewTaskID = buildMetadata['id']
            def brewNvr = buildMetadata['nvr']
            def brewName = buildMetadata['name']
            def product = buildMetadata['component']
            namespace = "atomic-sadc-rhel7-container-test"

            def msgContent = """
             {
                "category": "${testName}",
                "status": "${status}",
                "ci": {
                    "url": "https://jenkins-cvp-ci.cloud.paas.upshift.redhat.com/",
                    "team": "atomic-qe",
                    "email": "weshen@redhat.com",
                    "name": "Edward Shen"
                },
                "run": {
                    "url": "${BUILD_URL}",
                    "log": "${BUILD_URL}/console"
                },
                "system": {
                    "provider": "openshift",
                    "os": "openshift"
                },
                "artifact": {
                    "nvr": "${brewNvr}",
                    "component": "${product}",
                    "type": "brew-build",
                    "id": "${brewTaskID}",
                    "issuer": "Unknown issuer"
                },
                "type": "${type}",
                "version": "0.1.0",
                "namespace": "${namespace}"
              }"""

            echo "Sending the following message to UMB:\n${msgContent}"

            sendCIMessage(messageContent: msgContent,
                messageProperties: '',
                messageType: 'Custom',
                overrides: [topic: "VirtualTopic.eng.ci.brew-build.test.complete"],
                providerName: provider)
          }
        }
      }
    }
  }
}
