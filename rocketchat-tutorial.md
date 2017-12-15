# Introduction

In the "Up and running with the OpenShift Ansible Broker" blog (by Jesus Rodriguez), we saw how you can leverage the OpenShift Ansible Broker to easily provision services on OpenShift Container Platform. In this blog, we're going to explore developing an Ansible Playbook Bundle (APB) to manage services.

APBs leverage the power of Ansible to allow you to define your application in the same language you would define your infrastructure. Most commonly, APBs are used to orchestrate pre-certified containers while also using Ansible to provision on and off-platform services while also allowing you to package your application management logic in a container image. With APBs you can define how to install, and uninstall, your application with the flexibility provided by Ansible.

This tutorial will give a user a walk-through on developing an APB to deploy the Rocket.Chat service. To simplify this process and focus on just the APB development, we're going to assume that Rocket.Chat and MongoDB are already containerized and can run under the `restricted` Security Context Constraint (SCC) in OpenShift. These container images can be found on RHCC([RocketChat](https://access.redhat.com/containers/?tab=overview#/registry.connect.redhat.com/rocketchat/rocketchat) and [MongoDB](https://access.redhat.com/containers/?tab=overview#/registry.access.redhat.com/rhscl/mongodb-32-rhel7)).

If you're only interested in deploying the published [RocketChat APB](https://access.redhat.com/containers/?tab=overview#/registry.connect.redhat.com/rocketchat/rocketchat-apb) you can skip to the end of this tutorial.

