Demo Microservice Architecture
==============================

Overview
********
``demo-svc`` is a Microservice Architecture implementation which is currently formed of the following
services and components: 

* `hawkbit-fota <https://github.com/jonathanyhliang/hawkbit-fota>`_
  
  ``hawkbit-fota`` service consists of a frontend server which provides RESTful APIs to manage images,
  distributions, and deployments to be deployed to ``hawkbit-fota`` clients. And a backend server which
  implements `Hawkbit DDI <https://www.eclipse.org/hawkbit/apis/ddi_api/>`_ compliant RESTful APIs
  so that ``hawkbit-fota`` clients could poll and launch FOTA processes.

* `slcan-svc <https://github.com/jonathanyhliang/slcan-svc>`_
  
  ``slcan-svc`` Bridging serial-line CAN communication via RESTful APIs

* `mcumgr-svc <https://github.com/jonathanyhliang/mcumgr-svc>`_

  ``mcumgr-svc`` implements a ``hawkbit-fota`` client as the frontend to retrive FOTA deployments from
  ``hawkbit-fota`` service backend by polling and launching FOTA processes. The backend of the service
    
* `rabbitmq <https://www.rabbitmq.com/kubernetes/operator/quickstart-operator.html>`_
  
  Since both ``slcan-svc`` and ``mcumgr-svc`` deal with the same serial port, an inter-service measure is
  required to make a sequencial transitioning of the interface from one to the other. ``rabbitmq`` is utilised
  here so that when ``slcan-svc`` has released the serial port and put the ``slcan`` device into firmware
  update mode, ``mcumgr-svc`` is informed to launch the update process.

.. image:: docs/plantuml/diag/demo-svc.svg

Part of the work to demonstrate this Microservice Architecture is to develop an ``slcan`` device and a
``hawkbit-fota`` client device to interact with the Microservice. A fork of
`zephyrproject-rtos/zephyr <https://github.com/jonathanyhliang/zephyr>`_ has been added with an
`slcan <https://github.com/jonathanyhliang/zephyr/tree/slcan/samples/subsys/canbus/slcan>`_ sample application
which runs on `ST Nucleo F446RE <https://docs.zephyrproject.org/latest/boards/arm/nucleo_f446re/doc/index.html>`_
board, and the existing `Hawkbit FOTA client <https://github.com/jonathanyhliang/zephyr/tree/cc32xx-hawkbit-bringup/samples/subsys/mgmt/hawkbit>`_
sample application has been patched to support `CC3220SF LaunchXL <https://docs.zephyrproject.org/latest/boards/arm/cc3220sf_launchxl/doc/index.html>`_
WIFI board bring-up.

Prerequisute
************
* Raspberry Pi arm64
* `minikube arm64 <https://minikube.sigs.k8s.io/docs/start/>`_
* socat
* An `SLCAN device <https://github.com/jonathanyhliang/zephyr/tree/slcan/samples/subsys/canbus/slcan>`_


Building and Running
********************
Start a socat server process which listens to tcp:80 and links to the serial port of ``slcan`` device.
(/dev/ttyACM0 in this example)

.. code-block:: console

    sudo socat tcp-l:80,reuseaddr,fork file:/dev/ttyACM0,nonblock,waitlock=/var/run/ttyACM0.lock &

Enable the Ingress controller:

.. code-block:: console

    minikube addons enable ingress

Install the RabbitMQ Cluster Operator:

.. code-block:: console

    kubectl apply -f "https://github.com/rabbitmq/cluster-operator/releases/latest/download/cluster-operator.yml"

Create a RabbitMQ Cluster. It could take serveral minutes for RabbitMQ Cluster to get ready.

.. code-block:: console

    kubectl apply -f demo-svc-rabbitmq.yaml

Create a ``hawkbit`` deployment. This deploys a pod that ``hawkbit-svc`` is running on,
and a servie that expsoe the frontend and backend ports. ``mcrmgr-svc`` depends on the
backend port of ``hawkbit-svc`` exposed.

.. code-block:: console

    kubectl apply -f demo-svc-hawkbit.yaml

Create an ``slcan`` deployment. This deploys an ``slcan-svc`` and an ``mcumgr-svc``

.. code-block:: console

    kubectl apply -f demo-svc-slcan.yaml

Wait until both ``slcan-svc`` and ``mcumgr-svc`` are in running state

References
**********
* `Set up Ingress on Minikube with the NGINX Ingress Controller <https://kubernetes.io/docs/tasks/access-application-cluster/ingress-minikube/>`_
* `RabbitMQ Cluster Kubernetes Operator Quickstart <https://www.rabbitmq.com/kubernetes/operator/quickstart-operator.html>`_
