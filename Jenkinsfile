// The common preparation
// 1. Install pyFreenet3 (could be prepared in the docker image used)
// 2. "install" dgof     (could be prepared in the docker image used)
//
// The logic for every repo:
// outside of Jenkins:
// 1. git clone --mirror
// 2. dgof_setup --as-maintainer
//
// within this job
// 1. clone and throw the clone away
// 2. fetch from github
// 3. Push to freenet
// 4. If the clone didn't work, reinsert. The reinsert will abort if pushing.
//
// Fetch-URI in the script. Private-URI in the freesitemgr config.
//
// The Organization plugin requires the Jenkinsfile to be in the repo.
// I will work with stages instead.

def fetchURI = 'USK@Mm9MIkkeQhs~OMiCQ~83Vs48EvNwVRxjfeoFMOQHUYI,AxOZEuOyRM7oJjU43HFErhVw06ZIJLb8GMKNheWR3g4,AQACAAE/'

def dgofdir = '/home/debbiedub/.dgof_sites'
def freesitemgrdir = '/home/debbiedub/.freesitemgr'
def mirrors = '/home/debbiedub/JenkinsSlave/mirrors'

node ('debbies') {
  deleteDir()
  writeFile file:'Dockerfile', text: '''
FROM python:3

RUN pip3 install pyFreenet3
  '''
  // freesitemgr uses $HOME to find its dir (really os.path.expanduser("~"))
  // Both freesitemgr config and mirrors config points to dgof dir
  // using absolute path.
  docker.build('pyfreenet:3').inside("--network=host --env HOME='${env.WORKSPACE}' -v $mirrors:$mirrors -v $dgofdir:$dgofdir -v $freesitemgrdir:${env.WORKSPACE}/.freesitemgr") {
    stage('Get dgof') {
      sh '''
        if git clone http://localhost:8888/freenet:USK@nrDOd1piehaN7z7s~~IYwH-2eK7gcQ9wAtPMxD8xPEs,y61pkcoRy-ccB7BHvLCzt3RUjeMILf8ox26NKvPZ-jk,AQACAAE/dgof/26/ dgof 2> gitclone.out
        then
          cat gitclone.out 1>&2
        else
          cp gitclone.out newusk
          sed -i '$s/.*USK@/USK@/p;d' newusk
          sed -i 's,\\(/dgof/[0-9]*/\\).*,\\1,' newusk
          git clone http://localhost:8888/freenet:$(cat newusk) dgof
        fi
        '''
    }

    def dirnames = []
    new File(mirrors).eachDir { dir ->
      dirnames.add(dir.name)
    }
    dirnames.sort().each { dirname ->
      stage(dirname) {
        boolean succeeded = false
        for (int i = 0; i < 5 && !succeeded; i++) {
          sleep(1 + 20 * i)
	  sh 'rm -rf newclone'
          int result = sh returnStatus: true, script: 'PATH="$PATH:$(pwd)/dgof" git clone ' + "freenet::$fetchURI$dirname/0 newclone"
          if (result == 0) {
            succeeded = true
          }
	  sh 'rm -rf newclone'
        }
        sh "cd $mirrors/$dirname && git fetch --all && git push freenet"
        if (!succeeded) {
          sh "freesitemgr reinsert $dirname"
	  unstable "Could not clone the repo. Repo reinserted."
        }
      }
    }
  }
}