# Creating RocketChat APB
To start we will [install the APB tooling](https://github.com/ansibleplaybookbundle/ansible-playbook-bundle/blob/master/docs/apb_cli.md#installing-the-apb-tool). Then run:
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

| file/directory                     | description                                                   |
|------------------------------------|---------------------------------------------------------------|
| `apb.yml`                          | The APB spec declaration                                      |
| `Dockerfile`                       | The APBs Dockerfile                                           |
| `playbooks/provision.yml`          | An Ansible Playbook defining the APBs `provision` action      |
| `playbooks/deprovision.yml`        | An Ansible Playbook defining the APBs `deprovision` action    |
| `roles/provision-rocketchat-apb`   | An Ansible Role defining which tasks are run on `provision`   |
| `roles/deprovision-rocketchat-apb` | An Ansible Role defining which tasks are run on `deprovision` |

## APB Spec
`apb.yml` is the declaration of the APB spec. In here, we will list all relevant application specific information including: OpenShift Service Catalog metadata, all of the APBs `plans`, and input parameters to prompt the user when provisioning. Looking at apb.yml, we are free to edit the default plan that is created by the tooling. Go ahead and edit the file to match below.
```yaml
version: 1.0
name: rocketchat-apb
description: This APB deploys RocketChat backed by MongoDB
bindable: False
async: optional
metadata: 
  documentationUrl: https://rocket.chat
  imageUrl: https://github.com/RocketChat/Rocket.Chat.Artwork/blob/master/Logos/rocketcat.png?raw=true
  dependencies: ['registry.connect.redhat.com/rocketchat/rocketchat:latest', 'registry.access.redhat.com/rhscl/mongodb-32-rhel7']
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
```
* For information on configuring metadata for your application, see [here](https://github.com/ansibleplaybookbundle/ansible-playbook-bundle/blob/master/docs/developers.md#metadata)

Note the `parameters` section, our Rocket.Chat container image will expect these parameters as environment variables to be used and configured on startup. Now that we have defined our APBs parameters, we can reference them as environment variables to be injected into the container from the `deploymentConfig`.

## Provisioning
First, we need to edit the provision role for the APB. `apb init` makes this easy by leaving us commented blocks of code that we can work from.

We want three standard resources to be created by our APB: a `service`, a `deploymentConfig`, and a `route`. Start by opening up `roles/provision-rocketchat-apb/tasks/main.yml` and uncommenting the `deploymentConfig` resource so necessary information for attaching a `persistentVolume` to the Rocket.Chat deployment. Once you're finished editing, your `deploymentConfig` should look like this:

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
      image: registry.connect.redhat.com/rocketchat/rocketchat
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
      image: registry.access.redhat.com/rhscl/mongodb-32-rhel7
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

In the `deploymentConfig`, you'll see the environment variables expected by Rocket.Chat have been set to the parameter fields defined in `apb.yml`. We also added a `persistentVolumeClaim` to the deploymentConfig named `mongo-storage`. This means we will need to create a `persistentVolumeClaim` resource as well.

The only value we haven't set yet is the `ROOT_URL` environment variable. To set this, we need to get the fully qualified route of where the application will exist on OpenShift. Let's uncomment the `route` and `service` resources that were generated and move the `route` before the `deploymentConfig` resource so we can store and use it. Your provision role should now look like this:

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
        value: "{{ route.route.spec.host }}"
      - name: MONGO_URL
        value: "mongodb://0.0.0.0:27017/{{ mongodb_name }}"
      image: registry.connect.redhat.com/rocketchat/rocketchat
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
      image: registry.access.redhat.com/rhscl/mongodb-32-rhel7
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

Note that we used `register` when we created the route to store the output of the `route` creation. We then use this output as the value for the `ROOT_URL` environment variable as `{{ route.route.spec.host }}`. We also created two individual service declarations for Mongo and RocketChat for each container in the pod.

## Deprovisioning
By default we recommend that an APB author provide a basic deprovision role. This allows a service consumer to delete the service from the OpenShift Service Catalog and the APBs corresponding resources to be properly deleted. `apb init` provides a skeleton deprovision role with commented resources just like the provision role. Uncomment the `service`, `route`, and `deploymentconfig` resources in the deprovision task like so:

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
This will remove all created resources in the APBs namespace.

## Building and Testing
Now that our APB is ready to be tested, we can build and push the image to be deployed by the OpenShift Ansible Broker. For this tutorial, I am assuming your Broker is configured to source APBs from the internal OpenShift registry (Please [refer to this guide](https://github.com/openshift/ansible-service-broker/blob/master/docs/config.md#local-openshift-registry) to configure the OpenShift Ansible Broker with the `local_openshift` registry adapter).

To push your image into the OpenShift registry, type:
```
apb push --openshift
```

This command will automatically tag & build your image, push it to the internal OpenShift registry, bootstrap the Broker, and relist the Service Catalog. Refreshing the Service Catalog will display your new APB in the console. You can now deploy your service by clicking on it and filling out the respective parameters to deploy Rocket.Chat.

# Troubleshooting

## APB not displaying in the Service Catalog
If you do not see your APB in the OpenShift web console after `apb push --openshift`, a good way to debug is to type:
```
apb list
```

Examine the output and check if your APB is listed. If it is, that means the OpenShift Ansible Broker knows about your APB and is part of it's list of bootstrapped APB specs. If it doesn't show up then try running:
```
apb bootstrap
```

If the bootstrap is successful and you now see your APB but don't see it in the Service Catalog then try:
```
apb relist
```

This will trigger the Service Catalog to get the full list of bootstrapped APB specs.

# Configuring OpenShift Ansible Broker to deploy RocketChat APB from RHCC
In order to use images published by ISVs (in this instance [rocketchat-apb](https://access.redhat.com/containers/?tab=overview#/registry.connect.redhat.com/rocketchat/rocketchat-apb)), we need to append the OpenShift Ansible Broker's registry configuration with the `openshift` registry adapter pointing to `registry.connect.redhat.com`. To do this, please make sure that your OpenShift Ansible Broker's configuration has the following:
```yaml
registry:
  - name: isv-registry
    type: openshift
    user: <RH_user>
    pass: <RH_pass>
    url: http://registry.connect.redhat.com
    images:
      - rocketchat/rocketchat-apb
    white_list:
      - ".*-apb$"
```

`<RH_user>` and `<RH_pass>` should be replaced with your credentials to authenticate against RHCC. The important thing to note is the `images` portion of the config. We have to manually define the images that we want to see from the ISV registry. In this case, we're adding `rocketchat/rocketchat-apb`.

If you already have an OpenShift cluster up, you can access the OpenShift Ansible Broker's configuration by typing (in the broker's namespace):
```
oc edit configmap broker-config
```

Saving your changes should deploy a newer pod for the Broker. Once the pod is ready, you can type:
```
apb relist
```

You should now be able to see and deploy the RocketChat APB from the OpenShift Service Catalog.

# Conclusion
I hope this helps developers see how easy it is to take complex applications and deploy them using APBs. This tutorial showed you how to create a RocketChat APB from scratch using Red Hat signed images and also showed you how to configure the OpenShift Ansible Broker to deploy the Red Hat signed RocketChat APB from the Red Hat Container Catalog.
