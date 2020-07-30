..
  Technote content.

  See https://developer.lsst.io/restructuredtext/style.html
  for a guide to reStructuredText writing.

  Do not put the title, authors or other metadata in this document;
  those are automatically added.

  Use the following syntax for sections:

  Sections
  ========

  and

  Subsections
  -----------

  and

  Subsubsections
  ^^^^^^^^^^^^^^

  To add images, add the image file (png, svg or jpeg preferred) to the
  _static/ directory. The reST syntax for adding the image is

  .. figure:: /_static/filename.ext
     :name: fig-label

     Caption text.

   Run: ``make html`` and ``open _build/html/index.html`` to preview your work.
   See the README at https://github.com/lsst-sqre/lsst-technote-bootstrap or
   this repo's README for more info.

   Feel free to delete this instructional comment.

:tocdepth: 1

.. Please do not modify tocdepth; will be fixed when a new Sphinx theme is shipped.

.. sectnum::

.. TODO: Delete the note below before merging new content to the master branch.

.. note::

   When deploying the Telescope and Site systems for the main telescope at NCSA we started to experience severe startup slow downs.
   These occurrences affected components both using SalObj and SAL (e.g. CCCamera) and manifested mainly by timeout's in getting historical data followed by the communication being seriously unstable.

   With the deployment of a new cycle (with xml 6.1) we decided to take a more close look at the problem. This tech-note describes the findings, proposed solutions and points of concern for the future deployment.

.. _Introduction:

Introduction
============

Up until early 2020, practically all system deployment, integration and on-sky operations where conducted with the Auxiliary Telescope (AT) components.
Some limited and isolated testing with the Main Telescope (MT) components where performed at the summit and local developers environment.
On these occasions the DDS communication mostly worked as expected, with some occasional failures or unreliable message exchanges occurrences, but nothing that would prevent users from operating the system.

Nevertheless, with the onset of more intensive testing with the MT components (initially at NCSA test stand and then at the base facility) we started to experience routine severe system failures.
These failures would mostly manifest once a sufficiently large number of more intensive components (e.g. MTM1M3 and MTMount) where brought up in the system.
Initially, they where observed mainly by users trying to run control sequences from Jupyter notebooks in nublado, though it was soon realized that the problem affected all other components in the system (salkafka producers, CSCs running SAL and SalObj).

Following, are some details of the history and experience accumulated over time with DDS, results of the investigation conducted at NCSA test stand about this new failure mode and the proposed solution.

.. note::

   These tests occurred in the week of July 20-24, 2020.

.. _Overview:

Overview
========

The test started by using the OpenSplice standard deployment, which employ the DDS :ref:`single process mode <fig-single-process>`.
This operation mode has been used with the Vera Rubin Observatory system from the beginning and, despite some occasional issues, worked relatively well over time.
In fact, most of the time we encountered issues with the DDS communication we could relate them to one of the following issues:

  * Wrong DDS partition.

    The DDS partition is controlled by SAL and SalObj by the ``LSST_DDS_DOMAIN`` system environment variable.
    It was common on some occasions that users would set different values for that variable, which causes DDS to filter the messages and communication would not work.
    The solution is to make sure everyone on the system has the same values for the environment variable.

  * Systems running incompatible versions of xml interface.

    The xml interface determines the DDS topics data structure.
    If topics with the same identity have different data structures the last system to try to connect to the DDS network will fail.
    This created several issues in the past as DDS cache would prevent users from upgrading the topic data structure.
    A mitigation to this problem was introduced in recent versions of SAL where topics identity was attached to a hash code based on its content.
    This allowed users to upgrade topics without colliding with previous cached versions but also meant that incompatible versions silently fail to communicate.

  * Systems running with incompatible versions of SAL/SalObj.

    SAL and SalObj are responsible for managing topics Quality of Service (QoS) parameters.
    The system defines 4 different types of QoS depending on the "type" of topic: Command, Event, Telemetry and Acknowledgement.
    Topics with different QoS may sometimes be filtered out by the DDS reader so, an incompatible version of SAL/SalObj may not be able to exchange some messages.
    This was improved considerably in more recent versions of SAL/SalObj and we have been able to keep breakages to a minimum, especially by rigidly managing deployments.

  * Wrong network attached.

    By default the OpenSplice configuration will automatically select a network to attach to.
    It mainly works by selecting the first suitable network (excluding local and loopback networks for instance) in alphabetical order.
    For servers with more than one "suitable network" it may happen that the selected network is not the correct one.
    In this cases, users must edit the configuration file and explicitly select network attached to where the DDS communication is supposed to happen.

