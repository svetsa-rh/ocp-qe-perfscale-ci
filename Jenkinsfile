@Library('flexy') _

// rename build
def userCause = currentBuild.rawBuild.getCause(Cause.UserIdCause)
def upstreamCause = currentBuild.rawBuild.getCause(Cause.UpstreamCause)

userId = "svetsa"
if (userCause) {
  userId = userCause.getUserId()
} else if (upstreamCause) {
  def upstreamJob = Jenkins.getInstance().getItemByFullName(upstreamCause.getUpstreamProject(), hudson.model.Job.class)
  if (upstreamJob) {
    def upstreamBuild = upstreamJob.getBuildByNumber(upstreamCause.getUpstreamBuild())
    if (upstreamBuild) {
      def realUpstreamCause = upstreamBuild.getCause(Cause.UserIdCause)
      if (realUpstreamCause) {
        userId = realUpstreamCause.getUserId()
      }
    }
  }
}
if (userId) {
  currentBuild.displayName = userId
} 


println "user id $userId"
def RETURNSTATUS = "default"
def output = ""
def status = "FAIL"

pipeline {
  agent none

  parameters {
    string(
      name: 'GITHUB_USERNAME', 
      defaultValue: 'svetsa-rh', 
      description: 'Github user that has installed the cluster.'
    )
    string(
      name: 'CLUSTER_NAME', 
      defaultValue: 'sv-rosa-2', 
      description: 'Cluster name to be created.'
    )
    string(
      name: 'COMPUTE_WORKERS_TYPE', 
      defaultValue: 'm5.2xlarge', 
      description: 'AWS instance type to use for creating Compute Worker nodes.'
    )
    string(
      name: 'COMPUTE_WORKERS_NUMBER', 
      defaultValue: '2', 
      description:
      '''If value is set to anything greater than 2, cluster will be scaled up to this number.'''
    )
    string(
      name: 'NETWORK_TYPE', 
      defaultValue: 'OVNKubernetes', 
      description:
      '''Set to ovn by default'''
    )
    string(
        name: "INSTALLATION_PARAMS",
        defaultValue: ' --sts -m auto --yes',
        description: 'Specify any specific install params if necessary.'
    )
    string(
      name:'JENKINS_AGENT_LABEL',
      defaultValue:'oc413',
      description:
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
    text(
      name: 'ENV_VARS', 
      defaultValue: '', 
      description:'''<p>
        Enter list of additional (optional) Env Vars you'd want to pass to the script, one pair on each line. <br>
        e.g.<br>
        SOMEVAR1='env-test'<br>
        SOMEVAR2='env2-test'<br>
        ...<br>
        SOMEVARn='envn-test'<br>
        </p>'''
    )
  }

  stages {
    stage('Create rosa cluster'){
      agent { label params['JENKINS_AGENT_LABEL'] }
      steps{
        script {
          withCredentials([string(credentialsId: 'perfscale-rosa-token', variable: 'ROSA_TOKEN'),
            file(credentialsId: 'b73d6ed3-99ff-4e06-b2d8-64eaaf69d1db', variable: 'OCP_AWS')]) {
          sh(returnStatus: true, script: '''
            mkdir -p ~/.aws
            cp -f $OCP_AWS ~/.aws/credentials
            aws iam get-user
            rosa version
            ocm version
            echo $ROSA_TOKEN
            echo "Write to file"
            echo $ROSA_TOKEN >& /tmp/rosa-token
            cat /tmp/rosa-token
            rosa login --env=staging --token=$ROSA_TOKEN --debug
            rosa login --env=staging --token="$ROSA_TOKEN"
            echo 
            echo "Print debug output"
            rosa login --env=staging --token="$ROSA_TOKEN" --debug
            rosa verify openshift-client
            rosa whoami
            which oc
            oc version
            rosa create cluster --tags=User:${GITHUB_USERNAME} --cluster-name ${CLUSTER_NAME} \
              --compute-machine-type ${COMPUTE_WORKERS_TYPE} --replicas ${COMPUTE_WORKERS_NUMBER} \
              --network-type ${NETWORK_TYPE} ${INSTALLATION_PARAMS} 
            ''')
          }
        }
      }
    }
  }
}
