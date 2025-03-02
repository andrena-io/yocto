**Project Scope: Yocto Image with KVM, K3s, and API Control**

**1. Project Overview & Goal:**

* **Goal:** To create a custom Yocto Linux image tailored for running virtual machines (using KVM) and containerized applications (using K3s).  This image will include a REST API to programmatically manage both KVM VMs and K3s deployments.
* **Use Case:**  This image is intended for scenarios where a lightweight, optimized, and programmatically controllable virtualization and containerization platform is needed.  Examples could include edge computing devices, private cloud infrastructure building blocks, or specialized server appliances.

**2. In Scope Features:**

* **Yocto Image Generation:**
    * Build a Yocto image based on a chosen distribution (e.g., Poky reference distribution, or a specific BSP layer).
    * Image will be optimized for size and performance, removing unnecessary packages and services.
    * Image should be bootable on target hardware (we'll define a target architecture - e.g., x86-64 for initial development, ARM64 for embedded example if desired).
    * Implement shared state (sstate) and download cache for improved build efficiency:
        * Configure persistent shared volume mounts for sstate and download caches
        * Enable cache sharing across dev containers to reduce rebuild times
        * Set up proper cache management policies (cleanup, retention)
        * Document cache setup and maintenance procedures
* **KVM Virtualization Support:**
    * Enable KVM kernel modules in the Yocto kernel configuration.
    * Include necessary user-space tools for KVM management:
        * QEMU 7.0+ (or latest stable version available in Yocto)
        * Core packages: `qemu-system-x86_64`, `qemu-kvm`, `qemu-img`, `qemu-nbd`
        * Optional: `virglrenderer` for improved VM graphics performance
        * Consider `libvirt` or direct QEMU management approach based on resource constraints
    * Hardware acceleration optimization:
        * Configure QEMU to leverage Intel N305 (Alder Lake-N) architecture features
        * Enable CPU model passthrough options and appropriate instruction sets (AVX2, SSE4.1/4.2, AES-NI)
        * Implement vCPU pinning strategy optimized for Gracemont E-cores
        * Configure memory optimization with huge pages support
    * Storage backend configuration:
        * Primary: QCOW2 with optimized cluster size, preallocation, and compression options
        * Alternative: Raw format for maximum performance use cases
        * Support for both thin and thick provisioning
        * Configure optimal I/O performance settings (aio, cache modes, IOThreads)
    * VM template framework:
        * Define YAML-based VM template system
        * Include standard templates (Minimal Linux, Standard Server, etc.)
        * Template parameters for CPU, memory, storage, and network configuration
    * PCI Passthrough for Network Functions:
        * Configure VFIO support for Intel 10G NIC passthrough
        * Implement static device mapping for dedicated VM functions
        * Create specialized VPP Router VM template with:
          * Optimized core allocation (minimum 4 dedicated cores)
          * Direct Intel 10G NIC assignment through PCI passthrough
          * Pre-configured VPP installation and basic router configuration
          * Hugepages allocation specifically sized for VPP
        * Document PCI device identification and IOMMU group management
    * Implement a REST API to control KVM VMs. API functionalities will include:
        * VM creation (from pre-defined templates or configurations)
        * VM starting, stopping, rebooting, deleting
        * VM listing (status, resources)
        * Basic VM resource monitoring (CPU, memory usage, disk I/O, network bandwidth)
        * Template management (list, import/export, versioning)
        * PCI passthrough device management (list, assign, unassign)
* **K3s Container Orchestration Support:**
    * Include K3s (lightweight Kubernetes) in the Yocto image.
    * Configure K3s to run in single-node mode initially.
    * Implement a REST API to interact with the K3s Kubernetes cluster. API functionalities will include:
        * Deployment of containerized applications (using Kubernetes Deployments).
        * Service creation and management.
        * Pod listing and status monitoring.
        * Namespace management (optional, could be in future scope).
* **REST API Framework:**
    * Choose a lightweight REST API framework (e.g., Flask, FastAPI for Python; or a C/C++ framework if desired for even lower overhead, but Python is generally easier for rapid prototyping).
    * API should be well-documented (e.g., using OpenAPI/Swagger).
    * API should handle basic authentication (e.g., API keys, basic HTTP auth - more advanced auth can be future scope).
* **Basic Host System Services:**
    * Include essential system services (e.g., SSH for remote access, systemd for service management, networking utilities).
    * Minimalistic approach to services to reduce image size and overhead.
* **Documentation:**
    * Basic documentation outlining the image build process, API usage, and initial setup.
    * Example API calls and scripts for common tasks (VM and container management).
    * Specific documentation for VPP router setup with PCI passthrough.

**3. Out of Scope Features (for initial phase):**

* **Advanced UI:** No web-based UI beyond the REST API. Command-line interaction and API usage are the primary focus.
* **Clustering and High Availability:** K3s will be single-node initially. Clustering and HA for KVM or K3s are out of scope for this phase.
* **Advanced Security Hardening:** Basic security practices will be followed, but deep security hardening (beyond standard Yocto security configuration) is out of scope initially.
* **Detailed Monitoring and Logging:** Basic resource monitoring for VMs is in scope. Comprehensive monitoring and logging solutions (e.g., Prometheus, ELK stack) are out of scope for this phase.
* **Storage Management:**  Basic disk image management for KVM VMs. Advanced storage solutions (e.g., Ceph, distributed storage) are out of scope initially.
* **Networking Complexity:** Basic bridged networking for VMs and K3s. Advanced networking configurations (e.g., VLANs, complex routing) are out of scope for the initial phase, though the VPP router VM will support basic routing functions.
* **Specific Hardware Optimizations (Beyond Architecture):** We will target a general architecture (e.g., x86-64 or ARM64). Deep hardware-specific optimizations beyond compiler flags and kernel configuration are out of scope initially.
* **Production-Ready Hardening and Testing:** This is a proof-of-concept/development project.  Production-level hardening, extensive performance testing, and reliability testing are out of scope for this initial scope.

**4. Technical Approach:**

* **Yocto Base:**  Start with Poky or a relevant BSP layer.
* **Build Optimization:**
    * Configure shared state (sstate) cache:
        * Set up persistent volume mounts for the sstate cache: `build/sstate-cache`
        * Configure in local.conf: `SSTATE_DIR = "${TOPDIR}/sstate-cache"`
    * Configure download cache:
        * Set up persistent volume mounts for downloads: `build/downloads`
        * Configure in local.conf: `DL_DIR = "${TOPDIR}/downloads"`
    * Implement volume sharing across development containers:
        * Create Docker volume mapping for both caches
        * Document container launch parameters for cache mounting
    * Cache management:
        * Set up rotation policies for sstate cache
        * Configure cleanup scripts for outdated cache entries
* **Kernel Configuration:**  Utilize Yocto's kernel recipe system to configure the Linux kernel. Enable:
    * KVM modules (`CONFIG_KVM`, `CONFIG_KVM_INTEL` or `CONFIG_KVM_AMD`).
    * VFIO support for PCI passthrough (if needed).
    * IOMMU support (`CONFIG_INTEL_IOMMU=y`, `CONFIG_INTEL_IOMMU_SVM=y`, `CONFIG_INTEL_IOMMU_DEFAULT_ON=y`).
    * Huge pages support for memory optimization.
    * Necessary containerization features (namespaces, cgroups, overlayfs, etc. - as discussed previously).
* **KVM Integration:**
    * Include QEMU packages in the Yocto image:
        * Core packages: `qemu-system-x86_64`, `qemu-kvm`, `qemu-img`, `qemu-nbd`
        * Configure appropriate permissions for `/dev/kvm` access
    * QEMU optimization for target hardware (Intel N305):
        * CPU flags and passthrough configurations
        * Memory allocation strategies with huge pages
        * I/O optimizations (virtio, aio=native/io_uring, vhost-net)
    * VM storage management:
        * Configure optimal QCOW2 settings (cluster size, compression, preallocation)
        * Implement storage management utilities
        * Support snapshot capabilities
    * VM template system:
        * Develop YAML template format for VM specifications
        * Create standard templates for common use cases
        * Implement template versioning and management
        * Create specialized VPP router template with PCI passthrough configuration
    * PCI Passthrough Implementation:
        * Configure the host system to support VFIO passthrough
        * Develop scripts to identify Intel 10G NIC PCI addresses and IOMMU groups
        * Implement static device mapping in QEMU/libvirt configurations
        * Optimize IRQ and interrupt handling for passthrough devices
        * Boot parameters for IOMMU support: `intel_iommu=on iommu=pt`
    * Decide on KVM management approach:
        * **Libvirt:** More feature-rich, but heavier.  Use `libvirt` and `libvirt-python` for API interaction.
        * **Direct QEMU management:**  More lightweight, but requires more manual scripting and potentially less feature-rich API.
        * For initial scope, **libvirt is likely easier for API integration**, but both approaches should be evaluated based on resource constraints.
* **K3s Integration:**
    * Include the K3s package in the Yocto image.  Use a recipe that downloads and installs the K3s binary.
    * Configure K3s to run as a systemd service.
    * Use the standard Kubernetes API for K3s control (accessible via `kubectl` or Kubernetes client libraries).
* **VPP Integration for Router VM:**
    * Include VPP packages in the VM template for the router
    * Configure VPP for high-performance packet processing
    * Optimize memory allocation for DPDK/VPP (additional hugepages)
    * Prepare basic router configuration templates
* **REST API Implementation:**
    * Choose Python Flask or FastAPI for REST API framework (or a C/C++ alternative if desired for lower overhead).
    * **For KVM API:** 
        * If using libvirt: Use `libvirt-python` library to interact with the libvirt daemon.
        * If direct QEMU: Develop Python wrapper for QEMU command-line interactions.
        * Expose REST endpoints for VM lifecycle management, resource monitoring, and template management.
        * Add endpoints for PCI device management and assignment
    * **For K3s API:** Use a Kubernetes Python client library (e.g., `kubernetes-client`) to interact with the K3s API server. Expose REST endpoints to deploy containers, manage services, etc., wrapping Kubernetes API calls.
    * API framework will handle request routing, JSON serialization/deserialization, and basic authentication.
    * Implement resource monitoring endpoints for both VMs and containers.
* **Development Environment Setup:**
    * Create Docker/container-based development environment:
        * Dockerfile with all required build dependencies
        * Container configuration with volume mounts for sstate and download caches
        * Scripts to launch development containers with proper cache sharing

**5. Deliverables:**

* **Yocto Image Recipe:**  Yocto recipe to build the custom image (e.g., `.bb` files, configuration files).
* **QEMU/KVM Configuration:**
    * Optimized QEMU configuration files for Intel N305
    * VM template definitions and management tools
    * Storage configuration and optimization scripts
    * PCI passthrough configuration scripts and templates
* **VPP Router VM Template:** 
    * Pre-configured VM image with VPP installed
    * Documentation for PCI device identification and assignment
    * Performance tuning recommendations
* **Bootable Yocto Image:**  The compiled Yocto image (e.g., `.iso`, `.wic` image) for target architecture.
* **REST API Code:**  Source code for the REST API (Python or chosen language), including API documentation (OpenAPI/Swagger).
* **Example API Usage Scripts:**  Example scripts (e.g., Python, `curl`) demonstrating how to use the REST API to create VMs and deploy containers.
* **VM Templates:** A set of standard VM templates ready for deployment.
* **Development Environment:**
    * Docker/container configuration files for build environment
    * Cache sharing configuration and scripts
    * Documentation on setting up and using the development environment
* **Basic Documentation:**  README file explaining the image build process, API endpoints, and basic usage.

**6. Potential Challenges:**

* **Yocto Build System Complexity:**  Yocto has a learning curve. Building a custom image can be time-consuming initially.
* **Build Optimization:**  Properly configuring and managing sstate and download caches to maximize build efficiency.
* **Kernel Configuration:**  Ensuring the kernel is correctly configured for KVM and containerization features, and optimized for performance.
* **QEMU/KVM Optimization:** Properly configuring QEMU for optimal performance on the Intel N305 architecture.
* **Resource Balancing:** Determining optimal resource allocation between host, KVM VMs, and K3s containers on the 8-core N305.
* **PCI Passthrough Configuration:** Setting up and troubleshooting VFIO/IOMMU for passthrough devices can be complex.
* **VPP Performance Tuning:** Optimizing VPP for high-throughput packet processing within the resource constraints.
* **Libvirt/KVM API Integration:**  Learning and effectively using the libvirt API in Python.
* **Kubernetes API Integration:**  Learning and using the Kubernetes API and Python client library.
* **REST API Development:**  Designing and implementing a robust and well-documented REST API.
* **Resource Management:**  Balancing resource usage between the host OS, KVM VMs, and K3s containers within a resource-constrained environment.
* **Testing and Debugging:**  Testing the entire system and debugging issues across Yocto build, kernel, KVM, K3s, and the API.

**7. Timeline (Estimated - Highly Variable):**

* **Phase 1: Development Environment and Yocto Image Setup (4-6 weeks):**
    * Set up Yocto build environment with sstate and download caches.
    * Configure Docker development containers with cache volume mounts.
    * Choose base layers and configure kernel for KVM and PCI passthrough.
    * Add QEMU/KVM packages to image.
    * Optimize QEMU configuration for Intel N305.
    * Implement VM template system.
    * Configure PCI passthrough for Intel 10G NIC.
    * Basic KVM VM creation and management (command line, no API yet).
* **Phase 2: VPP Router VM and K3s Integration (2-4 weeks):**
    * Create VPP router VM template with PCI passthrough.
    * Test and optimize VPP performance.
    * Add K3s to Yocto image.
    * Configure K3s single-node.
    * Choose REST API framework and set up basic API structure.
* **Phase 3: API Implementation (3-5 weeks):**
    * Implement basic REST endpoints for KVM VM management (start, stop, list).
    * Add PCI device management endpoints.
    * Implement REST API endpoints for K3s container deployment and management.
    * Enhance API documentation (OpenAPI).
    * Add basic API authentication.
    * Implement basic resource monitoring for VMs in API.
* **Phase 4: Testing, Documentation, and Refinement (2-3 weeks):**
    * Thorough testing of all features (VM and container management via API).
    * Benchmark and optimize VM performance on N305.
    * Specific testing for VPP router functionality with PCI passthrough.
    * Documentation and examples.
    * Code cleanup and refinement.

**Total Estimated Time: 11-18 weeks (depending on experience, complexity, and effort level).**  This is a rough estimate and can vary significantly.

**8. Next Steps:**

1. **Environment Setup:** 
   * Set up Yocto build environment with Docker containers.
   * Configure persistent volumes for sstate and download caches.
   * Create scripts for launching development environment with cache sharing.
2. **Base Image Selection:** Choose a base Yocto distribution and BSP layer if needed.
3. **Kernel Configuration:** Start configuring the kernel for KVM, IOMMU, and PCI passthrough.
4. **QEMU Package Selection:** Determine which QEMU packages to include and their configuration.
5. **PCI Device Identification:** Identify the Intel 10G NIC PCI address and IOMMU group.
6. **VM Template Design:** Design the VM template format, including specialized VPP router template.
7. **Layer Creation:**  Create Yocto layers to organize recipes and configurations.
8. **Start Building!** Begin building the Yocto image incrementally, focusing on KVM first, then K3s, and finally the API.

This scope provides a solid foundation for building your Yocto-based virtualization and containerization platform with specialized support for a VPP router VM using PCI passthrough. Remember to start incrementally, test frequently, and adjust the scope as needed based on progress and learnings. Good luck!
