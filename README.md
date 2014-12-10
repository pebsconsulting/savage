Savage
======
[![Build Status](https://travis-ci.org/twbs/savage.svg?branch=master)](https://travis-ci.org/twbs/savage)

Savage is a service watches for new or updated pull requests on a given GitHub repository. For each pull request, it evaluates whether the changes are "safe" (i.e. we can run a Travis CI build with them with heightened permissions without worrying about security issues) and "interesting" (i.e. would benefit from a Travis CI build with them with heightened permissions), based on which files were modified. If the pull request is "safe" and "interesting", then it initiates a Travis CI build with heightened permissions on a specified GitHub repository. When the Travis CI build completes, it posts a comment ([like this one](https://github.com/twbs/bootstrap/pull/15178#issuecomment-63756231)) with the test results on the pull request. If the test failed, the pull requester can then revise their code to fix the problem.
Users who are members of trusted GitHub organizations (see the `trusted-orgs` setting) can ask Savage to retry a pull request by leaving a comment on the pull request of the form: "@\<username-of-savage-bot> retry" (e.g. "@twbs-savage retry")

Savage's original use-case is for running Sauce Labs cross-browser JS tests on pull requests via Travis CI, while keeping the Sauce Labs access credentials private & secure.

Affectionately named after an experimenter known for "busting" misconceptions, often with explosives.

## Motivation
(Savage is general enough to be used in other situations, but the following is the specific one it was built for.)

You're a member of a popular open source project that involves front-end Web technologies. Cool.

Specifically, the project involves JavaScript. Because it's a serious project, you have automated cross-browser testing for your JavaScript. You happen to use [Open Sauce](https://saucelabs.com/opensauce) for this.

Unfortunately, [due to certain limitations](http://support.saucelabs.com/entries/25614798-How-can-we-set-up-an-open-source-account-that-runs-tests-on-people-s-pull-requests-), it's not possible to do cross-browser testing on pull requests "the obvious way" via Travis CI without potentially compromising your Sauce login credentials. This means that either (a) cross-browser problems aren't discovered in pull requests until after they've already been merged (b) repo collaborators must manually initiate the cross-browser tests on pull requests (and manage the resulting branches, and possibly post comments communicating the test results).

By automating the process of initiating Travis-based Sauce tests and posting the results, cross-browser JavaScript issues can be discovered more quickly and with less work on the part of repo collaborators.

## How it works (for the Open Sauce use-case)
1. Use GitHub webhooks to listen for new or updated pull requests in a given GitHub repository.
2. If the pull request does not modify any JavaScript files, ignore it.
3. Ensure that no sensitive build files (e.g. `.travis.yml`, `Gruntfile.js`) have been modified, since these files have the potential to cause leakage/exposure of the Sauce login credentials.
4. Clone the pull request's branch and push it to a test repo under an autogenerated name.
5. Travis CI will automatically run a build on the new branch *under the test repo's user*. Thus, this build will have access to Travis secure environment variables; in particular, it will have access to the Sauce Labs credentials.
6. Use webhooks to track the status of the Travis build.
7. When the build finishes, post a comment to the GitHub pull request explaining the test results, and delete the corresponding branch.

## Used by
* [Bootstrap](https://github.com/twbs/bootstrap); see [@twbs-savage](https://github.com/twbs-savage)

## DISCLAIMER
The current authors are not security experts and this project has not been subjected to a third-party security audit.

## Usage
Using Savage involves two GitHub repos (which can both be the same repo, although that's much less secure):
* The *main repo*
  * This repo is the one receiving pull requests
  * Savage needs its GitHub web hook set up for this repo
  * Savage does NOT need to be a Collaborator on this repo
* The *test repo*
  * The repo that Savage will push test branches to
  * Travis CI should be set up for this repo
  * Savage needs to be a Collaborator on this repo, so that it can push branches to it and also delete branches from it

Java 7+, [Git](http://git-scm.com/), OpenSSH, and a Unix-like OS are required to run Savage. For instructions on building Savage yourself, see [the Contributing docs](https://github.com/twbs/savage/blob/master/CONTRIBUTING.md).

For step-by-step setup instructions, see [`SETUP.md`](https://github.com/twbs/savage/blob/master/SETUP.md).

Savage accepts exactly one optional command-line argument, which is the port number to run its HTTP server on, e.g. `8080`. If you don't provide this argument, the default port specified in `application.conf` will be used. Once you've built the JAR, run e.g. `java -jar savage-assembly-1.0.jar 8080` (replace `8080` with whatever port number you want). Note that running on ports <= 1024 requires root privileges (not recommended) or using port mapping.

When running Savage, its working directory needs to be a non-bare git repo which is a clone of the repo being monitored.

The Unix user that Savage runs as needs to have an SSH key setup, and that SSH key needs to be registered in Savage's GitHub user account, so that Savage can pull-to/push-from GitHub securely via SSH. GitHub's public key also needs to be present in the `known_hosts` of Savage's Unix user.

If you're using Sauce, we recommend [using a sub-account](https://saucelabs.com/sub-accounts) for Savage, to completely prevent any possibility of compromise of your main account.

Other settings live in `application.conf`. In addition to the normal Akka and Spray settings, Savage offers the following settings:
```
savage {
    // Port to run on, if not specified via the command line
    default-port = 6060
    // Full name of GitHub repo to watch for new pull requests
    github-repo-to-watch = "twbs/bootstrap"
    // Full name of GitHub repo to push test branches to
    github-test-repo = "twbs/bootstrap-tests"
    // Ignore pull requests whose branch is from the watched repo (and is thus from a project team member)
    ignore-branches-from-watched-repo = true
    // List of GitHub organization names whose users Savage should trust to authorize retries of builds
    trusted-orgs = [ "twbs" ]
    // List of Unix file globs constituting the whitelist of safely editable files
    whitelist = [
        "**.md",
        "/bower.json",
        "/composer.json",
        "/fonts/**.{eot,ttf,svg,woff}",
        "/less/**.less",
        "/sass/**.{sass,scss}",
        "/js/**.{js,html,css}",
        "/dist/**.{css,js,map,eot,ttf,svg,woff}",
        "/docs/**.{html,css,js,map,png,ico,xml,eot,ttf,svg,woff,swf}"
    ]
    // List of Unix file globs constituting the watchlist of files
    //   which trigger a Savage build.
    // To prevent unnecessary builds, a Savage build isn't triggered
    // unless the pull request affects a file that matches one of the watchlist globs.
    file-watchlist = [
        "/js/**/*.js"
    ]
    // Prefix to use for branches that Savage pushes to the main repository.
    // The branch name is generated by prefixing the pull request number with this prefix.
    branch-prefix = "savage-"
    // GitHub login credentials for the Savage bot to use
    username = throwaway9475947
    password = XXXXXXXX
    // This goes in the "Secret" field when setting up the Webhook
    // in the "Webhooks & Services" part of your repo's Settings.
    // This string will be converted to UTF-8 for the HMAC-SHA1 computation.
    // The HMAC is used to verify that Savage is really being contacted by GitHub,
    // and not by some random hacker.
    github-web-hook-secret-key = abcdefg
    // Used as a shared secret in a hashing scheme that's used to verify
    // that Savage is really being contacted by Travis CI,
    // and not by some random hacker. For how to find your Travis token,
    // see http://docs.travis-ci.com/user/notifications/#Authorization-for-Webhooks
    travis-token = abcdefg
}
```

### GitHub webhook configuration

* Payload URL: `http://your-domain.example/savage/github`
* Content type: `application/json`
* Secret: Same as your `web-hook-secret-key` config value
* Which events would you like to trigger this webhook?: "Pull Request" and "Issue comment"

### Travis webhook configuration
In `.travis.yml`:
```
notifications:
  webhooks:
    - http://your-domain.example/savage/travis
```

## Acknowledgments
We all stand on the shoulders of giants and get by with a little help from our friends. Savage is written in [Scala](http://www.scala-lang.org) and built on top of:
* [Akka](http://akka.io) & [Spray](http://spray.io), for async processing & HTTP
* [Eclipse EGit GitHub library](https://github.com/eclipse/egit-github), for working with [the GitHub API](https://developer.github.com/v3/)

## See also
* [LMVTFY](https://github.com/cvrebert/lmvtfy), Savage's sister bot who does HTML validation
* [Rorschach](https://github.com/twbs/rorschach), Savage's sister bot who sanity-checks Bootstrap pull requests
