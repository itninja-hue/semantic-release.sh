# semantic-release.sh
Shell script that automates software versioning, change-log creation, tagging and releasing on GitHub. Implements: [semVer](https://semver.org/), [Conventional Commits](https://www.conventionalcommits.org/en/v1.0.0-beta.4/).

## Getting Started

### Installing
```bash
curl https://raw.githubusercontent.com/itninja-hue/semantic-release.sh/master/semantic-release.sh --output semantic-release.sh
```
### Usage
```bash
chmod +x semantic-release.sh 
./semantic-release.sh
```

## How does it work?
* [semantic-release.sh](https://github.com/itninja-hue/semantic-release.sh) analyzes commit messages that implements [Conventional Commits](https://www.conventionalcommits.org/en/v1.0.0-beta.4/), recongnized by a regex expression supllied to the script by default in **REGEX_CONVENTIONAL_COMMITS** variable in [**---- RULES ----**] section, (all non conform commits messages will be ignored), and decides how to bump the version following 3 regex (major,minor,patch) expression set in **[REGEX_MAJOR,REGEX_MINOR,REGEX_PATCH]** variables in [**---- RULES ----**] section.

* [semantic-release.sh](https://github.com/itninja-hue/semantic-release.sh) supports 2 types of releases (**pre-release**,**release**) , as a release is released only on your main branch, by default set to master in **RELEASE_BRANCH** variable in [**---- RULES ----**] section, and a pre-release is released from any non-main branch. In an indeal workflow a release should only be emited from your main branch (example:master) and your pre-release branch (example: pr/rc).

* [semantic-release.sh](https://github.com/itninja-hue/semantic-release.sh) generates and push a changelog.md file based on the analyzed commit messages. The changelog.md commit is set by default to **"chore(release): ${NEXT_VERSION} [skip ci]"**
in **RELEASE_COMMIT_COMMENT** variable in [**---- CREATE/UPDATE CHANGELOG.md ----**] section, that way you in your pipeline you can skip the pipeline trigger by testing **[skip ci]** in the commit message.

## Rules(configs) summary
|Vaiable|Usage|
|-------|--------|
|REGEX_CONVENTIONAL_COMMITS|Regex expression to filter and analyze commit messages|
|REGEX_MAJOR|Regex expression that filter commit messages for major version release bump|
|REGEX_MINOR|Regex expression that filter commit messages for minor version release bump|
|REGEX_PATCH|Regex expression that filter commit messages for patch version release bump|
|RELEASE_BRANCH|Branch name on wich to release a non pre-release|
|RELEASE_COMMIT_COMMENT|CHANGELOG.md commit message|

**Note:** you can edit the rules as you see fit.

## Example in pipeline (Jenkins)
```groovy
#!groovy

def lastCommitInfo = ""
def skippingText = ""
def commitContainsSkip = 0
def shouldBuild = false
def tag = ""
String[] releaseBranches = ["master","pre/rc"]
pipeline {
    agent any
    stages{
        stage('init') {
            steps{
                script{
                    lastCommitInfo = sh(script: "git log -1 --pretty=medium", returnStdout: true).trim()
                    commitContainsSkip = sh(script: "git log -1 | grep '.*\\[skip ci\\].*'", returnStatus: true)
                    if(commitContainsSkip == 0) {
                        skippingText = "Skipping commit."
                        env.shouldBuild = false
                        currentBuild.result = "NOT_BUILT"
                    }
                }
            }
        }
        stage('semantic-release.sh'){
            when{
                expression{
                    return env.shouldBuild != "false"
                }
            }
            steps{
                script{
                    if(releaseBranches.contains(env.BRANCH_NAME)){
                        withCredentials([string(credentialsId: 'GH_TOKEN', variable: 'GH_TOKEN')]){
                            sh './semantic-release.sh'
                        }
                    }
                }
            }     
        }
    }   
}
```