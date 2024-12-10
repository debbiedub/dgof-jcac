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
// The logic for every repo outside of Jenkins:
// 1. git clone --mirror URL name
//    name is specified to not get the the .git suffix
// 2. dgof-setup --as-maintainer --bootstrap name (after doing cd name)
//    Add the private URI that matches the FETCH_URI of the jenkins
//    configuration and is suffixed with name
//    If it fails because of the cannot pack into small enough files,
//    redo without --bootstrap and the repo will not be available
//    without dgof
// 3. Deactivate the default update for the freenet remote for the mirrored
//    repo:
//      git config remote.freenet.skipDefaultUpdate true
// 4. Remove the freesitemgr update command and the echos from the
//    post-update hook in the dgof/name/hooks directory
// 5. Make the first freesitemgr update
// 6. Move the freesitemgr file from ~/.freesitemgr to ~/freesitemgr-dgof-jcac
// 
//
// Within this job, for every repo
// 1. Clone and throw the clone away.
// 2. Fetch from source and push to the copy used for freenet
// 3. If the clone didn't work, reinsert. If the clone did work, update.
// 4. Wait for any insert to complete.
//
// If started with "bootstrap" and then "growing out of it", remove
// the blocking lines in post-update hook to be able to run the repo.
//
// This job assumes that when allocating the same node and creating
// a new container with the same image, the same workspace is used.
// This is only true if there is a single executor on the node.
//
// Fetch-URI is in the Jenkins settings. Private-URI in the freesitemgr config.

import org.jenkinsci.plugins.workflow.steps.FlowInterruptedException

def fetchURI = "${env.FETCH_URI}"

def freesitemgrdir = '/home/debbiedub/freesitemgr-dgof-jcac'
def mirrors = "${env.MIRRORS_DIR}"

// global variables:
//   docker_image
//   docker_params


def files_list

timestamps {
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
      docker_params = "--network=host --env HOME='${env.WORKSPACE}' -v $mirrors:$mirrors -v $freesitemgrdir:${env.WORKSPACE}/.freesitemgr"
      docker_image = docker.build('pyfreenet:3')
    }
  }


  stage('Get dgof') {
    // Get and/or install dgof into the workspace
    // Commands using dgof will need $(pwd)/dgof to be added to the PATH
    node ('debbies') {
      docker_image.inside(docker_params) {
	timeout(100) {
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
	// Remove empty directories
	// The script for some reason creates empty directories
	// with adding @tmp at the end of the filename.
	sh "rmdir --ignore-fail-on-non-empty $mirrors/*"
	files_list = sh (script: "ls $mirrors", returnStdout: true).trim()
      }
    }
  }
}

// The generated closure is used to perform the work done in the node.
// It is called over and over again util done.
// When it is done, it returns 0 or less.
// Whenever it needs to sleep it returns the amount of seconds to sleep.
// This is run in a way so that the node is released when sleeping
// to allow other legs in the parallel execution to run.
def gen_cl(name, mirrors, fetchURI) {
  boolean preparation_done = false
  boolean cloning_done = false
  boolean upload_done = false

  int preparation_laps = 1
  int cloning_laps = 1
  int upload_laps = 1

  return {
    if (!preparation_done) {
	def lap = preparation_laps++
	echo "$name: Start pre-check lap $lap"
	int result1 = timeout(30) {
	  sh returnStatus: true, script: """freesitemgr --no-insert update $name | tee output.txt
		egrep -v 'No update required|No update desired|site insert has completed|checking if a new insert is needed' < output.txt"""
	}
	if (result1 == 0 &&      // grep found something
	    lap < 5) {
	  echo "$name: Pre-check deferred lap $lap"
	  return 1000 + lap * 22
	}
	echo "$name: Pre-check done lap $lap"
	preparation_done = true
    }

    if (!cloning_done) {
	def lap = cloning_laps++
	echo "$name: Start cloning lap $lap"
	int result2 = 0
	try {
	  result2 = timeout(100) {
	    sh returnStatus: true, script: 'rm -rf ' + "newclone.$name" + ' && PATH="$PATH:$(pwd)/dgof" git clone ' + "freenet::$fetchURI$name/1 newclone.$name" + ' && rm -rf ' + "newclone.$name"
	  }
	} catch (FlowInterruptedException ex) {
	  // We got timeout. Consider it as clone failed.
	  // This will retry some times
	  result2 = -1
	}
	if (result2 != 0) {
	  if (lap < 5) {
	    // Wait before trying again.
	    // The recent cache is 1800s in the default configuration
	    // It is pointless to hit again before that is aged.
	    echo "$name: Cloning failed lap $lap - will retry"
	    return 1800 + lap * 8
	  } else {
	    echo "$name: Cloning failed lap $lap will reinsert"
	  }
	} else {
	  echo "$name: Cloning done lap $lap"
	}
	dir ("$mirrors/$name") {
	  // Get new things from the mirrored repos
	  // Add a file with the used versions of the tools      
	  sh '''git fetch --all && git push freenet &&
	    cd $(git config --get remote.freenet.pushurl || 
		 git config --get remote.freenet.url) &&
	    git --version > v.new &&
	    head -1 `which freesitemgr` | sed 's/^#!//p;d' |
	    sed 's/$/ --version/' | sh >> v.new &&
	    pip3 list >> v.new &&
	    freesitemgr --version >> v.new &&
	    { cmp -s versions v.new && rm v.new || mv v.new versions; }
	  '''
	}
	// This is to handle the problem described in JENKINS-52750
	sh "D=$mirrors/$name@tmp;" + 'if test -d $D; then rmdir $D; fi'

	cloning_done = true
	if (result2 != 0) {
	  timeout(100) {
	    sh "freesitemgr --cron reinsert $name"
	  }
	  unstable "$name: Could not clone the repo. Repo reinserted."
	  // There is no point in doing update immediately after reinsert.
	  return 600
	} else {
	  timeout(100) {
	    sh "freesitemgr --cron update $name"
	  }
	  // For the case where there is no update (most cases), let's proceed
	  // immediately to the check.
	}
    }

    if (!upload_done) {
	def lap = upload_laps++
	echo "$name: Start wait for upload lap $lap"
	int result3 = timeout(30) {
	  sh returnStatus: true, script: """freesitemgr --no-insert update $name | tee output.txt
		egrep -v 'No update required|No update desired|site insert has completed|checking if a new insert is needed' < output.txt"""
	}
	if (result3 == 0) {        // grep found something
	  echo "$name: Wait for upload not completed lap $lap"
	  if (lap < 30) {
	    return 600 + lap * 18
	  }
	  error "$name: Did not complete wait for upload"
	} else {
	  echo "$name: Wait for upload done lap $lap"
	}
	upload_done = true
    }

    return 0
  }
}


def buildParallelMap = [:]
def dirnames = files_list.split("\\r?\\n")
timestamps {
  dirnames.each { dirname ->
    buildParallelMap[dirname] = {
      stage(dirname) {
	def cl = gen_cl(dirname, mirrors, fetchURI)
	def result = 1
	// The closure cl is run using cl() on the node
	// When it cannot do anymore (completed or needs to
	// sleep or wait for something) it returns.
	// If a positive value is returned, it sleeps this many seconds.
	// This allows others to use the node while this closure doesn't.
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
}