As we started to operate the system with an increased number of components, and especially, components with large sets of topics published at higher frequencies (20 to 50 Hz, as opposed to 1Hz) we started to observe a new failure mode.
This failure would, in principle, happened at random and is mostly characterized by long or timeouts in reading historical data followed by unreliable DDS communication.

It is important to emphasize that reading historical data is a standard DDS procedure and consists of synchronizing the internal DDS read queues with all the other components in the system.
The communication is actually established before this synchronization occurs and it is even possible to receive newly published data before it finishes.
Nevertheless, historical data (for previously published Events) will not be available until the operation is completed.
Note that Command, Telemetry and Acknowledgement topics are configured with disposable QoS durability, which means that no historical data is received.

.. _DDS-Communication-Architecture:

DDS Communication Architecture
==============================

Before describing the tests performed at NCSA, it might be useful to have some understanding of how OpenSplice DDS communication works.
Basically, the communication is broken down into a set of services, that are in charge of a particular aspect of the process.
These services are highly configurable and it is even possible to run the the system with only a subset of them.

These are the `OpenSplice DDS communication services`_:

.. _OpenSplice DDS communication services: http://download.ist.adlinktech.com/docs/Vortex/html/ospl/DeploymentGuide/overview.html#vortex-opensplice-architecture

  * `networking` - providing QoS-driven real-time networking based on multiple reliable multicast channels
  * `durability` - providing fault-tolerant storage for both real-time state data as well as persistent settings
  * `remote control and monitoring SOAP service` - providing remote web-based access using the SOAP protocol from various Vortex OpenSplice tools
  * `dbms service` - providing a connection between the real-time and the enterprise domain by bridging data from DDS to DBMS and vice versa

By default only the `networking` and `durability` services are configured.

Furthermore, OpenSplice DDS can be configured to run in two modes of operation, :ref:`single process <fig-single-process>` or :ref:`shared memory <fig-shared-memory>`.
When running in single process mode, the configured DDS services will run in the same process as the application.
That means, alongside the resources needed by the application itself to run (computing, memory allocation, etc) there will also be additional resources needed by the DDS services themselves.
In addition, there is also the question of how those additional resources are handled by the application and the programming language itself.
For instance, Python does not easily support multiprocessing, so DDS services will be competing for the same computing resources as those needed by the application to run properly.

.. figure:: /_static/SingleProcessFig.png
   :name: fig-single-process
   :target: ../_images/SingleProcessFig.png
   :alt: DDS single process mode

   DDS architecture when running in single process mode.
   In the left hand panel, a diagram showing the internal architecture of a single application.
   The "DDS Application Process" represents the software developed on top of SAL/SalObj to communicate over DDS and execute some operation.
   In the case of Rubin Observatory system this would be a CSC, salkafka producer, user notebook server, etc.
   In the right hand panel, a high level overview of a group of applications (CSCs, salkafka producers, etc).
   Each application runs their own DDS services alongside their processes.

As one can imagine, this architecture will not scale well for large scale systems.
If the DDS communication grows large enough (in terms of number of topics and published frequency) the processing requirements for the DDS services alone may start to interfere with the application and vice-versa.
To overcome this problem, OpenSplice DDS can be configured to run in shared memory mode.

When running in shared memory mode, the user configures and runs a single daemon per node that is dedicated to running the OpenSplice services.
Applications running in the same node can then be configured to attach to the daemon using system shared memory.
In this case, several applications share the same DDS services.
The computing resources needed by the DDS services are separated from those of the application and will not interfere with each other (assuming the node has enough resources available).

.. figure:: /_static/SharedMemoryFig.png
  :name: fig-shared-memory
  :target: ../_images/SharedMemoryFig.png
  :alt: DDS shared memory mode

  DDS architecture when running in shared memory mode.
  In the left hand panel, a diagram showing the internal architecture of several applications (denoted "DDS Application Processes") running concurrently in the same node.
  All applications share the same OpenSplice DDS services through the shared memory space.
  In the right hand panel, a high level overview of a group of applications running in the same node (CSCs, salkafka producers, etc) sharing the OpenSplice DDS services.


