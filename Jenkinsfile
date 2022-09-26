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

def map = [
    'app' : '1',
    'browser' : '0',
    'contrib' : '0',
    'fred' : '0',
    'free-chat-2' : '0',
    'flircp' : '0',
    'generate-media-site' : '0',
    'guile-freenet' : '0',
    'Icicle' : '0',
    'infocalypse' : '0',
    'java_installer' : '0',
    'jfniki' : '0',
    'jSite' : '0',
    'kweb-up-poc' : '0',
    'lib-AdaFN' : '0',
    'lib-jfcp' : '0',
    'locutus' : '0',
    'mactray' : '1',
    'N2NChat' : '0',
    'node-wrapper' : '0',
    'plugin-FlogHelper' : '0',
    'plugin-Freemail' : '0',
    'plugin-Freetalk' : '0',
    'plugin-KeepAlive' : '0',
    'plugin-KeyUtils' : '0',
    'plugin-Library' : '0',
    'plugin-Spider' : '0',
    'plugin-sharesite' : '0',
    'plugin-UPnP2' : '0',
    'plugin-UPnP' : '0',
    'plugin-WebOfTrust' : '0',
    'plugins' : '0',
    'pyFreenet' : '0',
    'pyProbe' : '0',
    're-stream-into-freenet' : '0',
    'scripts' : '0',
    'seedrefs' : '0',
    'website' : '0',
    'website-old' : '0',
    'wiki' : '0',
    'wininstaller-innosetup' : '1',
    'wininstaller' : '0',
    'wintray' : '0',
    ]

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

    map.each { entry ->
      stage(entry.key) {
        boolean succeeded = false
        for (int i = 0; i < 5 && !succeeded; i++) {
	  sleep(1 + 20 * i)
	  int result = sh returnStatus: true, script: 'PATH="$PATH:$(pwd)/dgof" git clone ' + "freenet::$fetchURI$entry.key/$entry.value newclone-$entry.key && rm -rf newclone-$entry.key"
	  if (result == 0) {
	    succeeded = true
	  }
	}
        sh "cd $mirrors/$entry.key && git fetch --all && git push freenet"
        if (!succeeded) {
          sh "freesitemgr reinsert $entry.key"
          sh 'exit 1' // The clone failed
        }
      }
    }
  }
}
