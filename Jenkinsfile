def distributedTasks = [:]

// This object provides a easy place to configure the known branches for your repository
// for instance if you use a develop based trunk then you would update trunk to be be
// "remotes/origin/develop"
@groovy.transform.Field
def BRANCHES = [
  trunk: "remotes/origin/master",
  beta: "remotes/origin/beta",
  next: "remotes/origin/next",
]

stage("Building Distributed Tasks") {
  jsTask {
    checkout scm
    sh 'yarn install'

    distributedTasks << distributed('test', 3)
    distributedTasks << distributed('lint', 3)
    distributedTasks << distributed('build', 3, '--prod')
  }
}

stage("Run Distributed Tasks") {
  parallel distributedTasks
}

def jsTask(Closure cl) {
  node {
    withEnv(["HOME=${workspace}"]) {
      docker.image('node:latest').inside('--tmpfs /.config', cl)
    }
  }
}


def distributed(String target, int bins, String args='') {
  def jobs = splitJobs(target, bins)
  def tasks = [:]

  jobs.eachWithIndex { jobRun, i ->
    def list = jobRun.join(',')
    def title = "${target} - ${i}"

    tasks[title] = {
      jsTask {
        stage(title) {
          checkout scm
          sh 'yarn install'
          sh "npx nx run-many --target=${target} --projects=${list} --parallel ${args}"
        }
      }
    }
  }

  return tasks
}

def splitJobs(String target, int bins) {
  def String baseSha = getBaseRef(env.BRANCH_NAME, env.CHANGE_TARGET, BRANCHES)
  def String raw
  raw = sh(script: "npx nx print-affected --base=${baseSha} --target=${target}", returnStdout: true)
  def data = readJSON(text: raw)

  def tasks = data['tasks'].collect { it['target']['project'] }

  if(tasks.size() == 0){
     return tasks
  }

  // Simple ceiling function as Math.ceil is not allowed by jenkins sandbox (╯°□°）╯︵ ┻━┻
  int c = (tasks.size() + bins -1) / bins
  def split = tasks.collate(c)

  return split
}

 /**
  * determines the baseRef for comparision when running the affected
  * commans. If your repo uses different logic to determine the affected
  * base then you should update this method.
  *
  * @param branchName   The current branch name
  * @param changeTarget The branch where the PR is targeted
  * @param branches     An object with keys for each branch and the reference it should map to
  *
  * @return             A string representing the git ref of the affected base
  */
def getBaseRef(String branchName, String changeTarget, Map<String, String> branches ){

    String baseBranch = changeTarget == null ? branchName : changeTarget
    String affectedBaseRef =  branches.containsKey(baseBranch) ? branches[baseBranch] : branches["trunk"]

    if(!changeTarget){
      affectedBaseRef += "~1"
    }

   return affectedBaseRef

}
