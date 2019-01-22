# Deploying Spinnaker

## Spinnaker How to Bdila Guide

### Halyard

    docker pull gcr.io/spinnaker-marketplace/halyard:1.13.1
    docker save -o ./halyard-1-13-1.tar  gcr.io/spinnaker-marketplace/halyard:1.13.1

## Lets get This party started

### Run Halyard

Start Halyard in a new Docker container.
The following command creates the Halyard Docker container, mounting the Halyard config directory:

    $ mkdir ~/.hal
    $
    $ # On Linux machine:
    $ docker run -p 8084:8084 -p 9000:9000 \
        --name halyard --rm \
        -v ~/.hal:/home/spinnaker/.hal \
        -v ${HOME}/.kube/config:/home/spinnaker/.kube/config \
        -d \
        gcr.io/spinnaker-marketplace/halyard:1.13.1
    $
    $ # On Windows machine:
    $ docker run -p 8084:8084 -p 9000:9000 \
    $   --name halyard --rm \
    $   -v /c/Users/Eyal/.hal/:/home/spinnaker/.hal \
    $   -v  /c/Users/Eyal/.kube:/home/spinnaker/.kube \
    $   -d   gcr.io/spinnaker-marketplace/halyard:1.13.1
    $
    $ docker exec -it halyard bash
    $ source <(hal --print-bash-completion)

#### Set Kubernetes provider v2.0

    $ hal config provider kubernetes enable
    $ export CONTEXT=$(kubectl config current-context)

    # Assign spinnaker k8s service account and RBAC roles
    $ kubectl apply -f  /home/spinnaker/.hal/RBAC.yaml

    # Modify the existing kubectl context adding the new service account token
    $ TOKEN=$(kubectl get secret --context $CONTEXT \
        $(kubectl get serviceaccount spinnaker-service-account \
            --context $CONTEXT \
            -n spinnaker \
            -o jsonpath='{.secrets[0].name}') \
        -n spinnaker \
        -o jsonpath='{.data.token}' | base64 --decode)
    $ kubectl config set-credentials ${CONTEXT}-token-user --token $TOKEN

    # Switching to the new service account credentials
    $ kubectl config set-context $CONTEXT --user ${CONTEXT}-token-user

    # Create the new provider account
    $ export ACCOUNT=schef
    $ hal config provider kubernetes account add $ACCOUNT --provider-version v2 --docker-registries [] --service-account true

    $ hal config features edit --artifacts true


    # Spinnaker set distributed install on k8s
    $ hal config deploy edit --type distributed --account-name $ACCOUNT

    # Set s3 bucket as storage backend
    # NOTE: do not supply the value of --secret-access-key on the command line, you will be prompted to enter the value on STDIN once the command has started running
    $ export REGION=us-east-2
    $ export SPIN_S3_BUCKET=my_bucket_uniq_name
    $ export YOUR_SECRET_KEY_ID=bla-bla-bla
    $ hal config storage s3 edit \
        --access-key-id $YOUR_SECRET_KEY_ID \
        --secret-access-key \
        --region ${REGION} \
        --bucket ${SPIN_S3_BUCKET}
    $ hal config storage edit --type s3

    # List Spinnaker Versions
    $ hal version list

    # Configure you desired version
    $ hal config version edit --version 1.11.6

    # Deploy
    $ hal deploy apply

    # Halyard backup config - This includes all secrets you’ve supplied to hal. Keep this safe!
    $ hal backup create

    # Halyard backup restore on another machine - Halyard will expand & replace the existing ~/.hal directory with the backup
    $ hal backup restore --backup-path <backup-name>.tar

