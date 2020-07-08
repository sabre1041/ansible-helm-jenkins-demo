ansible-helm-jenkins-demo
==========================

Demonstrating the integration between Jenkins, Ansible Tower and Helm to streamline application delivery 

## Configuration

Use following sections details how to prepare the demonstration environment

### Ansible Tower

1. Create a new project called `tower` for Ansible Tower

```
$ oc new-project tower
```

2. Install and deploy Ansible Tower on OpenShift according to the [product documentation](https://docs.ansible.com/ansible-tower/latest/html/administration/openshift_configuration.html).


#### Ansible Tower OpenShift Runner

The execution of Tower jobs for Helm integration will occur as part of a [Container Group](https://docs.ansible.com/ansible-tower/latest/html/administration/external_execution_envs.html#ag-container-groups). A custom image will be built extending the default `ansible-runner`. Since the `ansible-runner` image is located in the Red Hat Container Catalog, you must provide a [pull secret](https://docs.openshift.com/container-platform/4.4/openshift_images/managing_images/using-image-pull-secrets.html) in order to retrieve the image. 

With the pull secret created, process the _ansible-openshift-runner_ template substituting the name of the pull secret created previously:

```
$ oc process ansible/tower/ansible-openshift-runner-build.yaml -p APPLICATION_NAME=ansible-openshift-runner -p PULL_SECRET=<pull_secret> -n tower | oc apply -f-
```

Build the _ansible-openshift-runner-build_ image

```
$ oc start-build -n tower ansible-openshift-runner --follow
```


#### Service Account

Create a service account in the `tower` project to be used as part of the Container Group to create container instances running jobs as well as the Service Account to run the container.

```
$ oc create sa -n tower helm-tower-runner
```

Grant the service account elevated permissions:

```
$ oc adm policy add-cluster-role-to-user cluster-admin -n tower -z helm-tower-runner
```

NOTE: In the future, customized policies should be created in order to limit the permissions granted to the _helm-tower-runner_ Service Account

Obtain the OAuth token for the only created _helm-tower-runner_ service account.

```
$ oc -n tower serviceaccounts get-token helm-tower-runner
```

This value will be used in subsequent sections

#### Configuring Ansible Tower

Using the admin credentials specified during the installation, login to the Ansible Tower web interface using the route created in the `tower` project

1. Create a new _Credential_ to be able to launch containers as part of an instance group
    1. Select **Credentials** on the left hand navigation bar
    2. Hit the **+** sign
    3. Enter a name for the credential and then under _Credential Type_, select **OpenShift or Kubernetes API Bearer Token**
    4. Enter `https://kubernetes.default.svc` for the OpenShift API endpoint
    5. Enter the OAuth token from the service account obtained in the prior section
    6. Uncheck _Verify SSL_
    7. Hit Save
2. Create a new Container Group
    1. Select **Instance Groups** on the left hand navigation bar
    2. Click the **+** button and then select **Create Container Group**
    3. Enter a name for the container group
    4. Select the credential previously created
    5. Check the _Customize Pod Spec_ option
    6. Locate the image reference of the OpenShift Runner previously created
    ```
    $ oc get istag ansible-openshift-runner:latest -o jsonpath={.image.dockerImageReference}
    ```
    7. Specify the Pod Spec that customizes the Service Account used to execute the Pod and the Image Reference obtained previously
    ```
    apiVersion: v1
    kind: Pod
    metadata:
      namespace: tower
    spec:
      containers:
        - image: <openshift_runner_image_reference>
        tty: true
        stdin: true
        imagePullPolicy: Always
        args:
            - sleep
            - infinity
      serviceAccount: helm-tower-runner
    ```
    8. Click Save
3. Create a new Project to retrieve assets from this repository
    1. Select **Projects** on the left hand navigation bar
    2. Hit the **+** button
    3. Enter a name for the project and select **Git** as the SCM type
    4. Enter the URL for this repository into the _SCM  URL_ field
    5. Click **Save**
4. Create a new Job Template to execute included playbook
    1. Select **Templates** on the left hand navigation bar
    2. Click the **+** button and select **Job Template**
    3. Enter a name for the job
    4. Select **Run** as the _Job Type_
    5. Select an inventory (The included _Demo Inventory_ that is provide by default will suffice)
    6. Select the project previously created
    7. Under _Instance Groups_, select the Container Group previously created
    8. Under _Extra Variables_, select **Prompt on Launch**
    9. Click **Save**

At this point, Ansible Tower has been configured

### Jenkins

Jenkins will be used to initiate the demo and communicate with Ansible Tower to trigger the deployment of the Helm Chart.

1. Create a new project called `jenkins`

```
$ oc new-project jenkins
```

2. Instantiate the `jenkins-persistent` template

```
$ oc process -n openshift -p MEMORY_LIMIT=1536Mi jenkins-persistent | oc -n jenkins apply -f-
```

3. Login to Jenkins once the container is running and active using the created route
4. Install and update the required plugins
    1. Click on **Manage Jenkins**
    2. Select **Manage Plugins**
    3. Under the _Updates_ tab, select all plugins related to Git (Resolves issues experienced during testing)
    4. Click **Download Now And Restart After Install**
    5. When Jenkins restarts, login again and return to the **Manage Plugins** page
    6. Select the _Available_ tab, search and select **Ansible Tower**
    7. Click **Download Now And Restart After Install**
5. Add a new credential to communicate with Ansible Tower
    1. Select credentials on the left hand side
    2. Select **System** under _Credentials_ on the lefthand size
    3. Select **Global Credentials** and then click **Add Credentials**
    4. Under _Kind_, ensure _Username with password_ is selected and enter the username and password for accessing Ansible Tower
    5. Click OK
6. Configure Ansible Tower Plugin
    1. From the Jenkins home screen, select **Manage Jenkins** and then **Configure System**
    2. Enter a name to represent the tower installation
    3. Enter the URL for tower
    4. Select the credential previously created in the _Credentials_ section
    5. Check the **Force Trust Cert** checkbox
    6. Click Save

At this point, Jenkins has been setup and configured successfully

### Jenkins pipeline build

A _BuildConfig_ representing a Jenkins Pipeline build is available in an included [template](jenkins/buildconfig.yaml).

Instantiate the template in the `jenkins` namespace by providing two parameters:

1. Reference to the name of the tower instance configured in Jenkins
2. Name of the Job Template

```
$ oc process -p TOWER_INSTANCE=<jenkins_tower_instance_name> -p JOB_TEMPLATE_NAME=<tower_job_template_name> -f jenkins/buildconfig.yaml | oc -n jenkins apply -f-
```

A new job will now appear in Jenkins for which the demonstration can be executed.