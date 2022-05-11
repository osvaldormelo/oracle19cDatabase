# Oracle19c Database using Podman and Openshift
This repository shows how to build a image and deploy an Oracle database in Openshift(for developers purpusing only)
# Instructions

1. Pull down the GitHub Repo from Oracle 
    This repo from an oracle can be used to create multiple different services.
    ```bash
    git clone https://github.com/osvaldormelo/oracle19cDatabase.git
    ```
2. Change into the Database Dockerfiles 
    ```bash
    cd oracle19cDatabase/OracleDatabase/SingleInstance/dockerfiles
    ```
3. Pull down Binaries from Oracle 
    Go to https://www.oracle.com/database/technologies/oracle-database-software-downloads.html and pull down the binaries. In this example, we will be utilizing 19c with the Linux 64 zip file.

    An oracle account will need to be created for free if one is not already created.
    The size of the download is roughly 2.9gb insize. Add the zip folder into: oracle19cDatabase/OracleDatabase/SingleInstance/dockerfiles/19.3.0

    This repo will be utilizing 19.3.0 as the version.

    The version represents the specific Oracle Database Image being created. This repo will be using 19.3

    ![](/images/2022-05-11-09-58-35.png)
4. Utilize Bash to create Oracle Image
    First, make sure that the binary is set to be executable.
    ```bash
    chmod +x buildContainerImage.sh
    ``` 
    
    Next, execute the bash script to build the image:
    
    ```bash
    ./buildContainerImage.sh -e -v 19.3.0 -t oracle19c:1.0.0 -o '--build-arg SLIMMING=false'
    ```
    
    e: builds with an Enterprise Edition

    v: Specifies the Oracle Database Version

    -o with build-arg SLIMMING=false: Adds in the additional features for patching and etc.

5. Setting up the Oracle Database on Run
    To test the image out that we just created we can run it locally with docker. Of course, you can also do this with Podman as well. If you do not want to test the image then you can move on to the Kubernetes portion. However, it is recommended to test the container locally before pushing it up.

    Run the Oracle Database with:

    ```bash
    podman run --name oracle-19c -p 127.0.0.1::1521 -p 127.0.0.1::5500 -e ORACLE_SID=ABC -e ORACLE_PDB=ABCPDB1 -e ORACLE_PWD=GoFor1t! oracle19c:1.0.0
    ```
    This Oracle Database uses Pluggable databases. In that case, there are few environment variables that need to be identified.

    ORACLE_SID: Is the SID for the Database

    ORACLE_PDB: Is the Pluggable Database also used as the service to connect with a user.

    ORACLE_PWD: This is of course the password.

6. Changing the Password after Creation
    After the database server is set up the password can be changed. This can be done by running:
    
    ```bash
    podman exec <container name> ./setPassword.sh <your-password>
    ```

7. Pushing the Image to a Repository
    Now push to any image repository like dockerhub or a private image repository. Like this:
    
    Login into your private registry:
    
    ```bash
    podman login <your registry name> --tls-verify=false
    ```
    Tag your local image to match with your registry:

    ```bash
    podman tag <your image id> <your registry name>/<your project name>/oracle19c:1.0.0
    ```

    Push your image on private repository:

    ```bash
    podman push <your registry name>/<your project name>/oracle19c:1.0.0 --tls-verify=false
    ```


8. Create the Password Secret Resource
    This is just a simple base64 password. Not truly secure but at least it will not be in the deployment configurations. Create a YAML file called oracle-db-secret.yml and add the following:

    ```
    apiVersion: v1
    kind: Secret
    type: Opaque
    metadata:
     name: oracledb-secret
     namespace: <your-namespace>
    data:
     oracleRootPassword: V29ybGQyayE=
    ```
    
    Great! Now apply the file by running:

    ```bash
    oc apply -f oracle-db-secret.yml -n <your namespace>
    ```

9. Create the Deployment Resource for Kubernetes
    Next, let’s create the deployment resource that we will utilize to start up the image we created and provision a new oracle database.

    Create a database-deployment.yml file and add the following:

    ```
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: oracle-19c
      namespace: <your-namespace>
    spec:
      replicas: 1
      selector:
        matchLabels:
          app: oracle-19c
      template:
        metadata:
          labels:
            app: oracle-19c
            selector: oracle-19c
        spec:
          containers:
          - name: oracle-19c
            image: <your image creation location>
            env:
            - name: ORACLE_SID
              value: <your-sid>
            - name: ORACLE_PDB
              value: <your-portable-db-name>
            - name: ORACLE_EDITION
              value: enterprise
            - name: ORACLE_PWD
              valueFrom:
                secretKeyRef:
                  name: oracledb-secret
                  key: oracleRootPassword
            resources:
              limits:
                cpu: 1
                memory: 2500Mi
              requests:
                cpu: 1
                memory: 2000Mi
            ports:
            - name: main-port
              containerPort: 1521
            - name: em-port
              containerPort: 5500
    ```

10. Now apply by running:

    ```bash
    cd yaml-files
    ```

    Create a service account for Oracle19c:
    
    ```bash
    oc create sa oracle-sa
    ```

    Add policy to service account created:

    ```bash
    oc adm policy add-scc-to-user anyuid -z oracle-sa
    ```

    Create Deployment for Database:

    ```bash
    oc apply -f database-deployment.yml -n <your namespace>
    ```
    If you want, you can apply service account to deployment manually:
    
    ```bash
    oc set serviceaccount deployment/oracle-19c oracle-sa
    ```
    Now finnaly, you will create a service to deployment with following command:

    ```bash
    oc apply -f service.yml -n <your namespace>
    ```

Great. You have now created your first Oracle 19c database in Openshift. 

![](/images/2022-05-11-16-51-09.png)

Stay tuned as ill either append to this repo or write another one on Oracle 19c persistence with Openshift!

