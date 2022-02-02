@Library('flexy') _

// rename build
def userId = currentBuild.rawBuild.getCause(hudson.model.Cause$UserIdCause)?.userId
if (userId) {
  currentBuild.displayName = userId
}

pipeline {
  agent none

  parameters {
        string(name: 'BUILD_NUMBER', defaultValue: '', description: 'Build number of job that has installed the cluster.')
        string(name: 'SCALE_UP', defaultValue: '0', description: 'If value is set to anything greater than 0, cluster will be scaled up before executing the workload.')
        string(name: 'SCALE_DOWN', defaultValue: '0', description:
        '''If value is set to anything greater than 0, cluster will be scaled down after the execution of the workload is complete,<br>
        if the build fails, scale down may not happen, user should review and decide if cluster is ready for scale down or re-run the job on same cluster.'''
        )
        string(name:'JENKINS_AGENT_LABEL',defaultValue:'oc45',description:
        '''
        scale-ci-static: for static agent that is specific to scale-ci, useful when the jenkins dynamic agen
 isn't stable<br>
        4.y: oc4y || mac-installer || rhel8-installer-4y <br/>
            e.g, for 4.8, use oc48 || mac-installer || rhel8-installer-48 <br/>
        3.11: ansible-2.6 <br/>
        3.9~3.10: ansible-2.4 <br/>
        3.4~3.7: ansible-2.4-extra || ansible-2.3 <br/>
        '''
        )
        text(name: 'ENV_VARS', defaultValue: '', description:'''<p>
               Enter list of additional (optional) Env Vars you'd want to pass to the script, one pair on each line. <br>
               e.g.<br>
               SOMEVAR1='env-test'<br>
               SOMEVAR2='env2-test'<br>
               ...<br>
               SOMEVARn='envn-test'<br>
               </p>'''
            )
        string(name: 'E2E_BENCHMARKING_REPO', defaultValue:'https://github.com/cloud-bulldozer/e2e-benchmarking', description:'You can change this to point to your fork if needed.')
        string(name: 'E2E_BENCHMARKING_REPO_BRANCH', defaultValue:'master', description:'You can change this to point to a branch on your fork if needed.')
    }

  stages {
    stage('Run Pod Network Policy Network Perf Test'){
      agent { label params['JENKINS_AGENT_LABEL'] }
      environment{
          EMAIL_ID_FOR_RESULTS_SHEET = "${userId}@redhat.com"
      }
      steps{
        script{
          if(params.SCALE_UP.toInteger() > 0) {
            build job: 'scale-ci/e2e-benchmarking-multibranch-pipeline/cluster-workers-scaling', parameters: [string(name: 'BUILD_NUMBER', value: BUILD_NUMBER), string(name: 'WORKER_COUNT', value: SCALE_UP), string(name: 'JENKINS_AGENT_LABEL', value: JENKINS_AGENT_LABEL)]
          }
        }
        deleteDir()
        checkout([
          $class: 'GitSCM', 
          branches: [[name: params.E2E_BENCHMARKING_REPO_BRANCH ]],
          doGenerateSubmoduleConfigurations: false, 
          userRemoteConfigs: [[url: params.E2E_BENCHMARKING_REPO ]
          ]])

        copyArtifacts(
            filter: '', 
            fingerprintArtifacts: true, 
            projectName: 'ocp-common/Flexy-install', 
            selector: specific(params.BUILD_NUMBER),
            target: 'flexy-artifacts'
        )
        script {
          buildinfo = readYaml file: "flexy-artifacts/BUILDINFO.yml"
          currentBuild.displayName = "${currentBuild.displayName}-${params.BUILD_NUMBER}"
          currentBuild.description = "Copying Artifact from Flexy-install build <a href=\"${buildinfo.buildUrl}\">Flexy-install#${params.BUILD_NUMBER}</a>"
          buildinfo.params.each { env.setProperty(it.key, it.value) }
        }
        ansiColor('xterm') {
          withCredentials([file(credentialsId: 'sa-google-sheet', variable: 'GSHEET_KEY_LOCATION')]) {
            sh label: '', script: '''
            # Get ENV VARS Supplied by the user to this job and store in .env_override
            echo "$ENV_VARS" > .env_override
            # Export those env vars so they could be used by CI Job
            set -a && source .env_override && set +a
            mkdir -p ~/.kube
            cp $WORKSPACE/flexy-artifacts/workdir/install-dir/auth/kubeconfig ~/.kube/config
            oc config view
            oc projects
            ls -ls ~/.kube/
            env
            cd workloads/network-perf
            ./run_pod_network_policy_test_fromgit.sh
            rm -rf ~/.kube
            '''
          }
        }
        script{
          // if the build fails, scale down will not happen, letting user review and decide if cluster is ready for scale down or re-run the job on same cluster
          if(params.SCALE_DOWN.toInteger() > 0) {
            build job: 'scale-ci/e2e-benchmarking-multibranch-pipeline/cluster-workers-scaling', parameters: [string(name: 'BUILD_NUMBER', value: BUILD_NUMBER), string(name: 'WORKER_COUNT', value: SCALE_DOWN), string(name: 'JENKINS_AGENT_LABEL', value: JENKINS_AGENT_LABEL)]
            }
        }
      }
        
    }
}
}

