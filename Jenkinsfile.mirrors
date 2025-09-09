// The common preparation
// 1. Install pyFreenet3 (could be prepared in the docker image used)
// 2. "install" dgof     (could be prepared in the docker image used)
//
// The logic for every repo outside of Jenkins:
// 1. In the MIRRORS_DIR:
//      git clone --mirror URL name
//    name is specified to not get the the .git suffix
//    This will create a remote named origin and that is expected by
//    the script.
// 2. dgof-setup --as-maintainer --bootstrap name (after doing cd name)
//    Add the private URI that matches the FETCH_URI of the jenkins
//    configuration and is suffixed with name
//    If it fails because of the cannot pack into small enough files,
//    remove the dgof directory and redo without --bootstrap and the
//    repo will not be available without dgof
// 3. Deactivate the default update for the freenet remote for the mirrored
//    repo:
//      git config remote.freenet.skipDefaultUpdate true
// 4. Remove, in the dgof/name/hooks directory,
//    the freesitemgr update command and the echos after that from
//    the post-update hook and
//    the pre-receive hook
// 5. Make the first freesitemgr update
// 6. Set up MIRROR_DIR/.name.config as a config dir for freesitemgr
//    for that repo.
//    This is probably easiest done by creating the dir and copying
//    the .config file from another such dir.
// 7. Move the freesitemgr file from ~/.freesitemgr to the
//    MIRROR_DIR/.name.config directory.
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
// Fetch-URI is in the Jenkins settings. Private-URI in the freesitemgr config.

import org.jenkinsci.plugins.workflow.steps.FlowInterruptedException

def fetchURI = "${env.FETCH_URI}"

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
  RUN pip3 install cmarkgfm
  RUN pip3 install git+http://localhost:8888/$(curl http://localhost:8888/freenet:USK@nrDOd1piehaN7z7s~~IYwH-2eK7gcQ9wAtPMxD8xPEs,y61pkcoRy-ccB7BHvLCzt3RUjeMILf8ox26NKvPZ-jk,AQACAAE/dgof/0/ | sed '/Permanent/s/.*freenet://;s/".*//;s/@/%40/')
      '''
      docker_image = docker.build('dgof:3', '--network=host .')
      docker_params = "--network=host -v $mirrors:$mirrors"
    }
  }


  stage('List') {
    node ('debbies') {
      docker_image.inside(docker_params) {
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
  boolean fetching_done = false
  boolean upload_done = false

  int preparation_laps = 1
  int cloning_laps = 1
  int fetching_laps = 1
  int upload_laps = 1

  return {
    def freesitemgrdir = "$mirrors/.${name}.config"
    if (!preparation_done) {
	def lap = preparation_laps++
	echo "$name: Start pre-check lap $lap"
	int result1 = timeout(30) {
	  sh returnStatus: true, script: 'HOME=$(pwd) ' + """freesitemgr --no-insert --config-dir $freesitemgrdir update $name | tee output.txt
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

    int result2 = 0
    if (!cloning_done) {
	def lap = cloning_laps++
	echo "$name: Start cloning lap $lap"
	try {
	  result2 = timeout(100) {
	    sh returnStatus: true, script: 'rm -rf ' + "newclone.$name" + ' && HOME=$(pwd) git clone ' + "freenet::$fetchURI$name/1 newclone.$name" + ' && rm -rf ' + "newclone.$name"
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
	cloning_done = true
    }

    if (!fetching_done) {
	dir ("$mirrors/$name") {
	  def laps = fetching_laps++
	  // Get new things from the mirrored repos
	  // Add a file with the used versions of the tools      
	  int result3 = sh returnStatus: laps < 5, script: '''git fetch --all &&
	    HOME=$(pwd) dgof-copy-readme --description="Mirror of $(git remote get-url origin)" freenet &&
            HOME=$(pwd) git push freenet &&
	    cd $(git config --get remote.freenet.pushurl || 
		 git config --get remote.freenet.url) &&
	    git --version > v.new &&
	    head -1 `which freesitemgr` | sed 's/^#!//p;d' |
	    sed 's/$/ --version/' | sh >> v.new &&
	    pip3 list >> v.new &&
	    freesitemgr --version >> v.new &&
	    { cmp -s versions v.new && rm v.new || mv v.new versions; }
	  '''
	  if (result3 != 0) {
	    if (lap < 5) {
	      // Wait before trying again.
	      echo "$name: Fetching failed lap $lap - will retry"
              return 100 + 1000 * laps
	    }
	  }
	}
	fetching_done = true

	// This is to handle the problem described in JENKINS-52750
	sh "D=$mirrors/$name@tmp;" + 'if test -d $D; then rmdir $D; fi'

	if (result2 != 0) {
	  timeout(100) {
	    sh 'HOME=$(pwd) ' + "freesitemgr --cron --config-dir $freesitemgrdir reinsert $name"
	  }
	  unstable "$name: Could not clone the repo. Repo reinserted."
	  // There is no point in doing update immediately after reinsert.
	  return 600
	} else {
	  timeout(100) {
	    sh 'HOME=$(pwd) ' + "freesitemgr --cron --config-dir $freesitemgrdir update $name"
	  }
	  // For the case where there is no update (most cases), let's proceed
	  // immediately to the check.
	}
    }

    if (!upload_done) {
	def lap = upload_laps++
	echo "$name: Start wait for upload lap $lap"
	int result4 = timeout(30) {
	  sh returnStatus: true, script: 'HOME=$(pwd) ' + """freesitemgr --no-insert --config-dir $freesitemgrdir update $name | tee output.txt
		egrep -v 'No update required|No update desired|site insert has completed|checking if a new insert is needed' < output.txt"""
	}
	if (result4 == 0) {        // grep found something
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
