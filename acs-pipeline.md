**initialize demo**
```
git clone https://github.com/rcarrata/devsecops-demo.git
ansible-galaxy collection install community.kubernetes
cd devsecops-demo
pip3 install openshift pyyaml kubernetes --user
./install.sh 
```
**get credentials and urls**
```
./status.sh
```

**Demo Steps**\
**trigger pipeline**\
option-1 \
Switch to Developer > Pipelines > cicd-demo > petclinic-build-dev > Start a pipeline > Dev_Namespace=devsecops_dev > workspace:pvc:petclinic-build-workspace > maven-settings:config-map:maven-settings

option-2\
commit a change in gogs repo

****

https://redhat-scholars.github.io/acs-workshop/acs-workshop/index.html

