# notes on docker the magic container store

* [what the fuck is docker in the field of vitrualization](https://en.wikipedia.org/wiki/Docker_(software))?
	* an [open-source project](https://en.wikipedia.org/wiki/Open-source_software) that automates the deployment of applications inside software containers
		* guarantees that it will always run the same regardless of the environment it is running in
	* allows independent containers to run within a single linux instance avoiding the overhead of starting and maintaining virtual machines
		* additional layer of abstraction and automation in operating-system-level virtualization
			* isolation features of the linux kernel
				* cggroups
					* provide the ability to limit resources
						* cpu
						* memory
						* block I/O
						* network
				* kernel namespaces
					* isolates an application's view of the operating environment
						* process trees
						* network
						* user IDs
						* mounted file systems
			* union-capable file system
	* `libcontainer`
		* a library to directly use virtualization facilities provided by the linux kernel
	* in addition uses abstracted virtualization interfaces
		* libvirt
			* open source api, daemon and management tool for managing platform virtualization
		* LXC
			* an operating-system-level virtualization method for running multiple isolated linux containers on a control host using a single linux kernel
		* systemd-nspawn
			* fully virtualizes 
				* file system hierarchy 
				* process tree
				* various IPC subsystems
	* overview
		* as actions are done to a Docker base image
			* union file system layers are created and documented
				* each layer fully describes how to recreate an action
			* this enables Docker's lightweight images
				* only layer updates need to be propagated
		* Docker basically implements a high-level api to provide lightweight containers that run processes in isolation
			* each container can be constrained to only use a defined amount of resources

* what is a [union-capable file system in the field of virtualization](https://en.wikipedia.org/wiki/Union_mount)?
	* a way of combining multiple directories into one that appears to contain their combined contents
		* supported
			* Linux
			* BSD (and successors)
			* Plan 9
				* similar but subtly different behavior
	* Linux
		* multiple iterations of implementations	
			* inheriting file system - 1993
				* abandoned by its developler because of its complexity
			* UnionFS
				* FiST project at Stony Brook University
			* aufs - 2006
				* attempt to replace UnionFS
			* OverlayFS - 2009
				* added to the standard Linux kernel source code in 2014
		* GlusterFS offers the possibility to mount different filesystems distributed across a network
			* rather than being located on the same machine
	* Unix and BSD
		* due to traditional Unix file system behavior their implementation of unions is complicated
			* problems
				* duplicate file names within a directory are not acceptable
					* would break applications' expectations of how a Unix file system works
					* compromised by using a stack-like precedence ordering on the unix's constituents
						* requires memory to record which files need to be skipped over during a directory listing
							* would be nearly stateless operation otherwise
				* deletion requires special support if
					* files with the same name exists in several of the union directory's constituents
						* deleting these files causes a file from one of the others to reappear in its stead
				* insertion of a directory into the stack can cause incoherency in the kernel's file name cache
				* renaming a file within a single mounted file system (using the `rename` system call) should be an atomic operation
					* renaming within a union mount can require changes to multiple of the union's constituent directories
						* possible solution is to disallow `rename` in such sittuations and require implementations to copy-and-delete instead
				* stable inode numbers are hard to implement correctly for
					* files
					* hard links
					* memory-mapped I/O (mmap)
		* this concept is super janky on this operating system but doable
	* Plan 9
		* union mounting is a central concept
			* replaces several older Unix conventions
				* several directories containing executables, unioned together at a single `/bin` directory 
					* replace the `PATH` variable for command lookup in the shell
						* this is badass
		* union semantics are greatly simplified compared to the implementations for POSIX-style operating systems
			* union of two directories is simply the concatenation of their contents
				directory listing of the union may display duplicate names
			* no effort is made to recursively merge subdirectories
				* extremely simple implementation
			* directories are unioned in a controllable order
			* `u/name` where `u` is a union directory
				* denotes the file called `name` in the first constituent directory that contains such a file

* what are [namespaces in linux](https://en.wikipedia.org/wiki/Linux_namespaces)?
	* a feature of the Linux kernel that isolates and virtualizes system resources of a collection of processes
		* resources that can be virtualized
			* process IDs
			* hostnames
			* user IDs
			* network access
			* interprocess communication
			* filesystems
	* a fundamental aspect of containers on Linux
	* Linux developers use the term namespace to refer to both the namespace kinds as well as to specific instances of these kinds
	* a Linux system is initialized with a single instance of each namespace type
		* after initialization additional namespaces can be created or joined
	* namespaces
		* namespace functionality is the same across all kinds
		* each process is associated with a namespace and can only see or use the resources associated with that namespace and descendant namespaces where applicable
			* each process or group of can have a unique view on the resources
	* namespace kinds
		* (mnt) mount
			* mount namespaces control mount points
			* uppon creation the mounts from the current mount namespace are copied to the new namespace
				* mount points created afterwards do not propagate between namespaces
		* (pid) process ID
			* provides processes with an independent set of process IDs from other namespaces
				* PID namespaces are nested
					* when a new process is created it will have a PID for each namespace from its current namespace up to initial PID namespace
					* initial PID namespace is able to see all processes
						* with different PIDs than other namespaces will see the same process as
			* first process created in a PID namespace is assigned process id 1
				* receives most of the same speial treatment as the normal init process
					* orphaned processes within the namespace are attached to it
				* terminating PID 1 process will immediately terminate all processes in its PID namespace and any descendants
		* (net) network
			* network namespaces virtualize the network stack
				* on creation a network namespace contains only a loopback interface
			* each network interface (physical or virtual) is present in exactly 1 namespace and can be moved between namespaces
			* each namespace will have a private set of
				* ip addresses
				* routing table
				* socket listing
				* connection tracking table
				* firewall
				* other network-related resources
			* on its destruction
				* network namespace will destroy any virtual interfaces within it and move any physical interfaces back to the initial network namespace
		* (ipc) interprocess communication
			* ipc namespaces isolate processes from SysV style inter-process communication
				* prevents processes in different ipc namespaces from intefering with each others inter-process communication
		* uts
			* allow a single system to appear to have different host and domain names to different processes
		* (user) user ID
			* provides both privilege isolation and user identification segregation across mutliple sets of processes
				* with administrative assistance it is possible to build a container with seeming administrative rights without actually giving elevated privileges to user processes
			* contains a mapping table converting user IDs from the container's point of view to the system's point of view
				* allows the root user to have user id 0 in the container but is actually treated as user id 1,400,000 by the system for ownership checks
				* a similar table is used for group id mappings and ownership checks
			* each namespace type is owned by a user namespace based on the active user namespace at the moment of creation
				* user with admin privileges in the appropriate user namespace will be allowed to perform admin action within that other namespace type
					* example
						* if a process has admin permission to change the IP address of a network interface, it may do so as long as the applicable network namespace is owned by a user namespace that either matches or is a child of the process' user namespace
							* initial user namespace has admin control over all namespace types in the system
		* cgroup suggested namespace 
			* to prevent leaking the control group to which a process belongs 
				* new namespace type is suggested
			* would hide the actual control group a process is a memeber of
				* a process in this namespace would see a path that is actually relative to the control group set at creation time hiding its true control group position and identity

* what is a [virtual loopback interface in the context of linux namespaces](https://en.wikipedia.org/wiki/Loopback#Virtual_loopback_interface)?
	* any traffic a process sends to a virtual loopback interface is immediately passed back up the network software stack as if it had been received from another device

* what is [SysV style inter-process communication](http://www.tldp.org/LDP/lpg/node22.html#SECTION00741000000000000000)?
	* three forms of IPC facilities
		* [message queues](http://www.tldp.org/LDP/lpg/node28.html#SECTION00742100000000000000)
			* an internal linked list wihtin the kernel's addressing space
			* messages can be sent to the queue in order and retrieved from the queue in several different ways
			* each message queue is uniquely identified by an IPC identifier
		* [semaphores](http://www.tldp.org/LDP/lpg/node47.html#SECTION00743100000000000000)
			* counters used to control access to shared resources by multiple processes
				* often used a locking mechanisms to prevent proceses from accessing a particular resource while another process is performing operations on it
		* [shared memory](http://www.tldp.org/LDP/lpg/node66.html#SECTION00744100000000000000)
			* the mapping of an area of memory that will be mapped and shared by more than one process
				* fastest form of ipc
					* no intermediation
			* information is mapped directly from a memory segment and into the addressing space of the calling process
				* a segment can be created by one process and subsequently written to and read from by any number of processes

* what is the [cgroups feature in linux](https://en.wikipedia.org/wiki/Cgroups)?
	* name is abbreviated from control groups
	* linux kernel feature targeted at collections of processes
		* resource limiting
			* groups can be set to not exceed a configured memory limit
				* also includes the file system cache
		* resource prioritization
			* some groups may get a larger share of CPU utilization or disk I/O throughput
		* resource accounting
			* measures a group's resource usage				
		* resource control
			* freezing groups of processes their checkpointing and restarting
	* provides a unified interface to many different use cases
		* controlling single processes
		* operating system-level virtualization
	* control group
		* a collection of processes that are bound by the same criteria and associated with a set of parameters or limits
			* groups can be hierarchical
				each group inherits limits from its parent group
		* kernel provides access to multiple controllers through the cgroup interface
			* also known as subsystems
		* vectors of using control groups
			* accessing the cgroup virtual file system manually
			* creating and managing groups on the fly using tools
				* `cgcreate`
				* `cgexec`
				* `cgclassify` (from `libcgroup`)
			* rules engine daemon
				* can automatically move processes of certain users, groups, or commands to cgroups as specified in its configuration
			* indirectly through other software that uses cgroups
				* docker
				* linux containters (LXC) virtualization
				* libvirt
				* systemd
				* open grid scheduler/grid engine
				* lmctfy

* what does [checkpointing mean in the linux environment](https://en.wikipedia.org/wiki/Application_checkpointing)?
	* a technique to add fault tolerance into computing systems
	* consists of saving a snapshot of the application's state
		* can restart from that point in case of failure
	* important for long running applications that are executed in failure-prone systems

* what is the [rules engine daemon in the linux environment]()?
	* i'm basically not going to really understand what this is without using it.
		* at least it didn't point me to an rfc

* what is [Docker, Inc the company](https://en.wikipedia.org/wiki/Docker,_Inc.)?
	* the company behind the development of the docker software open source project
	* 120 employees based in SF
	* history
		* october, 2013
			* dotCloud, Inc is renamed to Docker, inc
		* september, 2013
			* Docker and Red Hat announce alliance
		* august, 2014
			* Docker sells the doCloud platform and brand to cloudControl
		* october, 2014
			* Docker and Microsoft partner to help drive adoption of distributed applications
		* november, 2014
			* AWS announces EC2 container service
		* november, 2015
			* Docker passes 1.2 billion container downloads
		* march, 2016
			* Docker announces Docker Cloud
	* acquisitions
		* july, 2014	
			* Orchard
		* october, 2014
			* Koality
		* february, 2015
			* SocketPlane
		* march, 2015
			* Kitematic
		* october, 2015
			* Tutum
		* january, 2016
			* Unikernel Systems
		* march, 2016
			* Conductant
		* december, 2016
			* Infinit

* what is [Docker cloud](https://docs.docker.com/docker-cloud/getting-started/intro_cloud/)?

* what is an [abstracted virtualization interface in the field of virtualization]()?

* what is the [libvirt api](https://en.wikipedia.org/wiki/Libvirt)?

* what is [LXC in operating-system-level virtualization](https://en.wikipedia.org/wiki/LXC)?

* what is [systemd in linux](https://en.wikipedia.org/wiki/Systemd)?

* what does [systemd-nspawn do towards virtualization](https://en.wikipedia.org/wiki/Docker_(software))?

* what is [docker ce](https://www.docker.com/community-edition)?
	* the open source version of docker
	* features
		* the latest docker version with integrated tooling to build, test, and run container apps
		* available for free with software maintenance for the latest shipping version
		* integrated and optimized for developer desktops, linux servers and clouds
		* montly edge and quarterly stable release channels available
		* native desktop or cloud provider experience for easy onboarding
		* unlimited public and one free private repo storage as a service
		* automated builds as a service
		* image scanning and continuous vulnerability monitoring as a service
	* use cases
		* containerizing traditional applications
			* security
				* increase app security with containers to isolate apps and guarantee safe transport with image signing and verification of the app as it travels through the lifecycle
					* though security gained through containerization is not super huge, this can be considered a feature (it's more like a consequence)
			* portability
				* package everything the app needs into a container and migrate from one vm to another, to server, or cloud without having to refactor the app
			* cost savings
				* increase workload density on servers and VMs to save on infastructure costs
				* gain IT operational efficiency in deployment, maintenance, scale out, and roll back
			* pretty sure this use case about containerizing old apps, allowing you to not have to deal with upgrade hell
				* upgrading one thing breaks everything else
		* continuous integration and deployment [CI / CD]
			* accelerate app pipeline automation and app deployment and ship 13X more with Docker
				* accelerate
					* docker streamlines CI testing time and scales CI testing infastructure
				* integrate
					* APIs, open interfaces and webhooks allow for easy integration into existing tools and processes to futher automate the app pipeline
				* automate
					* save time and improve software quality by instantly spawning docker hosts and containers to run more tests in parallel
			* this is the process cris was talking to you about
				* do more research on this when you get to that
		* microservices
			* accelerate the path to modern app architectures
				* empower
					* developers are free to use the right tools and stacks for the app without worry of creating app conflicts
				* innovate
					* accelerate the rate of innocation of new softwate features
				* standardize
					* gain consistency in the app environment to gain efficiency without slowing down software development
		* IT infrastructure optimization
			* get more out of your infrastructure and save money
				* consolidate
					* containerize apps and consolidate the VMs and servers to reduce the infrastructure footprint
					* consolidate datacenters from M&A or migrate to cloud
				* efficiency
					* reduce operational overhead of patching and maintaining additional operating systems, virtual and physical servers
				* optimize
					* improve resource utilization, app scalability, disaster recovery and availability

* how do i [get started using docker](https://docs.docker.com/engine/getstarted/)?
	* download and install docker
	* verify your installation
		* `docker version`
			* check version
		* `docker ps`
			* shows running containers
		* `docker run hello-world`
			* will create a docker container hello world to test installation
		* `docker run  -it ubuntu bash`
			* super cool creates a container running ubuntu
		* `docker ps -a`
			* shows all containers, running or not running
	* [learn about images and containters](https://docs.docker.com/engine/getstarted/step_two/)
		* docker engine provides the core docket technology that enables images and containers
			* `docker run hello-world`
				* command break down
					* run
						* a subcommand that creates and runs a docker container
					* hello-world
						* tells docker which image to load into the container
				* docker engine steps for command
					* checks to see if `hello-world` image exists
					* downloads the image from docker hub if it didn't exist
					* loads the image into the container and runs it
	* [find and run the whalesay image](https://docs.docker.com/engine/getstarted/step_three/)
		* docker hub is a repository of images
			* github for images
		* tutorial
			* locate the whalesay image
			* run the whalesay image
				* `docker run docker/whalesay cowsay boo`
	* [building your own image](https://docs.docker.com/engine/getstarted/step_four/)
		* write a dockerfile
			* from
				import a existing image
			* run
				* run something in environment setup
			* cmd
				* what it should do after the environment is set up
		* build an image from your Dockerfile
			* `docker build -t docker-whale .`
				* build an image from a docker file
				* -t
					* gives yout image a tag
				* .
					* tells the docker build command to look in the current directory for a file called `Dockerfile`
		* learn about the build process
			* the output of the build command broken down
				* `sending build context to Docker daemon 2.048 kb`
					* docker checks to make sure it has everything it needs to build
				* `Step 1: FROM docker/whalesay:latest ---> 6b362a9f73eb`
					* looks at the `FROM` statement 
					* docker checks to see if it already has the `whalesay` image locally and pulls it from docker hub if not
					* `6b362a9f73eb`
						* at the end of each step an ID is printed
						* this is the ID of the layer that was created by this step
					* each line in a Dockerfile corresponds to a layer in the image
					* this layer shit definitely has to do with the union-capable file system concept
						* the ID probably corresponds to a layer of in the union file system
				* `Step 2: RUN apt-get -y update && apt-get install -y fortunes ---> Running in 05d4eda04526`
					* this is the output of the environment setup
					* when the `RUN` command finishes a new layer is created and the old one is removed
				* `Step 3: CMD /usr/games/fortune -a | cowsay`
					* a new intermediate container is created
					* docker adds a laer for the `CMD` line in the dockerfile and removes the intermediate container
				* the `docker-whale` is created
		* run your new docker-whale
			* `docker images`
				* verifies that your new image exist				
			* `docker run docker-whale`
				* runs the image docker-whale
	* [tag, push, and pull your image](https://docs.docker.com/engine/getstarted/step_six/)
		






* what is a [docker daemon](https://docs.docker.com/engine/reference/commandline/dockerd/#related-commands)?
	* `dockerd` 
		* a self-sufficient runtime for containers
			* a persistent process that manages containers
		* to run the daemon
			* `dockerd`
		* to run daemon with debug output
			* `dockerd -D`


* what does a [runtime really mean]()?
	* magic-wand word that basically means an environment that provides everything necessary to properly run the intended program