.. _NCSA-Standard-Deployment-Test:

NCSA Standard Deployment Test
=============================

This test consisted of trying to reproduce the failure mode observed in multiple occasions at the NCSA test stand when running the MT components.
For that, we wanted to make sure we started with a fresh system and also deployed all the components with the :ref:`standard OpenSplice configuration <Appendix-Standard-OpenSplice-Configuration>`.

All SAL components where brought down so that there was no DDS traffic to start with.
This was verified using the OpenSplice tools, that allow us to listen for any DDS traffic on the network, and with network traffic analysis tools (e.g. tcpdump and etherape).
Since the OpenSplice tools also create DDS traffic, we made sure to switch it off and check the network for traffic again (with tcpdump and etherape) before starting the deployment of the system.

With the system clear, we started deploying the Telescope and Site components one by one.
We started with the CSCs, going from the lighter to heavier components, while leaving the salkafka producers off.
After each component was deployed, we inspected the network topology (the network connections stablished between the components) and traffic.
Furthermore we also verified the integrity of the system by starting remotes for each deployed component and verifying that they could read historical data and that communication was operational.
We verified a steady increase in network activity after each component was brought up and increasing spikes in network traffic during startup.

Once the last component was brought up we observed a spike of around 11Mbps followed by a steady network traffic of around 3.5Mbps, with all components still in ``STANDBY`` state (only publishing heartbeats).
At this point it was verified that communication problems manifest when trying to initialize remotes for the MT components.
:ref:`A network topology diagram <fig-net-topology-full-community>`, showing the established DDS connectivity between the components, was constructed for the system at this state.

.. figure:: /_static/net_topology_full_community.png
   :name: fig-net-topology-full-community
   :target: ../_images/net_topology_full_community.png
   :alt: Network topology for the full system in single process mode

   Network connectivity diagram for the full system running with DDS single process mode at NCSA.
   Each ellipse on the diagram represents a component on the system.
   The text inside the ellipses are the hostname or the IP address of the component, see text and :numref:`table-services` for more information.
   Arrows indicate the network traffic direction.

Note that, although each CSC/service has their own IP address, in fact they are all being deployed as docker containers from two different host nodes.
The deployment strategy uses docker macvlan network driver to give each container it's own IP address, overlaid on a bridge network.
This solution proved to be more reliable than use the host network driver option, commonly used in these situations.
It also proved to be a good strategy when deploying systems on Kubernetes clusters.

.. _table-services:

