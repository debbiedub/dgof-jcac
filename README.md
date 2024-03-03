# dgof-jcac

Jenkinsfile and other files needed to mirror repos from github to
freenet/hyphanet.

## Required Jenkins settings and modules

Environment variables:

FETCH_URI - The URI, ending with /, used to fetch the
repos. freesitemgr has the private key.

MIRROR_DIR - The directory with the mirrored repos.

Environment Injector (envinject)
