This tutorial will give a user a walkthrough on developing an APB for RocketChat. In order to do this we must first make some assumptions. We are assuming that RocketChat and MongoDB are already containerized applications which can run under the `restricted` scc in OpenShift. These images exist on Dockerhub([RocketChat](https://hub.docker.com/_/rocket.chat) and [MongoDB](https://hub.docker.com/_/mongo)).

# Creating RocketChat APB
To get started we will need to install the APB tooling. Please see the [installation guide](https://github.com/ansibleplaybookbundle/ansible-playbook-bundle/blob/master/docs/apb_cli.md#installing-the-apb-tool) to install the APB tooling. Once installed, we will run:
```
apb init rocketchat-apb
```

At this point you will see the following file structure:
```
rocketchat-apb/
├── apb.yml
├── Dockerfile
├── playbooks
│   ├── deprovision.yml
│   └── provision.yml
└── roles
    ├── deprovision-rocketchat-apb
    │   └── tasks
    │       └── main.yml
    └── provision-rocketchat-apb
        └── tasks
            └── main.yml
```

## APB Spec
Looking at apb.yml, we are free to edit the default plan that is created by the tooling. Go ahead and edit the file to match below.
```yaml
version: 1.0
name: rocketchat-apb
description: This APB deploys RocketChat backed by MongoDB
bindable: False
async: optional
metadata: 
  documentationUrl: https://rocket.chat
  imageUrl: https://github.com/RocketChat/Rocket.Chat.Artwork/blob/master/Logos/rocketcat.png?raw=true
  dependencies: ['rocket.chat:latest', 'mongo:3.2']
  displayName: RocketChat (APB)
  longDescription: An APB that deploys RocketChat to OpenShift backed by MongoDB
plans:
  - name: default
    description: This plan deploys a single RocketChat application backed by MongoDB
    free: True
    metadata:
      displayName: Default
      longDescription: This plan provides a RocketChat application backed by MongoDB
      cost: $0.00
    parameters: 
      - name: mongodb_user
        default: rocketchat
        type: string
        title: MongoDB Username
        required: True
      - name: mongodb_pass
        default: changeme
        type: string
        title: MongoDB Password
        required: True
      - name: mongodb_name
        default: rocketchat
        type: string
        title: MongoDB Database Name
        required: True
      - name: mongodb_admin_pass
        default: changeme
        type: string
        title: MongoDB Admin Password
        required: True
      - name: mongodb_version
        default: "3.2"
        enum: ['3.2', '3.4', '3.5']
        type: enum
        title: MongoDB Version
        required: True
      - name: rocketchat_version
        default: latest
        enum: ['latest', '0.56.0']
```
* For information on configuring metadata for your application, see [here](https://github.com/ansibleplaybookbundle/ansible-playbook-bundle/blob/master/docs/developers.md#metadata)

The important thing to note here are the `parameters`. Our rocketchat container image will expect these parameters as environment variables to be used and configured on startup. Now that we have defined our APBs parameters, we can reference them as environment variables to be injected into the container from the `deploymentConfig`. This will be made obvious in the next section.

## Provisioning
The first thing we want to do is to edit the provision role for the APB. Thankfully `apb init` makes this easy by leaving us commented blocks of code we can simply uncomment. We want to uncomment the three standard resources that should be created by an APB. A `service`, a `deploymentConfig`, and a `route`. The first thing we will edit is the `deploymentConfig`. Open up `roles/provision-rocketchat-apb/tasks/main.yml` and uncomment the `deploymentConfig` resource and add the needed information to attach a `persistentVolume` to the rocketchat deployment. Your `deploymentConfig` should look like the following:

```yaml
- name: create deployment config
  openshift_v1_deployment_config:
    name: rocketchat
    namespace: '{{ namespace }}'
    labels:
      app: rocketchat
      service: rocketchat
    replicas: 1
    selector:
      app: rocketchat
      service: rocketchat
    spec_template_metadata_labels:
      app: rocketchat
      service: rocketchat
    containers:
    - env:
      - name: ROOT_URL
      - name: MONGO_URL
        value: "mongodb://0.0.0.0:27017/{{ mongodb_name }}"
      image: rocket.chat:{{ rocketchat_version }}
      name: rocketchat
      ports:
      - container_port: 3000
        name: rocketchat-3000
        protocol: TCP
    - env:
      - name: MONGODB_USER
        value: "{{ mongodb_user }}"
      - name: MONGODB_PASSWORD
        value: "{{ mongodb_pass }}"
      - name: MONGODB_DATABASE
        value: "{{ mongodb_name }}"
      - name: MONGODB_ADMIN_PASSWORD
        value: "{{ mongodb_admin_pass }}"
      image: mongo:{{ mongodb_version }}
      volume_mounts:
      - mount_path: /data/db
        name: mongo-storage
      name: db
      ports:
      - container_port: 27017
        protocol: TCP
        name: mongo-27017
    volumes:
    - name: mongo-storage
      persistent_volume_claim:
        claim_name: mongo-storage
```

In the `deploymentConfig` you'll see that we have set the value of the environment variables that the app container expects to the parameter fields that we defined in `apb.yml`. We also added a `persistentVolumeClaim` resource to the deploymentConfig named `mongo-storage`. This means in the next section we need to be sure we create a `persistentVolumeClaim` along with our other resources. The only value we have not set yet is the `ROOT_URL` environment variable. To set this, we need to get the fully qualified route of where the application will exist on OpenShift. To get this, lets uncomment the `route` and `service` resources that were generated and move them before the `deploymentConfig` resource so that we can store the route and use it. This will also include creating a `service` resource.  Your provision role should now look like:

```yaml
- name: create rocketchat route
  openshift_v1_route:
    name: rocketchat
    namespace: '{{ namespace }}'
    spec_port_target_port: rocketchat-3000
    labels:
      app: rocketchat
      service: rocketchat
    to_name: rocketchat
  register: route

- name: create persistent volume claim for mongo
  k8s_v1_persistent_volume_claim:
    name: mongo-storage
    namespace: '{{ namespace }}'
    state: present
    access_modes:
      - ReadWriteOnce
    resources_requests:
      storage: 1Gi

- name: create deployment config
  openshift_v1_deployment_config:
    name: rocketchat
    namespace: '{{ namespace }}'
    labels:
      app: rocketchat
      service: rocketchat
    replicas: 1
    selector:
      app: rocketchat
      service: rocketchat
    spec_template_metadata_labels:
      app: rocketchat
      service: rocketchat
    containers:
    - env:
      - name: ROOT_URL
        value: "http://{{ route.route.spec.host }}"
      - name: MONGO_URL
        value: "mongodb://0.0.0.0:27017/{{ mongodb_name }}"
      image: rocket.chat:{{ rocketchat_version }}
      name: rocketchat
      ports:
      - container_port: 3000
        name: rocketchat-3000
        protocol: TCP
    - env:
      - name: MONGODB_USER
        value: "{{ mongodb_user }}"

- name: create rocketchat service
  k8s_v1_service:
    name: rocketchat
    namespace: '{{ namespace }}'
    labels:
      app: rocketchat
      service: rocketchat
    selector:
      app: rocketchat
      service: rocketchat
    ports:
      - name: rocketchat-3000
        port: 3000
        target_port: 3000

- name: create mongo service
  k8s_v1_service:
    name: db
    namespace: '{{ namespace }}'
    labels:
      app: rocketchat
      service: db
    selector:
      app: rocketchat
      service: db
    ports:
      - name: mongo-27017
        port: 27017
        target_port: 27017
```

What's important to note here is that we used the `register` portion of the Ansible module when we created the route to store the output of the `route` creation. This output is then used as the value for the `ROOT_URL` environment variable as `{{ route.route.spec.host }}`. We also created two individual service declarations for Mongo and RocketChat for each container in the pod.

## Deprovisioning
By default we recommend that an APB author provides a basic deprovisioning role. This way a user can delete the service from the WebUI and all of the APBs corresponding resources are properly deleted. `apb init` provides a skeleton deprovision role with commented resources just like the provision role. Uncomment the `service`, `route`, and `deploymentconfig` resources in the deprovision task like so:

```yaml
- openshift_v1_route:
    name: rocketchat
    namespace: '{{ namespace }}'
    state: absent

- k8s_v1_service:
    name: rocketchat
    namespace: '{{ namespace }}'
    state: absent

  k8s_v1_persistent_volume_claim:
    name: mongo-storage
    namespace: '{{ namespace }}'
    state: absent

- openshift_v1_deployment_config:
    name: rocketchat
    namespace: '{{ namespace }}'
    state: absent
```
This will properly delete all created resources in the APBs namespace.

## Building and Testing
Now that our APB is ready to be tested, we can build and push the image so that it can be deployed by the OpenShift Ansible Broker. For this tutorial I am assuming your Ansible Broker is configured to source APBs from the internal OpenShift Registry. Please see [here](https://github.com/openshift/ansible-service-broker/blob/master/docs/config.md#local-openshift-registry) to configure the broker with the `local_openshift` registry adapter.

To push your image onto the OpenShift registry, type:
```
apb push --openshift
```

This command will automatically tag & build your image and push it to the internal OpenShift registry. It will then bootstrap the broker and relist the service catalog. Refreshing the webUI will display your new APB in the console. You can now deploy your application by clicking on it and filling out the respective parameters to deploy rocketchat.