.. table:: Mapping between hostnames and services on the network.

   +--------------+------------------+-----------------+---------------------------------------+
   | Service type | Hostname         | IP address      | Information                           |
   +==============+==================+=================+=======================================+
   | ospl tool    | lsst-nts-ts3     | 141.142.238.252 |                                       |
   +--------------+------------------+-----------------+---------------------------------------+
   | salkafka     | lsst-l1-cl-tss16 | 141.142.238.115 | - ATHeaderService ATArchiver          |
   |              |                  |                 | - ATMonochromator ATSpectrograph      |
   |              |                  |                 | - Electrometer Test ScriptQueue       |
   |              |                  |                 | - Script ATDome ATDomeTrajectory      |
   |              |                  |                 | - ATAOS ATPneumatics Watcher          |
   |              |                  |                 | - ATHexapod ATPtg ATMCS ATCamera      |
   +--------------+------------------+-----------------+---------------------------------------+
   | ospl daemon  | lsst-l1-cl-tss11 | 141.142.238.110 | Only running in shared memory mode.   |
   +--------------+------------------+-----------------+---------------------------------------+
   | salkafka     | lsst-l1-cl-tss17 | 141.142.238.116 | - CCHeaderService CCArchiver CCCamera |
   +--------------+------------------+-----------------+---------------------------------------+
   | salkafka     | lsst-l1-cl-tss18 | 141.142.238.117 | - MTM1M3                              |
   +--------------+------------------+-----------------+---------------------------------------+
   | salkafka     | lsst-l1-cl-tss19 | 141.142.238.118 | - Rotator Hexapod MTM2 MTAOS          |
   |              |                  |                 | - MTPtg MTMount NewMTMount            |
   +--------------+------------------+-----------------+---------------------------------------+
   | ospl daemon  | lsst-l1-cl-tss30 | 141.142.238.129 | Only running in shared memory mode.   |
   +--------------+------------------+-----------------+---------------------------------------+
   | CSC          | lsst-l1-cl-tss32 | 141.142.238.131 | - MTPtg                               |
   +--------------+------------------+-----------------+---------------------------------------+
   | CSC          | lsst-l1-cl-tss33 | 141.142.238.132 | - Rotator                             |
   +--------------+------------------+-----------------+---------------------------------------+
   | CSC          | lsst-l1-cl-tss34 | 141.142.238.133 | - Camera Hexapod                      |
   +--------------+------------------+-----------------+---------------------------------------+
   | CSC          | lsst-l1-cl-tss35 | 141.142.238.134 | - M2 Hexapod                          |
   +--------------+------------------+-----------------+---------------------------------------+
   | CSC          | lsst-l1-cl-tss36 | 141.142.238.135 | - MTAOS                               |
   +--------------+------------------+-----------------+---------------------------------------+
   | CSC          | lsst-l1-cl-tss37 | 141.142.238.136 | - M1M3                                |
   +--------------+------------------+-----------------+---------------------------------------+
   | CSC          | lsst-l1-cl-tss38 | 141.142.238.137 | - M2                                  |
   +--------------+------------------+-----------------+---------------------------------------+


Although not all components subscribes to each other topics, the :ref:`network topology diagram <fig-net-topology-full-community>` displays an extremely high inter-connectivity between all the different components/hosts.
To begin with, note the IP address ``239.255.0.1`` in the diagram, which is the multicast server and receives traffic from all the components.
A more close look into the diagram show that there are two distinct behavior, related to the two different types of services deployed; CSCs and salkafka producers.

CSCs will send and receive DDS traffic to all other components of the system, whereas salkafka producers will send and receive traffic from CSCs only.
This is more easily seen by filtering all the inbound and outbound traffic to one of the CSCs and to one of the salkafka producers individually.
Diagrams with the traffic to the MT pointing component (MTPtg, hostname ``lsst-l1-cl-tss31``) and to the salkafka producer for the AT components (hostname ``lsst-l1-cl-tss16``) are shown in the left and right hand side of :numref:`fig-net-topology-inset-community`, respectively.

.. figure:: /_static/net_topology_inset_community.png
   :name: fig-net-topology-inset-community
   :target: ../_images/net_topology_inset_community.png
   :alt: filtered network topology to the MT Rotator CSC and to the M1M3 producer

   Filtered network connectivity to the MTPtg CSC (left hand side, hostname ``lsst-l1-cl-tss31``) and to the salkafka producer for AT components (right hand side, hostname ``lsst-l1-cl-tss16``).

One of the reasons we selected the salkafka producer (``lsst-l1-cl-tss16``) for the diagram in :numref:`fig-net-topology-inset-community`, is that it does not subscribe to any of the CSCs running on the system at that time.
Still, the component is receiving DDS traffic for all the CSCs.

At the same time, the MTPtg is a component that no other CSC subscribe to events or telemetry from.
This is one of the main actor component of the system and it operates mainly by sending commands to other CSCs (MTMount and Rotator).
Still, this CSC sends and receives DDS traffic from all the other CSCs, in addition to all salkafka producers in the system.

To understand this behavior we have to look at the how the DDS Communication works when operating in single process mode.
As explained in :ref:`DDS-Communication-Architecture`, each component running in single process mode will also run a full set of DDS services.
By design, these services will be processing not only the data required by the application itself but also in the full topic set present in the system at any moment.
This explains the extremely high inter-connectivity observed in :numref:`fig-net-topology-full-community` and also why upcoming components would lag behind in trying to keep up with the system traffic.

Therefore, following the `OpenSplice deployment guidelines`_ on how to deal with large scale systems, we proceeded to experiment with OpenSplice shared memory mode.

