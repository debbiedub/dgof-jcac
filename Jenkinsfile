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
// 2. fetch according to the repos' config
// 3. Push to freenet
// 4. If the clone didn't work, reinsert. The reinsert will abort if pushing.
//
// Fetch-URI is in the Jenkins settings. Private-URI in the freesitemgr config.

def fetchURI = "${env.FETCH_URI}"

def dgofdir = '/home/debbiedub/.dgof_sites'
def freesitemgrdir = '/home/debbiedub/.freesitemgr'
def mirrors = "${env.MIRRORS_DIR}"

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


    def files_list = sh (script: "ls $mirrors", returnStdout: true).trim()
    for (String dirname : files_list.split("\\r?\\n")) {
      stage(dirname) {
        boolean succeeded = false
        for (int i = 0; i < 5 && !succeeded; i++) {
	  // The recent cache is 1800s in the default configuration
	  // It is pointless to hit again before that is aged.
	  sh 'rm -rf newclone'
          int result = sh returnStatus: true, script: 'PATH="$PATH:$(pwd)/dgof" GIT_TRACE_REMOTE_FREENET=1 git clone ' + "freenet::$fetchURI$dirname/1 newclone"
          if (result == 0) {
            succeeded = true
	    continue
          }
	  sh 'rm -rf newclone'
          sleep(1850 + 8 * i)
        }
        sh "cd $mirrors/$dirname && git fetch --all && git push freenet"
        if (!succeeded) {
          sh "freesitemgr reinsert $dirname"
	  unstable "Could not clone the repo. Repo reinserted."
        }
      }
    }


    stage('wait for uploads') {
      retry (15) {
        // If any still inserting, we try again
        // This retry works as a while with limited amount of attempts
        sh """freesitemgr update `ls $mirrors` > output.txt
              cat output.txt
              if grep 'still inserting' < output.txt
              then
                  sleep 100
                  false
              fi"""
      }
    }
  }
}
