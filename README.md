# How to submit a job to HTCondor from Kubernetes

A special container that can submit HTC jobs must be used: https://gitlab.cern.ch/tbareiki/condorsubmit.
A built container image from gitlab-registry will be used: gitlab-registry.cern.ch/tbareiki/condorsubmit

## Gitlab credentials

By default, Kubernetes will not be able to pull the image from gitlab, so a secret has to be created:

```
kubectl create secret docker-registry gitlab-registry \
    --docker-server=gitlab-registry.cern.ch \
    --docker-username=tbareiki \
    --docker-password=<ACCESS-TOKEN>
```

The ACCESS-TOKEN can be retrieved from gitlab [access-tokens](https://gitlab.cern.ch/profile/personal_access_tokens)
page with the **read_registry** ticked on. Also, the `--docker-username` has te be modified.

## A pod that submits a job

An example pod that submits a job to HTCondor.

```yaml
  #############
 ## SECRETS ##
#############
---
apiVersion: v1
kind: Secret
metadata:
  name: kinit-secret
type: Opaque
data:
  username: <USERNAME-BASE64>
  password: <PASSWORD-BASE64>
---
apiVersion: v1
kind: Secret
metadata:
  name: step1-sub
type: Opaque
data:
  job.sub: ZXhlY3V0YWJsZSAgICAgICAgICAgICAgPSBqb2Iuc2gKb3V0cHV0ICAgICAgICAgICAgICAgICAgPSBvdXRwdXQKZXJyb3IgICAgICAgICAgICAgICAgICAgPSBlcnJvcgpsb2cgICAgICAgICAgICAgICAgICAgICA9IGxvZwoKcXVldWUgMQo=
  job.sh: IyEvYmluL2Jhc2gKCmVjaG8gIkhlbGxvIgo=
---
  ##########
 ## JOBS ##
##########
---
apiVersion: batch/v1
kind: Job
metadata:
  name: step1
spec:
  ttlSecondsAfterFinished: 20
  template:
    spec:
      containers:
      - name: step1
        image: gitlab-registry.cern.ch/tbareiki/condorsubmit
        command: ["sh", "-c"]
        args: ["
          export CONDOR_USER=`cat /mnt/kinit/username` &&
          echo $CONDOR_USER &&
          useradd -Ms /bin/bash $CONDOR_USER &&
          mkdir -m 777 /scratch &&
          runuser -l $CONDOR_USER -c '
            cat /mnt/kinit/password| kinit &&
            cd /scratch &&
            cp /mnt/step1/..data/* . &&
            condor_submit -spool job.sub' &&
          sleep 999999999
          "]
        volumeMounts:
          - mountPath: /mnt/kinit
            name: kinit-secret-vol
          - mountPath: /mnt/step1
            name: step1-sub-vol
      restartPolicy: Never
      imagePullSecrets:
      - name: gitlab-registry
      volumes:
      - name: kinit-secret-vol
        secret:
          secretName: kinit-secret
      - name: step1-sub-vol
        secret:
          secretName: step1-sub
```

The image needs the submission file and an accompanying shell script, both of which are stored in Secret `step1-sub` endoded with base64.
Also, in order to communicate to HTCondor `kinit` needs to be executed, to which the password is supplied from Secret `kinit-secret`.

The previously made `gitlab-registry` Secret is used in

```yaml
      imagePullSecrets:
      - name: gitlab-registry
```

Also, the commands executed in the container are important

```yaml
        command: ["sh", "-c"]
        args: ["
          export CONDOR_USER=`cat /mnt/kinit/username` &&
          echo $CONDOR_USER &&
          useradd -Ms /bin/bash $CONDOR_USER && # your CERN user has to be created inside the container
          mkdir -m 777 /scratch && # since the volume of secrets is read only,
                                   # the job.sh and job.sub files are copied toa separate directory
          runuser -l $CONDOR_USER -c ' # the following is executed as your CERN user
            cat /mnt/kinit/password| kinit && # kinit supplied with the password
            cd /scratch &&
            cp /mnt/step1/..data/* . &&
            condor_submit -spool job.sub' && # The spool option is a must
          sleep 999999999 # so that the pod doesn't die after job submittion
          "]
```

After creating this pod we can see job that was submitted with `condor_q`. To inspect the HTCondor output logs we can exec into the pod and look for them in the `/scratch` directory.
