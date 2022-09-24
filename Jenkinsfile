// The logic for every site should be
// 1. Install pyFreenet3 (could be prepared in the docker image used)
// 2. "install" dgof     (could be prepared in the docker image used)
// 3. clone and throw the clone away
// 4. pull from github
// 5. Push to freenet
// 6. If the clone didn't work, reinsert. The reinsert will abort if pushing.
//
// Fetch-URI in the script. Private-URI in the freesitemgr config.

def map = [
    'newsite16' : 'USK@UTbgOlck8yonG-PBz3a0IAr2bo4OANpQIS5l7ujQzDg,Uw4kLUKyu5C~zFnGnYXfB9MsutJCgJBg8Cvf9~0GagI,AQACAAE/newsite16/1/',
    'newsite17ma' : 'USK@tMV0iY48lEGBOUtztCYemQ~Go7vye-2ZCs9coKGrpKA,aJqVO3DrFKzym8dBaeIv9C27B7Pk7soWYIkJhqVwPDg,AQACAAE/newsite17ma/1/',
    'newsite17mb' : 'USK@7WGX4ivgcShNlmvabhDDG4SklxGQKZqCkRW5s9Vlx1A,pLI1YJZidELVh6fZweFqI0XgpsNXTnDQZK6DaL~Gll0,AQACAAE/newsite17mb/1/',
    ]

node (label: debbies) {
  deleteDir()
  docker.image('python:3').inside('--network=host') {
    stage('Get dgof') {
      sh '''
	if git clone http://localhost:8888/freenet:USK@nrDOd1piehaN7z7s~~IYwH-2eK7gcQ9wAtPMxD8xPEs,y61pkcoRy-ccB7BHvLCzt3RUjeMILf8ox26NKvPZ-jk,AQACAAE/dgof/22/ dgof 2> gitclone.out
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
        sh 'PATH="$PATH:$(pwd)/dgof" git clone ' + "freenet::$entry.value $entry.key"
      }
    }
  }
}
