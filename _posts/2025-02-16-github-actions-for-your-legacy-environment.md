---
layout: post
title:  "GitHub Actions for Your Legacy Environment"
categories: devops
---

I started at a new company in 2023 and while it was a great opportunity to take several steps further in my career, it also was a step back in terms of automation. When I first started, there were two servers that ran everything. One server running airflow for orchestration and another, runner server, for executing jobs. These two servers were in the same datacenter and were pretty locked down. The initial design, or lack there of, ran under a single users account linked through Active Directory. This was not ideal as I couldn’t get access to the production environment and every single deployment had to go through a single person's account. 

From day one, I wanted to remove this obstruction and automate code deployments. I needed to be able to deploy changes without the owner. The owner is human. The owner gets sick. The owner doesn’t check their Teams messages sometimes. At my last org, we used GitHub actions. I tried to bring that to the IT team as a tool we could use and hit a roadblock. It wouldn’t let GH through to the servers in our on prem datacenter. I was aware of [Jenkins](https://www.jenkins.io) and that it runs a pull model rather than a push and started the process of getting that approved. That process is long though and I needed to do something now.

My first attempted fix was really bad. We used airflow as a CICD manager. Really. It was bad. For existing projects, the dag would have its first node run a bash command  ssh to the runner server, change directory to the target project, and git pull.


```
from airflow import DAG
from airflow.operators.bash import BashOperator
from datetime import datetime
default_args = {
    'owner': 'airflow',
    'start_date': datetime(2020, 2, 15),
}
with DAG(
    'ssh_git_pull_dag',
    default_args=default_args,
    schedule_interval=None,
    catchup=False,
) as dag:
    git_pull_task = BashOperator(
        task_id='git_pull_task',
        bash_command="""
            ssh your_server_alias << 'EOF'
                cd /path/to/your/project
                git pull origin main
            EOF
        """,
    )
```


This sucked. Firstly, it was a lot to set up. At the time of implementation I was managing over a hundred dags tied to twenty distinct repos. It was a pain to track where a repo would get pulled since I didn’t do it every DAG related to that project. It technically worked, but I don’t even think it saved time considering all the setup and maintenance it required. The only upside? I could fix problems without the account owner.

Enter: The actual solution. Jenkins was still in IT approval hell. Airflow was sort of kind of working as an option but still needed ongoing maintenance and no one but me was familiar with the dags. I then read about [self hosted runners](https://docs.github.com/en/actions/hosting-your-own-runners/managing-self-hosted-runners/about-self-hosted-runners)Self-hosted runners solved everything. They could run inside the data center, listen for GitHub triggers, and execute arbitrary code—without IT approval. Best of all, I could get started without IT approval. And IT probably wouldn’t even notice!

In tandem with this project, I had requested more “runner” servers to segregate our qa, uat, prod, and dev environments. To begin (without IT support) I co-opted the dev environment runner server as an infrastructure server. On dev, I created a different runner for each repo I wanted to automate. Ideally, I could make use of shared GitHub runner servers across all of our repos but that requires git hub enterprise admin credentials and I was still doing this on the down-low.

Each runner could be created by following [this](https://docs.github.com/en/actions/hosting-your-own-runners/managing-self-hosted-runners/adding-self-hosted-runners) with each action runner named after the repo it was going to be tied to. I also made sure to setup each runner as a service using the [link](https://docs.github.com/en/actions/hosting-your-own-runners/managing-self-hosted-runners/configuring-the-self-hosted-runner-application-as-a-service) found in the docs.
Once the runners were live, setting up the YAML workflows was easy. The workflow my team wanted was deploy on merge and the fastest way to get started is the relatively simple workflow of on successful merge to target branch, ssh to the target runner server and git pull.


```
name: deploy
on:
  pull_request:
    branches:
      - main
    types:
      - closed
jobs:
  deploy:
    if: github.event.pull_request.merged == true
    runs-on: self-hosted
    steps:
      - name: deploy via ssh
        run: |
          ssh  <user>@<ip>
            cd <project_dir>
            git pull
```


Note that there was some setup with this… Notably, this expects the target server to be a known host and for a suitable ssh key to be in ./ssh

But that was it! It worked and now my teams work isn’t delayed until “deployment Fridays” or worse, delayed for another week because one person was out of office. This obviously has a ton of room to improve pending buy in from the management team. Firstly, we should really only be using one runner shared by all of the repos. We can scale that up as needed but first need to get the GH admin team to enable [shared runner groups](https://docs.github.com/en/actions/hosting-your-own-runners/managing-self-hosted-runners/managing-access-to-self-hosted-runners-using-groups) Next, we should package our deployments in some way. I would love to move my team to using docker containers for everything but we are a ways off for that. In the interim, I have proposed cloning the repo from scratch into a new dir and then creating a symlink to keep the last few versions deployed.


```
name: deploy
on:
  push:
    branches:
      - main
jobs:
  deploy:
    runs-on: self-hosted
    steps:
      - name: deploy via ssh
        env:
          RELEASE_DIR: "/path/to/your/app/releases"
          TIMESTAMP: ${{ github.run_id }}  # or use `date +%Y%m%d%H%M%S` for timestamp
        run: |
          ssh -o <user>@<ip> << 'EOF'
            set -e
            mkdir -p $RELEASE_DIR/$TIMESTAMP
            git clone https://github.com/<your-repo>.git $RELEASE_DIR/$TIMESTAMP
            cd $RELEASE_DIR/$TIMESTAMP
            ln -sfn $RELEASE_DIR/$TIMESTAMP /path/to/your/app/current
            cd $RELEASE_DIR
            ls -1tr | head -n -5 | xargs -d '\n' rm -rf --
          EOF
```


Automating CI/CD in a locked-down legacy environment wasn’t easy, but here is what I learned:

### 1. Don’t Wait for IT Approval If You Don’t Have To.

* Self-hosted GitHub runners let me bypass bureaucracy.

### 2. Start Small and Iterate.

* Airflow was bad, but it got me to the next step.

### 3. Self-Hosted Runners Are a Great Workaround for Restricted Environments.

* They let me automate deployments without opening firewall rules.


If you’re stuck in a similar situation, try this. It’s not perfect, but it’s so much better than waiting for a human bottleneck.