.. _OpenSplice deployment guidelines: https://istkb.adlinktech.com/article/opensplice-dds-v6-offers-choice-two-memory-modes-shared-memory-single-process-i-use/

.. _NCSA-Shared-Memory-Test:

NCSA Shared Memory Test
=======================

The first step in testing shared memory mode was to rebuild all the components using the licensed version of the OpenSplice, since the community edition does not support this mode of operation.
This required some refactoring of the build scripts since the licensed version is stored in a password-protected private repository, and we also want to be able to build both licensed and community editions.
Furthermore, we also have to provide the ospl daemon to run the OpenSplice service.

Since all the services run from a docker containers, the daemon was also deployed on a container.
To make sure the system would work properly we had to enable two additional features when running the containers:

  * ``ipc=host``, to allow containers to access the host machine shared memory space and,
  * ``pid=host``, to allow containers to share the process id with the host machine.

In addition the container must also share a common ``/tmp`` folder with read and write access to all users and groups.
This is required so containers can access a socket and a lock file created by the ospl daemon and needed for the shared memory process to work.

Finally, shared memory is activate by use of a specific :ref:`ospl configuration <Appendix-Shared-Memory-OpenSplice-Configuration>`, which must be the same for the daemons and applications.

Once all the components where updated to the licensed version of OpenSplice and with the new configuration and the daemons where ready, they where deployed into a fresh system (see :ref:`NCSA-Standard-Deployment-Test` for details).
The daemons myst be deployed first, so that the OpenSplice DDS services become available.
Next we brought up all the CSCs, one by one, and salkafka producers following the same process of the :ref:`standard configuration test <NCSA-Standard-Deployment-Test>`.

This time we did not observed issues when trying to bring the producers up after all CSCs where running.
We also did not observed any issues trying to start remotes to communicate with the CSCs.
As expected, the :ref:`network topology now <fig-net-topology-shared-mem>` shows that only the ospl daemons (``lsst-l1-cl-tss11`` and ``lsst-l1-cl-tss30``) are communicating with each other,  since they in charge of the OpenSplice DDS services.

.. figure:: /_static/net_topology_shared_mem.png
   :name: fig-net-topology-shared-mem
   :target: ../_images/net_topology_shared_mem.png
   :alt: Network topology for the full system in shared memory mode

   Network topology for the full system in shared memory mode.
   Even though all components are running, only the OpenSplice daemon communicates with each other over the network.

Overall the preliminary results indicate that using the OpenSplice shared memory mode solves the issues we observed previously with a large scale system.

.. _Conclusions-and-Future-Work:

Conclusions and Future Work
===========================

From the tests executed at NCSA and described in previous sections, it is clear that the Vera Rubin Observatory Control system has grown large enough to be treated as a large scale system.
The immediate impact of this growth is that we are no longer able to run the system reliably with the standard OpenSplice DDS configuration.
The most immediate solution is to switch to shared memory mode, which is the recommended configuration for large scale system.

This new operation mode impacts considerably how we deploy the system.
To begin with, we now must now build all the system with a licensed version of OpenSplice, which required them to be privately hosted, rather than publicly accessible (over docker hub).
Furthermore, we must configure all systems to be able to connect to an ospl daemon, that is in charge of running the OpenSplice DDS services.
This will affect not only the CSCs Telescope and Site deploy in Kubernetes clusters but also, stand-alone CSCs deployed in embedded systems, nublado notebook servers, Camera, DM CSCs and so forth.

Finally, once a suitable number of systems are brought to a running state with this new configuration, we must also make sure to test it for longer periods of time and make sure they can be operated reliably.

Appendix
========

.. _Appendix-Standard-OpenSplice-Configuration:

Standard OpenSplice Configuration
---------------------------------

.. literalinclude:: ospl.xml

.. _Appendix-Shared-Memory-OpenSplice-Configuration:

Shared Memory OpenSplice Configuration
--------------------------------------

.. literalinclude:: ospl-6.10.4-shmem.xml

.. .. rubric:: References

.. Make in-text citations with: :cite:`bibkey`.

.. .. bibliography:: local.bib lsstbib/books.bib lsstbib/lsst.bib lsstbib/lsst-dm.bib lsstbib/refs.bib lsstbib/refs_ads.bib
..    :style: lsst_aa
