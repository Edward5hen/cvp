/**
 * This file is to implement the CVP testing in the "extras" way.
 * Listening build UMB messages -> trigger the test job -> running test -> send results to zelda -> post test result via UMB message.
 */

def ciMessage = params.CI_MESSAGE // UMB message which triggered the build
def buildMetadata = [:]           // image build metadata; parsed from the UMB message
def result_flag = 0               // flag for testing result, used for the post sending UMB
def status = ''                   // testing result for post sending UMB
def namespace = ''                // namespace for sending result


// Load the contra-int-lib library which will be used for UMB message parsing
library identifier: "contra-int-lib@master",
        retriever: modernSCM([$class: 'GitSCMSource',
                              remote: "https://gitlab.cee.redhat.com/cvp/contra-int-lib.git"])

pipeline {
  agent {
    // Ideally, jobs should _not_ run directly on a Jenkins master, but rather on a different agent (slave)
    // This sample is simple enough though and running on master requires less configuration.
    label params.AGENT
  }

  stages {
    //stage("Parse 'redhat-container-image.pipeline.running' message") {
    //  steps {
    //    script {
    //      echo "Raw message:\n${ciMessage}"

    //      buildMetadata = extractCVPPipelineRunningMessageData(ciMessage)

    //      def metadataStr = buildMetadata
    //          .collect { meta -> "\t${meta.key} -> ${meta.value}"}
    //          .join("\n")
    //      echo "Build metadata:\n${metadataStr}"
    //    }
    //  }
    //}

    stage("Run image tests") {
      steps {
        script {
          echo "just for test"

        }
      }
      post {
        always {
          script {
            // report test results to ResultsDB
            def provider = "Red Hat UMB" // change the provider to "Red Hat UMB Stage" for development purposes

            def msgContent = """
             {
               "artifact": {
                "component": "ubi8-minimal",
                "full_names": [
                "registry-proxy.engineering.redhat.com/rh-osbs/ubi8-minimal:8.8-787"
                ],
                "id": "sha256:76693708730539a59eab52dd707edb355aff0be0d55923598f4dceeb3e92db83",
                "issuer": "odcs/odcs-backend05.hosts.prod.psi.bos.redhat.com",
                "nvr": "ubi8-minimal-container-8.8-787",
                "scratch": false,
                "type": "redhat-container-image"
                },
              "contact": {
                "docs": "https://docs.engineering.redhat.com/display/CVP/Container+Verification+Pipeline+E2E+Documentation",
                "email": "container-runtime-qe@redhat.com",
                "name": "container-runtime-qe",
                "team": "Container Runtime QE Team"
              },
              "generated_at": "2023-04-18T08:18:42.754308Z",
              "pipeline": {
                "id": "0e2db094-bc51-4998-82de-3ecdbc739f49",
                "name": "crq-product-test"
              },
              "run": {
                "log": "https://",
                "url": "http://"
              },
              "system": [
                {
                  "architecture": "x86_64",
                  "os": "rhel8.8",
                  "provider": "beaker"
                }
              ],
              "test": {
                "category": "sanity",
                "namespace": "crq.rhproduct",
                "result": "passed",
                "type": "default"
              },
              "version": "1.1.17"
              }"""

            echo "Sending the following message to UMB:\n${msgContent}"

            sendCIMessage(messageContent: msgContent,
                messageProperties: '',
                messageType: 'Custom',
                overrides: [topic: "VirtualTopic.eng.ci.container-runtime-cvp.redhat-container-image.test.complete"],
                providerName: provider)
          }
        }
      }
    }
  }
}
