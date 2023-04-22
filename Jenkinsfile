// The common preparation
// 1. Install pyFreenet3 (could be prepared in the docker image used)
// 2. "install" dgof     (could be prepared in the docker image used)
// 3. Set up ~/freesitemgr-dgof-jcac as a config dir for freesitemgr
//    This is probably easiest done by adding a site using
//      freesitemgr --config-dir=~/freesitemgr-dgof-jcac add
//    and then removing it again.  The dir ~/freesitemgr-dgof-jcac
//    exists outside of the docker container.  Inside the docker
//    containers it is mapped to $HOME/.freesitemgr so --config-dir
//    need not be specified on the command line.  The reason for
//    not using a docker volume is to make it easier to access from
//    the outside, like when setting up new repos.
//
// The logic for every repo:
// outside of Jenkins:
// 1. git clone --mirror URL name
//    name is specified to not get the the .git suffix
// 2. dgof-setup --as-maintainer name (after doing cd name)
// 3. Remove the freesitemgr update command from the post-update hook
// 4. move the freesitemgr file from ~/.freesitemgr to ~/freesitemgr-dgof-jcac
//
// within this job
// 1. Clone and throw the clone away.
// 2. Fetch from source and push to freenet.
// 3. If the clone didn't work, reinsert. If the clone did work, update.
// 4. Wait for the inserts to complete in all projects.
//
// This job assumes that when allocating the same node and creating
// a new container with the same image, the same workspace is used.
// This is only true if there is a single executor on the node.
//
// Fetch-URI is in the Jenkins settings. Private-URI in the freesitemgr config.

def fetchURI = "${env.FETCH_URI}"

def dgofdir = '/home/debbiedub/.dgof_sites'
def freesitemgrdir = '/home/debbiedub/freesitemgr-dgof-jcac'
def mirrors = "${env.MIRRORS_DIR}"

// global variables:
//   docker_image
//   docker_params


def files_list

stage('Prepare docker image') {
  node ('debbies') {
    deleteDir()
    writeFile file:'Dockerfile', text: '''
FROM python:3

RUN pip3 install pyFreenet3
    '''
    // freesitemgr uses $HOME to find its dir (really os.path.expanduser("~"))
    // Both freesitemgr config and mirrors config points to dgof dir
    // using absolute path.
    docker_params = "--network=host --env HOME='${env.WORKSPACE}' -v $mirrors:$mirrors -v $dgofdir:$dgofdir -v $freesitemgrdir:${env.WORKSPACE}/.freesitemgr"
    docker_image = docker.build('pyfreenet:3')
  }
}


stage('Get dgof') {
  // Get and/or install dgof into the workspace
  // Commands using dgof will need $(pwd)/dgof to be added to the PATH
  node ('debbies') {
    docker_image.inside(docker_params) {
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


// The generated closure is used to perform the work done in the node.
// When it is done, it returns 0 or less.
// Whenever it needs to sleep it returns the amount of seconds to sleep.
// This is run in a way so that the node is released when sleeping
// to allow other legs in the parallel execution to run.
def gen_cl(name, mirrors, fetchURI) {
  int state = 1

  int laps1 = 1
  int laps2 = 1
  int laps3 = 1

  return {
    if (state == 1) {
      int result1 = sh returnStatus: true, script: """freesitemgr update $name | tee output.txt
	      egrep -v 'No update required|site insert has completed|checking if a new insert is needed' < output.txt"""
      if (result1 == 0 &&      // grep found something
          laps1++ < 10) {
        return 1000 + laps1 * 22
      }
      state = 2
    }

    if (state == 2) {
      def lap = laps2
      echo "Start cloning $name lap $lap"
      sh 'rm -rf newclone'
      int result2 = sh returnStatus: true, script: 'PATH="$PATH:$(pwd)/dgof" GIT_TRACE_REMOTE_FREENET=1 git clone ' + "freenet::$fetchURI$name/1 newclone"
      sh 'rm -rf newclone'
      if (result2 != 0) {
        if (laps2++ < 5) {
          // Wait before trying again.
          // The recent cache is 1800s in the default configuration
          // It is pointless to hit again before that is aged.
          echo "Cloning failed $name lap $lap - will retry"
          return 1800 + lap * 8
	} else {
	  echo "Cloning failed $name will reinsert"
	}
      }
      dir ("$mirrors/$name") {
        sh "git fetch --all && git push freenet"
        // Add a file with the used versions of the tools      
        sh '''cd $(git config --get remote.freenet.url) &&
          git --version > v.new &&
          head -1 `which freesitemgr` | sed 's/^#!//p;d' |
          sed 's/$/ --version/' | sh >> v.new &&
          pip3 list >> v.new &&
          freesitemgr --version >> v.new &&
          cmp -s versions v.new && rm v.new || mv v.new versions
        '''
      }
      state = 3
      if (result2 != 0) {
        sh "freesitemgr reinsert $name"
        unstable "Could not clone the repo. Repo reinserted."
	// There is no point in doing update immediately after reinsert.
	return 300
      }
    }

    if (state == 3) {
      int result3 = sh returnStatus: true, script: """freesitemgr update $name | tee output.txt
	      egrep -v 'No update required|site insert has completed|checking if a new insert is needed' < output.txt"""
      if (result3 == 0 &&      // grep found something
          laps3++ < 60) {
        return 600 + laps3 * 18
      }
      state = 4
    }

    return 0
  }
}


def buildParallelMap = [:]
def dirnames = files_list.split("\\r?\\n")
dirnames.each { dirname ->
  buildParallelMap[dirname] = {
    stage(dirname) {
      def cl = gen_cl(dirname, mirrors, fetchURI)
      def result = 1
      while ({
        node ('debbies') {
          docker_image.inside(docker_params) {
            result = cl()
	  }
	}
	result > 0
      }()) {
        sleep(result)
      }
    }
  }
}

parallel(buildParallelMap)
