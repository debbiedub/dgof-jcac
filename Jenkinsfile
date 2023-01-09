// The common preparation
// 1. Install pyFreenet3 (could be prepared in the docker image used)
// 2. "install" dgof     (could be prepared in the docker image used)
//
// The logic for every repo:
// outside of Jenkins:
// 1. git clone --mirror URL name
//    name is specified to not get the the .git suffix
// 2. dgof-setup --as-maintainer name (after doing cd name)
//
// within this job
// 1. Clone and throw the clone away.
// 2. If the clone didn't work, reinsert.
// 3. Wait for the reinserts to complete in all projects.
// 4. Fetch all and push to freenet.
// 5. Wait for push to complete in all projects.
//
// This job assumes that when allocating the same node and creating
// a new container with the same image, the same workspace is used.
// This is only true if there is a single executor on the node.
//
// Fetch-URI is in the Jenkins settings. Private-URI in the freesitemgr config.

def fetchURI = "${env.FETCH_URI}"

def dgofdir = '/home/debbiedub/.dgof_sites'
def freesitemgrdir = '/home/debbiedub/.freesitemgr'
def mirrors = "${env.MIRRORS_DIR}"

def waitForUpdatesToComplete(mirrors, laps) {
  boolean succeeded = false
  for (int i = 0; i < laps && !succeeded; i++) {
    // If any still inserting, we try again
    // This retry works as a while with limited amount of attempts
    int result = sh returnStatus: true, script: """freesitemgr update `ls $mirrors` > output.txt
          cat output.txt
    	  grep 'still inserting' < output.txt"""
    if (result != 0) {      // grep didn't find anything
      succeeded = true
    } else {
      sleep(100)
    }
  }
}

def files_list
def docker_params = "--network=host --env HOME='${env.WORKSPACE}' -v $mirrors:$mirrors -v $dgofdir:$dgofdir -v $freesitemgrdir:${env.WORKSPACE}/.freesitemgr"

stage('Get dgof') {
  node ('debbies') {
    deleteDir()
    writeFile file:'Dockerfile', text: '''
FROM python:3

RUN pip3 install pyFreenet3
    '''
    // freesitemgr uses $HOME to find its dir (really os.path.expanduser("~"))
    // Both freesitemgr config and mirrors config points to dgof dir
    // using absolute path.
    docker.build('pyfreenet:3').inside(docker_params) {
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
      files_list = sh (script: "ls $mirrors", returnStdout: true).trim()
    }
  }
}

def buildParallelMap = [:]
for (String dirname : files_list.split("\\r?\\n")) {
  buildParallelMap.put(dirname,
    stage(dirname) {
      boolean succeeded = false
      for (int i = 0; i < 10 && !succeeded; i++) {
        // The recent cache is 1800s in the default configuration
        // It is pointless to hit again before that is aged.
        node ('debbies') {
          docker.image('pyFreenet:3').inside(docker_params) {
            sh 'rm -rf newclone'
            int result = sh returnStatus: true, script: 'PATH="$PATH:$(pwd)/dgof" GIT_TRACE_REMOTE_FREENET=1 git clone ' + "freenet::$fetchURI$dirname/1 newclone"
            sh 'rm -rf newclone'
            if (result == 0) {
              succeeded = true
              continue
            }
	  }
        }
        sleep(1850 + 8 * i)
      }
      if (!succeeded) {
        node ('debbies') {
          docker.image('pyFreenet:3').inside(docker_params) {
            sh "freesitemgr reinsert $dirname"
            unstable "Could not clone the repo. Repo reinserted."
	  }
        }
      }
    })
}

parallel(buildParallelMap)

stage('wait for reinserts') {
  node ('debbies') {
    docker.image('pyFreenet:3').inside(docker_params) {
      waitForUpdatesToComplete(mirrors, 30)
    }
  }
}

stage('update') {
  node ('debbies') {
    docker.image('pyFreenet:3').inside(docker_params) {
      for (String dirname : files_list.split("\\r?\\n")) {
        sh "cd $mirrors/$dirname && git fetch --all && git push freenet"
      }
      waitForUpdatesToComplete(mirrors, 15)
    }
  }
}
