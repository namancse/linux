.. SPDX-License-Identifier: GPL-2.0

Hyper-V storage
===============
Linux guests on Hyper-V commonly access block storage through one of two
interfaces:

* A synthetic SCSI controller implemented by the ``storvsc`` driver.  SCSI
  requests and their data buffers are exchanged with the Hyper-V storage
  Virtualization Service Provider (VSP) over VMBus.
* An NVMe controller assigned through Hyper-V virtual PCI (vPCI).  VMBus is
  used to discover and configure the PCI function, while normal NVMe I/O uses
  the native PCI data path.

Both interfaces use the Linux filesystem and block layers.  They diverge at
the ``blk-mq`` driver callback: synthetic SCSI enters the SCSI mid-layer and
``storvsc``, while NVMe enters the NVMe core and ``nvme-pci`` directly.

Storage I/O flow
----------------
The main submission paths are:

.. code-block:: text

                         Linux guest

                   +-------------------+
                   |    Application    |
                   |  read() / write() |
                   +---------+---------+
                             |
                   +---------v---------+
                   | VFS + filesystem  |
                   +---------+---------+
                             |
                   +---------v---------+
                   | Page cache,       |
                   | writeback, or DIO |
                   +---------+---------+
                             |
                   +---------v---------+
                   | bio -> block layer|
                   |      -> blk-mq    |
                   +---------+---------+
                             |
                +------------+------------+
                |                         |
      +---------v----------+    +---------v----------+
      | SCSI mid-layer     |    | NVMe core          |
      | sd -> scsi_lib     |    | nvme_setup_cmd()   |
      +---------+----------+    +---------+----------+
                |                         |
      +---------v----------+    +---------v----------+
      | storvsc            |    | nvme-pci           |
      | SRB + PFN array    |    | command + PRP/SGL  |
      +---------+----------+    +---------+----------+
                |                         |
   --------------+-------------------------+--------------
                 Hyper-V / host boundary
                |                         |
      +---------v----------+    +---------v----------+
      | VMBus shared rings |    | Assigned NVMe      |
      | -> storage VSP     |    | controller         |
      +--------------------+    | DMA SQ/CQ + MMIO   |
                                +--------------------+

   vPCI control path only:

      VMBus offer/configuration/MSI mapping/hotplug
          -> hv_pci -> PCI core -> nvme-pci device setup

The arrows point in the submission direction.  Completions travel upward in
the reverse direction.  For synthetic SCSI, completion arrives through a
VMBus channel callback.  For NVMe, completion arrives through an MSI-X or MSI
interrupt after the controller writes an NVMe completion queue entry.

An important distinction is the interface presented to the guest, rather
than the technology backing the storage in the host.  A disk backed by NVMe
hardware or an NVMe-based storage service may still appear as a synthetic
SCSI disk to the guest.  Such a disk follows the ``storvsc`` path.  Only a
guest-visible NVMe PCI function follows the native NVMe path described below.

Storvsc driver deep dive
------------------------

Top-level idea
~~~~~~~~~~~~~~
``storvsc`` is the Linux guest-side driver for the Hyper-V synthetic storage
controller.  In Hyper-V terminology it is a Virtualization Service Client
(VSC).  Its peer in the host is the storage Virtualization Service Provider
(VSP).

The driver is a bridge between three interfaces:

* Upward, it looks like a SCSI host adapter.  The SCSI mid-layer gives it
  ``struct scsi_cmnd`` objects through the host template's ``queuecommand``
  operation.
* Downward, it is a VMBus client.  It exchanges control and I/O packets with
  the storage VSP through shared-memory VMBus rings.
* On the wire, it uses the Hyper-V storage protocol.  The important objects
  are ``struct vstor_packet`` and its embedded ``struct vmscsi_request``.

Its central job can be summarized as follows:

.. code-block:: text

   SCSI mid-layer                         Hyper-V storage VSP
          |                                        ^
          | struct scsi_cmnd                       |
          v                                        |
   +---------------+     vstor_packet       +-------------+
   |    storvsc    |----------------------->| VMBus rings |
   |               |     + PFN array        +-------------+
   | queuecommand  |                               |
   | completion    |<------------------------------+
   +---------------+       status and sense data

The driver performs five main tasks:

1. Bind to synthetic SCSI, synthetic IDE, and synthetic Fibre Channel VMBus
   offers and register a corresponding SCSI host.
2. Negotiate the VSC/VSP protocol and controller capabilities.
3. Translate SCSI commands and scatterlists into Hyper-V storage packets and
   Hyper-V PFN arrays.
4. Select a VMBus channel, submit packets, correlate completions with SCSI
   commands, and return status to the SCSI mid-layer.
5. Coordinate rescans, LUN removal, SCSI error handling, suspend/resume, and
   controller teardown.

``storvsc`` is not itself a filesystem, a generic block driver, or the SCSI
disk driver.  In particular, ``sd`` converts a disk request into a SCSI CDB,
and the SCSI mid-layer owns normal dispatch, retry, timeout, and error-handler
machinery.  ``storvsc`` transports that command to the VSP and translates the
result back into SCSI status.

Main objects and their lifetimes
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
It helps to keep four object types separate while reading the driver:

.. code-block:: text

   struct hv_device                         one VMBus device offer
       |
       +-- channel                          primary VMBus channel
       +-- driver data ------------------+
                                         |
                                         v
                               struct storvsc_device
                               protocol/channel state
                               primary + subchannel map
                               init/reset requests

   struct Scsi_Host                        one Linux SCSI host
       |
       +-- shost_priv() ----------------> struct hv_host_device
       |                                  links hv_device and Scsi_Host
       |                                  scan/error workqueue
       |
       +-- struct scsi_device             one discovered LUN
               |
               +-- blk-mq request
                       +-- struct scsi_cmnd
                               +-- scsi_cmd_priv()
                                      struct storvsc_cmd_request

The cardinality and ownership are:

.. list-table:: Storvsc object ownership
   :header-rows: 1

   * - Object
     - Number
     - Owner and lifetime
     - Main role
   * - ``hv_device``
     - One per VMBus storage offer
     - VMBus core
     - Device-model anchor and primary-channel owner
   * - ``vmbus_channel``
     - One primary plus zero or more subchannels
     - VMBus core
     - Ring pair, callback context, and interrupt target CPU
   * - ``Scsi_Host``
     - One per storvsc controller
     - SCSI core, from ``scsi_host_alloc()`` to ``scsi_host_put()``
     - Publishes one Linux HBA and its dispatch limits
   * - ``hv_host_device``
     - One per ``Scsi_Host``
     - Tail-private storage in the ``Scsi_Host`` allocation
     - SCSI-facing link to ``hv_device`` and scan/error work
   * - ``storvsc_device``
     - One per storvsc controller
     - Storvsc, from ``kzalloc()`` through drain and ``kfree()``
     - Negotiated VSP state, channel selection, and teardown accounting
   * - ``scsi_device``
     - One per discovered target/LUN
     - SCSI core
     - Linux representation of an individual disk-like device
   * - ``storvsc_cmd_request``
     - One per normal SCSI command, plus init and reset objects
     - SCSI command-private storage or embedded in ``storvsc_device``
     - Per-command wire packet, payload description, and completion state

``struct hv_device`` is created and owned by the VMBus core.  The primary
channel belongs to that device.  ``storvsc_probe()`` attaches a
``struct storvsc_device`` with ``hv_set_drvdata()`` so submission and callback
paths can find the controller's protocol and channel state.

``struct Scsi_Host`` is allocated through the SCSI core.  Extra bytes after
the host hold ``struct hv_host_device``; ``shost_priv()`` returns that object.
It is the SCSI-facing link back to the VMBus device and owns the ordered
workqueue used for scans and error-related work.

The host template sets ``cmd_size`` to ``sizeof(struct storvsc_cmd_request)``.
The SCSI request allocation therefore includes one such driver-private object
for each ``struct scsi_cmnd``.  It holds the outgoing ``vstor_packet``, the
multipage-buffer descriptor, and the pointer back to the SCSI command.  No
separate allocation is needed for the normal-sized per-command state.

The ``init_request`` and ``reset_request`` objects are different.  They are
embedded in ``struct storvsc_device`` because protocol initialization and bus
reset are controller operations rather than ordinary tagged SCSI commands.

The two controller-private structures provide lookup paths in opposite
directions.  A SCSI callback starts with the host or command::

  scsi_cmnd -> scsi_device -> Scsi_Host
             -> shost_priv() -> hv_host_device -> hv_device
             -> hv_get_drvdata() -> storvsc_device

A VMBus callback starts with the channel::

  vmbus_channel -> primary channel's device_obj -> hv_device
                -> hv_get_drvdata() -> storvsc_device -> Scsi_Host

This explains the apparently duplicated back-pointers.  ``hv_host_device``
belongs to the SCSI host's lifetime and gives SCSI entry points a route to
VMBus.  ``storvsc_device`` belongs to the VMBus driver's bound lifetime and
gives channel callbacks a route to SCSI.  It also carries state that must be
marked ``destroy``, drained, unpublished with ``hv_set_drvdata(NULL)``, and
freed independently of the SCSI host allocation.

For normal I/O, ``storvsc_queuecommand()`` obtains the per-command
``storvsc_cmd_request`` with ``scsi_cmd_priv()`` and fills its
``vstor_packet`` and multipage-buffer description.  ``storvsc_do_io()`` uses
the controller-wide ``storvsc_device`` to select a channel and increment the
outstanding count.  The VMBus transaction ID is derived from the blk-mq
unique tag.  On completion, the callback uses that ID to recover the
``scsi_cmnd`` with ``scsi_host_find_tag()``, obtains the same private request,
calls ``scsi_done()``, and decrements the controller-wide outstanding count.

How probe is reached
~~~~~~~~~~~~~~~~~~~~
Module initialization runs ``storvsc_drv_init()`` before any controller is
probed.  It aligns the configured ring size, calculates how many maximum-size
packets fit in one outbound ring, optionally registers Fibre Channel transport
support, and registers ``storvsc_drv`` with the VMBus core.

The driver's ID table contains ``HV_SCSI_GUID``, ``HV_IDE_GUID``, and
``HV_SYNTHFC_GUID``.  When Hyper-V offers a matching VMBus device, the VMBus
core creates ``struct hv_device`` and invokes ``storvsc_probe()``.  Probe is
marked ``PROBE_PREFER_ASYNCHRONOUS``, so device-core probing normally need not
serialize boot while storage controllers initialize.

Probe has five broad phases:

.. code-block:: text

   VMBus offer
       |
       v
   Size and allocate the SCSI host
       |
       v
   Allocate storvsc state and connect to the storage VSP
       |
       v
   Apply negotiated limits to the SCSI host
       |
       v
   Publish the host and scan for LUNs

Probe phase 1: classify and size the host
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
The ``driver_data`` value in the matched ID entry tells probe whether this is
synthetic SCSI, synthetic IDE, or synthetic Fibre Channel.  This affects
subchannel use, SCSI address limits, scanning, and Fibre Channel setup.

SCSI and Fibre Channel controllers may use one primary channel plus multiple
subchannels.  Probe first estimates the maximum number of subchannels from
the online vCPU count::

   max_sub_channels = (num_online_cpus() - 1) /
                      storvsc_vcpus_per_sub_channel

The default divisor is four.  Despite the parameter name, this calculation is
used only to size ``can_queue`` before protocol negotiation.  It does not
determine the ``sub_channel_count`` sent to Hyper-V.  The subtraction accounts
for the already existing primary channel.  Synthetic IDE does not use
subchannels.

Before probe, ``storvsc_drv_init()`` calculated
``max_outstanding_req_per_channel`` from the usable outbound-ring bytes and
the maximum packet footprint.  Probe combines that value with the estimated
channel count and reserves the configured ring low-water percentage::

   can_queue = requests_per_channel * (subchannels + 1) *
               (100 - ring_low_water_percent) / 100

``scsi_host_alloc()`` copies this value into ``Scsi_Host.can_queue``.
When the host is registered, ``scsi_mq_setup_tags()`` uses it as the normal
tag depth of each blk-mq hardware queue.  A command needs a tag before it can
reach ``storvsc_queuecommand()``, and completion releases the tag for reuse.
Thus ``can_queue`` limits admitted, outstanding work and command-memory use;
it is not a count of LUNs or channels.  With multiple hardware queues, the
SCSI core treats it as a per-hardware-queue depth unless the low-level driver
requests a host-wide shared tag set.

This is capacity planning rather than an exact reservation of VMBus ring
space.  A selected channel can still run out of ring space.  In that case
``storvsc_do_io()`` returns ``-EAGAIN``, ``storvsc_queuecommand()`` reports
``SCSI_MLQUEUE_DEVICE_BUSY``, and the SCSI and block layers retry the command
later.

Probe then calls ``scsi_host_alloc(&scsi_driver,
sizeof(struct hv_host_device))``.  The SCSI core allocates but does not yet
publish the host, assigns ``host_no``, copies relevant limits from the host
template, and reserves the requested private area.  Probe initializes that
private ``hv_host_device`` with links to the new host and the offered
``hv_device``.

The host template is also where the SCSI core learns that:

* normal commands enter ``storvsc_queuecommand()``;
* each command needs a ``struct storvsc_cmd_request`` private area;
* the controller supports tagged commands and many commands per LUN;
* host reset and timeout handling are supplied by ``storvsc``; and
* scatterlists must not contain gaps across Hyper-V 4-Kbyte page boundaries.

Probe phase 2: allocate and publish controller state
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
Probe separately allocates ``struct storvsc_device``.  It initializes the
drain waitqueue and channel-map lock, records both the VMBus device and SCSI
host, and attaches the object to ``hv_device`` with ``hv_set_drvdata()``.

Publishing this pointer before opening the channel is necessary because the
VMBus callback can run as soon as the channel is active.  Callback and
submission paths retrieve this object with ``hv_get_drvdata()``.

``dma_set_min_align_mask()`` records the Hyper-V 4-Kbyte minimum alignment
requirement for DMA mappings.  The SCSI host template's
``virt_boundary_mask`` complements it by making the block and SCSI layers
form scatterlists that ``storvsc`` can express as a single logical range of
Hyper-V PFNs.

Probe phase 3: connect and negotiate with the VSP
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
``storvsc_connect_to_vsp()`` sets the maximum VMBus packet size and installs
``storvsc_next_request_id()`` as the transaction-ID allocator.  It then calls
``vmbus_open()`` with equal-sized inbound and outbound rings and registers
``storvsc_on_channel_callback()``.

At this point the transport exists, but it is not ready for SCSI I/O.
``storvsc_channel_init()`` performs this synchronous protocol sequence:

.. code-block:: text

   Guest VSC                                  Host storage VSP

   BEGIN_INITIALIZATION ----------------------------->
                        <---------------- COMPLETE_IO

   QUERY_PROTOCOL_VERSION (6.2) -------------------->
                        <---------- accept or reject
   QUERY_PROTOCOL_VERSION (6.0, if needed) ---------->
                        <---------- accept or reject

   QUERY_PROPERTIES -------------------------------->
                        <--- channel count, flags,
                             maximum transfer bytes

   FCHBA_DATA, for synthetic FC only ---------------->
                        <--- active WWNs

   END_INITIALIZATION ------------------------------>
                        <---------------- COMPLETE_IO

   CREATE_SUB_CHANNELS, when supported -------------->
                        <--- completion and later
                             VMBus subchannel offers

Each control operation reuses ``stor_device->init_request`` and uses the
reserved ``VMBUS_RQST_INIT`` transaction ID.  ``storvsc_execute_vstor_op()``
initializes a completion, sends the packet on the primary channel, and waits
up to ``storvsc_timeout`` seconds.

The same channel callback used later for normal I/O also handles these
responses.  When ``storvsc_on_channel_callback()`` sees
``VMBUS_RQST_INIT``, it copies the response into ``init_request.vstor_packet``
and completes the waiter.  Normal tagged SCSI lookup is not used during this
handshake.

Protocol negotiation tries the newest supported version first: version 6.2
for Windows 10/Windows Server 2016 and later, followed by version 6.0 for
Windows 8.1/Windows Server 2012 R2 and the Azure HvLite paravisor.  Probe
fails if neither version is accepted.

``QUERY_PROPERTIES`` supplies two values that control later setup:

* ``max_transfer_bytes`` limits the size of one storage request.
* ``max_channel_cnt`` and ``STORAGE_CHANNEL_SUPPORTS_MULTI_CHANNEL`` describe
  the VSP's multichannel capability.

The driver allocates ``stor_chns`` with one slot per possible CPU, installs
the primary channel in the slot for its target CPU, and records that CPU in
``alloced_cpus``.  The array is a CPU-to-channel selection cache; its length
does not mean there is one channel per CPU.

After ``END_INITIALIZATION``, ``handle_multichannel_storage()`` asks the VSP
for no more than the smaller of its offered channel count and the number of
additional online CPUs.  Hyper-V subsequently offers the subchannels through
VMBus.  ``handle_sc_creation()`` opens each ring with the same callback and
adds the channel to the CPU map.  A subchannel setup failure is logged but
does not fail the already usable primary-channel controller.

Two different subchannel calculations
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
The driver calculates two values at different times for different purposes:

.. list-table:: Storvsc channel calculations
   :header-rows: 1

   * - Calculation
     - Formula
     - Purpose
   * - Estimated subchannels
     - ``(online CPUs - 1) / storvsc_vcpus_per_sub_channel``
     - Size the SCSI host's ``can_queue`` limit before VSP negotiation
   * - Requested subchannels
     - ``min(online CPUs - 1, VSP max_channel_cnt)``
     - Fill ``VSTOR_OPERATION_CREATE_SUB_CHANNELS.sub_channel_count``

For example, with 16 online CPUs and the default divisor of four, probe uses
three estimated subchannels when calculating ``can_queue``.  Later, if the
VSP advertises a maximum of eight, the guest requests eight subchannels, not
three.  If the VSP advertises three, the guest requests three and the result
happens to look like one subchannel per four CPUs.

The current Linux code therefore does not enforce an actual one-subchannel-
per-four-CPUs topology.  The host-advertised maximum may embody its own
policy, but that policy is not established by the guest driver or its source
comments.  ``can_queue`` is a dispatch-capacity estimate, while
``sub_channel_count`` controls channel creation.

The estimate is not recomputed after negotiation.  If fewer subchannels are
created than estimated, the tag budget may admit more work than the available
rings can accept at that instant; the busy-and-retry path supplies runtime
backpressure.  If more are created than estimated, the tag budget may leave
some aggregate ring capacity unused.  Using the requested count would be a
closer estimate, but it would still not be exact: ``num_sc`` records the
requested count, while each asynchronously offered subchannel can fail its
individual ``vmbus_open()``.  Successfully opened channels are represented by
``stor_chns`` and ``alloced_cpus`` instead.

What subchannels are for
~~~~~~~~~~~~~~~~~~~~~~~~
A VMBus channel contains an independent guest-to-host ring, host-to-guest
ring, and interrupt target CPU.  Using only the primary channel would make
all commands for the controller share one pair of rings and concentrate
completion processing on one CPU.  The ring lock, finite ring capacity, and
callback CPU could then become bottlenecks as I/O parallelism increases.

Subchannels provide additional transport lanes for the same VMBus device:

.. code-block:: text

                          one storvsc controller

   CPU 0 requests -> primary channel ----+
                       own ring pair      |
                                          |
   CPU 4 requests -> subchannel 1 --------+--> same storage VSP
                       own ring pair      |    and same set of LUNs
                                          |
   CPU 8 requests -> subchannel 2 --------+
                       own ring pair

After setup, a subchannel is functionally equivalent to the primary channel
for ordinary I/O.  It does not represent another disk, SCSI target, LUN, or
copy of the data.  A single request is submitted wholly on one selected
channel; requests are distributed across channels, but one request is not
striped across several channels.

The primary channel remains special for controller setup operations such as
protocol negotiation, requesting subchannel creation, and bus reset.  It is
also a normal I/O channel.

Hyper-V assigns each channel a target CPU for its synthetic interrupt.
Spreading channels across CPUs allows ring processing and completions to run
in parallel.  ``handle_sc_creation()`` opens every offered subchannel with
``storvsc_on_channel_callback()`` and records it in ``stor_chns`` at the
channel's target CPU.  ``storvsc_change_target_cpu()`` updates that map if a
channel's interrupt affinity changes.

``stor_chns`` is indexed by CPU but is a selection cache, not a list of
distinct channels.  Several entries may point to the same channel.  When the
issuing CPU has no cached entry, ``get_og_chn()`` prefers a channel associated
with that CPU, otherwise hashes across channels on the same NUMA node, and
caches the result.

For every command, ``storvsc_do_io()`` first tries the cached channel for the
issuing CPU.  If its outbound ring has no more than the configured low-water
percentage available, the driver looks for:

1. another channel with space on the same NUMA node;
2. a channel with space on any NUMA node; or
3. the original channel if all channels are busy.

Each channel uses the same transaction-ID scheme.  A completion arriving on
any channel carries the unique SCSI request tag plus one, so the common
callback can find the original ``struct scsi_cmnd`` independently of which
channel transported it.

The number of ``blk-mq`` hardware queues and the number of VMBus channels are
not required to be equal.  Hardware queues organize Linux request dispatch,
while VMBus channels are finite shared-memory transports to the VSP.  Many
issuing CPUs or hardware queues may therefore share a smaller set of VMBus
channels.

Probe phase 4: configure SCSI-visible limits
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
After negotiation, probe applies controller type and VSP capabilities to the
new ``Scsi_Host``.

``max_channel``, ``max_id``, and ``max_lun`` describe the SCSI address space
that scanning may examine.  They differ for synthetic SCSI, synthetic IDE,
and synthetic Fibre Channel.  ``max_cmd_len`` is 16 bytes, matching the CDB
space in ``struct vmscsi_request``.

The VSP's ``max_transfer_bytes`` is rounded down to a Hyper-V page boundary.
Fibre Channel additionally caps it at 512 Kbytes.  Probe converts the result
to 512-byte sectors for ``host->max_sectors`` and derives
``host->sg_tablesize`` from the maximum number of Hyper-V PFNs needed for a
transfer.  These limits cause the upper layers to split requests before they
reach ``storvsc_queuecommand()``.

For a non-IDE controller, ``host->nr_hw_queues`` is either the valid
``storvsc_max_hw_queues`` module parameter or the number of present CPUs.
Hardware queues are a Linux ``blk-mq`` scheduling concept.  They need not
match the number of VMBus channels; ``storvsc_do_io()`` maps an issuing CPU or
queue to an available primary channel or subchannel.

Probe phase 5: publish and scan
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
Probe creates an ordered workqueue for controller error and rescan work.
Ordering matters because host rescans, LUN removals, and capacity changes
must not race each other arbitrarily.

``scsi_add_host()`` publishes the host to the SCSI core and device model.  It
does not discover disks by itself.  For synthetic SCSI and Fibre Channel,
probe follows it with ``scsi_scan_host()``.  The scan issues commands such as
REPORT LUNS and INQUIRY; those commands already travel through
``storvsc_queuecommand()`` and the VSP just like later data I/O.

Synthetic IDE is handled specially.  Probe derives a target number from the
VMBus instance GUID and explicitly adds LUN 0 with ``scsi_add_device()``
instead of performing a general host scan.

When a disk-type LUN is discovered, the ``sd`` upper-level driver binds to
the resulting ``struct scsi_device`` and creates the gendisk and ``/dev/sdX``
device.  Probe can then return successfully; normal I/O is driven by the
block and SCSI layers rather than by the probe function.

Probe failure unwinding
~~~~~~~~~~~~~~~~~~~~~~~
The error labels reflect three ownership milestones:

* Before the VMBus connection succeeds, probe frees ``storvsc_device`` and
  drops the unpublished SCSI host reference directly.
* After connection, ``storvsc_dev_remove()`` marks the device as being
  destroyed, drains outstanding requests, clears the VMBus driver-data
  pointer, closes the channel, and frees the channel map and controller
  state.
* After ``scsi_add_host()``, probe first calls ``scsi_remove_host()`` so the
  SCSI core stops and removes published devices before tearing down the
  transport.

The workqueue is destroyed if it was created, and ``scsi_host_put()`` drops
the final host reference.  This reverse-order unwind is also the useful
mental model for understanding ``storvsc_remove()``.

State after successful probe
~~~~~~~~~~~~~~~~~~~~~~~~~~~~
At successful return, the following relationships are established:

.. code-block:: text

   Hyper-V VMBus offer
       -> hv_device + open primary channel
       -> optional open subchannels
       -> hv_device.driver_data = storvsc_device

   storvsc_device
       -> negotiated protocol and transfer limits
       -> Scsi_Host
       -> CPU-to-VMBus-channel map

   Scsi_Host
       -> registered SCSI devices/LUNs
       -> blk-mq queues
       -> queuecommand = storvsc_queuecommand

From then on, probe is out of the fast path.  A block request becomes a SCSI
command, ``storvsc_queuecommand()`` packages it for the VSP, and the VMBus
callback completes it.  Unsolicited enumerate or remove notifications queue
work that rescans the existing host rather than rerunning probe.

Suggested reading order after probe
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
Once probe and the main objects are understood, the remaining driver is best
read in this order:

1. Follow one ordinary command through ``storvsc_queuecommand()``,
   ``storvsc_do_io()``, ``storvsc_on_channel_callback()``, and
   ``storvsc_command_completion()``.  This is the normal fast path.
2. Study the three independent flow-control mechanisms: blk-mq/SCSI tags,
   per-LUN queue depth, and VMBus ring-full busy-and-retry handling.
3. Study channel selection in ``get_og_chn()`` and ``storvsc_do_io()``,
   including CPU caching, NUMA preference, ring-space fallback, and interrupt
   affinity changes.
4. Follow transaction-ID conversion from a ``storvsc_cmd_request`` pointer to
   ``blk_mq_unique_tag() + 1`` and back through ``scsi_host_find_tag()``.
   This is the lifetime guarantee that makes asynchronous completion safe.
5. Read ``storvsc_handle_error()``, the sense-data handling, invalid-LUN
   removal, host reset, and timeout policy.  These paths decide whether Linux
   retries, rescans, removes a LUN, or reports failure.
6. Read unsolicited ``ENUMERATE_BUS`` and ``REMOVE_DEVICE`` handling together
   with the ordered workqueue.  This explains storage hot-add and hot-remove.
7. Finish with remove, suspend, and resume.  The ``destroy`` flag,
   ``num_outstanding_req``, drain waitqueue, driver-data unpublication, and
   channel close order define the driver's concurrency and lifetime rules.

Protocol-version quirks, synthetic IDE and Fibre Channel differences, legacy
CHS geometry, and module tuning parameters are useful second-pass material.
They modify the main model but are not needed to understand an ordinary
synthetic SCSI read or write.

Common upper layers
-------------------
A typical filesystem I/O starts at ``read()`` or ``write()`` and passes
through the VFS and a filesystem.  The exact path depends on the filesystem,
whether the I/O is buffered or direct, and whether the requested data is
already in the page cache.

For buffered I/O, a read satisfied from the page cache does not reach the
block layer.  A buffered write normally dirties page-cache folios and returns
before writeback creates block I/O.  Direct I/O and page-cache misses create
block I/O more immediately.

When storage access is required, the filesystem or a helper such as ``iomap``
constructs a ``struct bio``.  A bio describes the operation, target block
device, sector range, and memory segments.  ``submit_bio()`` passes it to the
block layer, where it may be split, merged, throttled, plugged, or scheduled.
Consequently, there is not necessarily a one-to-one relationship between a
userspace operation, a bio, and a device command.

``blk_mq_submit_bio()`` converts bios into ``struct request`` objects.  The
request is eventually passed to the ``queue_rq`` operation for the target
device:

.. code-block:: text

   read()/write()
       -> VFS and filesystem
       -> page cache, writeback, or direct I/O
       -> struct bio
       -> submit_bio()
       -> blk_mq_submit_bio()
       -> struct request
       -> request_queue.mq_ops->queue_rq()

Synthetic SCSI storage
----------------------

Device discovery
~~~~~~~~~~~~~~~~
Hyper-V offers a synthetic SCSI controller to the guest as a VMBus device
with ``HV_SCSI_GUID``.  The VMBus core matches the offer with the ``storvsc``
driver and calls ``storvsc_probe()``.

``storvsc_probe()`` allocates a ``struct Scsi_Host`` and opens the primary
VMBus channel.  ``storvsc_channel_init()`` negotiates the storage protocol
version and capabilities with the VSP.  For supported hosts, ``storvsc`` also
creates VMBus subchannels so I/O can be distributed across CPUs and hardware
queues.

After registering the SCSI host, the SCSI mid-layer scans the targets and
LUNs reported by Hyper-V.  The SCSI disk upper-level driver, ``sd``, binds to
disk-type LUNs and registers the corresponding ``/dev/sdX`` block devices.

The generic block layer above SCSI and NVMe
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
The generic block layer is the first common storage layer below filesystems,
direct I/O, swap, and other block-device users.  It expresses I/O in terms of
byte counts, 512-byte sectors, memory segments, and operations such as read,
write, flush, and discard.  It does not understand SCSI CDBs or Hyper-V
packets, and the same machinery feeds both SCSI and NVMe block devices.

A buffered file read does not necessarily reach this layer: a page-cache hit
needs no device I/O.  On a cache miss, the filesystem maps file offsets to
device sectors and submits one or more ``bio`` objects.  Direct I/O and swap
also construct bios, while stacked block drivers such as device mapper may
remap, split, and resubmit them before they reach the SCSI disk queue.

The principal objects at the block-driver boundary are:

``struct bio``
  Describes one block operation over a sector range and a vector of memory
  segments.  It carries operation flags, the target block device, status, and
  an end-I/O callback.  A bio is the unit passed between block-device layers
  and eventually completed back to its submitter.

``struct request``
  Is blk-mq's scheduling and driver-dispatch unit.  It contains one or more
  compatible bios, the combined sector and byte range, operation flags, queue
  pointers, a timeout, and a tag.  SCSI receives a request, not an individual
  filesystem bio.

``struct request_queue``
  Represents the queue and limits of one block disk.  The limits inherited
  from the SCSI host and device tell the block layer how large an I/O may be,
  how many segments it may contain, its alignment requirements, and which
  operations are supported.

``struct blk_mq_ctx``
  Is a software submission context associated with a CPU.  Per-CPU contexts
  avoid a single global submission lock when many CPUs issue I/O concurrently.

``struct blk_mq_hw_ctx``
  Is a hardware dispatch context.  The tag-set queue map assigns software
  contexts to hardware contexts.  It owns dispatch state and the list of
  requests that were ready but could not yet be accepted below.

``struct blk_mq_tag_set``
  Describes the driver's hardware queues, queue depth, callback operations,
  and private payload size.  Tags identify occupied request slots and bound
  the number of commands that can be in flight.

Bio submission and request formation
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
For a non-stacking SCSI disk, the important submission path is::

  submit_bio()
    -> submit_bio_noacct()
    -> __submit_bio()
    -> blk_mq_submit_bio()

Before a bio becomes a request, the block layer validates it against the
queue's current limits.  It rejects unsupported or misaligned operations and
uses ``__bio_split_to_limits()`` when an operation exceeds limits such as the
maximum transfer size or segment count.

``blk_mq_submit_bio()`` then tries to merge the bio with a compatible request.
Merging adjacent bios can reduce dispatch and protocol overhead, but is only
allowed when operation, range, limits, and request flags permit it.  If no
merge is possible, blk-mq obtains a request from the queue's tag-set-backed
request pool and ``blk_mq_bio_to_request()`` attaches the bio and initializes
the request's range and operation.

A task may have an active block plug.  In that case, new requests are held
briefly in the plug so adjacent I/O produced by the same task can be merged
before dispatch.  Without a plug, the request may still enter an I/O scheduler
such as ``mq-deadline``, depending on queue configuration.  The scheduler can
reorder requests for latency, fairness, or locality, but it does not alter the
SCSI command because that command has not been prepared yet.

Dispatch into SCSI
^^^^^^^^^^^^^^^^^^
When a request is selected for dispatch, blk-mq maps it to a
``blk_mq_hw_ctx`` and obtains the resources needed to issue it.  A request can
take a fast path directly from submission to dispatch, or arrive later from a
plug, scheduler, or hardware-context dispatch list.  These paths converge at
the queue's ``blk_mq_ops.queue_rq`` callback:

.. code-block:: text

  bio
    -> validate and split to request_queue limits
    -> merge with an existing request, or allocate a request
    -> task plug, optional I/O scheduler, or direct issue
    -> blk_mq_hw_ctx dispatch
    -> request_queue.mq_ops->queue_rq()
    -> scsi_queue_rq()

The SCSI tag set installed ``scsi_queue_rq()`` as this callback.  This is the
precise handoff from generic block semantics to SCSI semantics.  Only after
the handoff does ``sd`` translate the request operation and sector range into
a CDB, and only later does storvsc encode that command for VMBus.

The callback return value is part of blk-mq flow control.  ``BLK_STS_OK``
means the lower layer accepted the request.  ``BLK_STS_RESOURCE`` or
``BLK_STS_DEV_RESOURCE`` means it did not; blk-mq retains the request on a
dispatch list and runs the queue again when resources may be available.  A
terminal error instead ends the request.  Therefore a request is neither lost
nor treated as in flight merely because a dispatch attempt reached SCSI.

Completion back to bios
^^^^^^^^^^^^^^^^^^^^^^^
On the return path, SCSI completes the request only after its retry and error
policy has selected a final outcome.  The block layer then accounts the I/O,
advances or completes every bio attached to the request, invokes each bio's
end-I/O path, releases quality-of-service and scheduler state, and returns the
request and tag to the reusable pool.  In simplified form::

  scsi_finish_command()
    -> SCSI I/O completion processing
    -> blk_update_request() / blk_mq_end_request()
    -> bio_endio()
    -> filesystem, direct-I/O, swap, or other original completion

Partial completion is possible: ``blk_update_request()`` can complete a byte
range and leave the request positioned at its remaining bios and sectors.
Normal storvsc disk I/O usually reports a final SCSI command result, after
which the completed request slot can be reused for an unrelated operation.

Blk-mq also owns generic request timeout tracking.  Its SCSI ``timeout``
callback hands an expired request to SCSI timeout and error-handling policy;
storvsc's low-level timeout callback participates below that boundary.  Queue
freezing uses the queue usage reference to stop new submission and wait for
users during device removal or reconfiguration.

Thus the generic block layer decides **when and in what grouping** storage
work is dispatched.  The SCSI layer decides **how the block operation is
represented and recovered as a SCSI command**.  Storvsc decides **how that
command is transported through Hyper-V**.

The layer directly above storvsc
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
The immediate caller of ``storvsc`` is the Linux SCSI mid-layer, principally
the dispatch code in ``drivers/scsi/scsi_lib.c``.  For a disk, the ``sd``
upper-level driver also participates by translating block operations into SCSI
CDBs.  The responsibilities are divided as follows:

.. code-block:: text

  block layer
     struct request: sector range, operation, bios
          |
          v
  SCSI mid-layer                 drivers/scsi/scsi_lib.c
     queue admission, command lifetime, retry and error handling
          |
          +--> sd             drivers/scsi/sd.c
          |    build READ, WRITE, FLUSH, DISCARD, and other CDBs
          |
          v
  SCSI low-level driver
     storvsc: encode the prepared SCSI command for Hyper-V

The object hierarchy follows the same split.  One ``Scsi_Host`` represents
the synthetic controller registered by ``storvsc``.  It contains targets and
one ``scsi_device`` for each discovered LUN.  The ``sd`` driver binds to a
disk-type ``scsi_device`` and provides its block disk and request queue.
``Scsi_Host.hostt`` points to the ``scsi_host_template`` supplied by
``storvsc``; that template's ``queuecommand`` method is the final dispatch
boundary from the SCSI mid-layer into the low-level driver.

For SCSI blk-mq, the driver-private payload of each ``struct request`` begins
with a ``struct scsi_cmnd``.  ``blk_mq_rq_to_pdu()`` obtains that command from
the request, while ``scsi_cmd_to_rq()`` performs the reverse conversion.
Storage reserved through ``scsi_host_template.cmd_size`` follows the
``scsi_cmnd``; ``scsi_cmd_priv()`` returns that area, which is a
``storvsc_cmd_request`` for this host.

How a blk-mq request is linked to storvsc
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
The link is established during controller and LUN registration, before any
ordinary I/O is submitted.  Storvsc defines a ``scsi_host_template`` with::

  .queuecommand = storvsc_queuecommand
  .cmd_size = sizeof(struct storvsc_cmd_request)

``storvsc_probe()`` passes that template to ``scsi_host_alloc()`` and later
calls ``scsi_add_host()``.  Host registration reaches
``scsi_mq_setup_tags()``, which creates the host's blk-mq tag set with:

* ``queue_rq = scsi_queue_rq`` in the SCSI blk-mq operations;
* ``driver_data = Scsi_Host``;
* the hardware-queue count and queue depth selected for the host; and
* a per-request payload large enough for ``scsi_cmnd``,
  ``storvsc_cmd_request``, and the inline scatterlist.

When scanning creates a ``scsi_device`` for a LUN, ``scsi_alloc_sdev()``
creates that device's ``request_queue`` from the same host tag set and stores
the ``scsi_device`` as the queue's ``queuedata``.  The resulting static pointer
chain is::

  request->q->queuedata = scsi_device
  scsi_device->host = Scsi_Host
  Scsi_Host->hostt = storvsc scsi_host_template
  Scsi_Host->hostt->queuecommand = storvsc_queuecommand

Blk-mq allocates driver command data immediately after each request.  For a
storvsc-backed SCSI queue, the relevant memory layout is:

.. code-block:: text

  +------------------------+
  | struct request         |
  +------------------------+ <- blk_mq_rq_to_pdu(request)
  | struct scsi_cmnd       |
  +------------------------+ <- scsi_cmd_priv(scsi_cmnd)
  | storvsc_cmd_request    |
  +------------------------+
  | inline SCSI SG storage |
  +------------------------+

At dispatch, ``scsi_queue_rq()`` obtains the embedded command with
``blk_mq_rq_to_pdu(request)``.  After ``sd`` prepares its CDB,
``scsi_dispatch_cmd()`` follows ``cmd->device->host->hostt->queuecommand`` and
therefore enters ``storvsc_queuecommand()``.  Storvsc then obtains both kinds
of state without allocating a separate normal command object::

  host_dev = shost_priv(host)
  cmd_request = scsi_cmd_priv(scmnd)

``host_dev`` leads to the controller's ``hv_device``.  ``cmd_request`` holds
this command's Hyper-V packet and PFN descriptor.  The original blk-mq request
remains recoverable through ``scsi_cmd_to_rq(scmnd)``, which is how storvsc
derives the unique transaction tag used to match the eventual completion.

In short, queue registration selects the call path, and request-private memory
supplies the per-I/O storvsc state.  The generic ``struct request`` itself does
not need to know about Hyper-V or storvsc.

What the SCSI blk-mq layer does
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
This layer adapts the block multi-queue model to the SCSI device, target, and
host model.  Three objects have different scopes:

``Scsi_Host.tag_set``
  Describes resources shared by the synthetic SCSI controller.  It supplies
  the SCSI blk-mq callbacks, hardware-queue count, tag depth, and command
  payload size used when requests are allocated.

``scsi_device.request_queue``
  Is the block queue for one discovered LUN.  It is allocated from the host's
  tag set.  Its ``queuedata`` points back to that ``scsi_device``, allowing a
  dispatched request to find its LUN, target, and host.

``struct request``
  Represents one block operation after adjacent bios have been merged and
  scheduled.  Its blk-mq tag identifies one occupied command slot.  The
  request's private payload carries the corresponding ``scsi_cmnd`` and
  storvsc state for as long as that slot is in use.

Blk-mq maps submitting CPUs onto hardware dispatch contexts, or ``hctx``
objects.  For storvsc, the SCSI core supplies the generic CPU mapping because
the storvsc host template does not override ``map_queues``.  An ``hctx`` is
not a VMBus channel: it is blk-mq's dispatch and tag-allocation context.
``storvsc_queuecommand()`` separately passes the current CPU to
``storvsc_do_io()``, which prefers that CPU's assigned VMBus channel but may
use another channel when rings are busy.  Consequently the two queueing layers
cooperate without requiring a permanent one-to-one mapping between an ``hctx``
and a VMBus channel.

The layer performs the following jobs for each normal request:

* **Admission and fairness.**  ``scsi_queue_rq()`` checks whether the LUN,
  target, and host can accept another command and accounts it against their
  busy limits.  This is above storvsc because several LUN request queues may
  share one target or host, while blk-mq otherwise sees separate block queues.
* **Command initialization.**  It resets the reusable ``scsi_cmnd`` and the
  storvsc-private area, associates the command with the ``scsi_device``, and
  prepares scatter-gather storage and sense data.
* **Protocol preparation.**  It invokes the upper-level SCSI driver.  For a
  disk, ``sd_init_command()`` translates the block operation into a SCSI CDB
  and sets transfer direction, expected length, retry allowance, and related
  command fields.
* **Low-level dispatch.**  It starts the blk-mq request and calls storvsc's
  ``queuecommand`` method through the host template.  From this point storvsc
  owns transport submission until it either refuses the command immediately
  or later calls ``scsi_done()``.
* **Backpressure.**  If storvsc cannot put the command on a VMBus ring, it
  returns a SCSI queue-busy reason without accepting ownership.  The SCSI
  layer releases its busy accounting and returns a resource status to blk-mq,
  which dispatches the request again later.
* **Completion policy.**  After an accepted command completes, SCSI interprets
  the host, driver, and target status.  It can finish the request, retry it,
  delay it for a temporary device-busy condition, or move it to the SCSI error
  handler.  Storvsc supplies the result; the common SCSI layer supplies this
  policy.

For example, a disk READ follows this object and callback path:

.. code-block:: text

  bio(s)
    -> struct request on scsi_device.request_queue
    -> scsi_queue_rq()
         queuedata                         -> scsi_device -> Scsi_Host
         blk_mq_rq_to_pdu(request)         -> scsi_cmnd
         scsi_cmd_priv(scsi_cmnd)          -> storvsc_cmd_request
    -> sd_init_command()                   build READ CDB
    -> scsi_dispatch_cmd()
    -> storvsc_queuecommand()              map data and build vstor_packet
    -> storvsc_do_io()                     submit on a VMBus channel
    -> Hyper-V storage service

  completion packet with request tag
    -> scsi_host_find_tag()                recover scsi_cmnd
    -> scsi_cmd_priv(scsi_cmnd)            recover storvsc_cmd_request
    -> storvsc_on_io_completion()          record SCSI/transport result
    -> scsi_done()
    -> blk_mq_complete_request()
    -> scsi_complete()                     finish, retry, requeue, or error-handle
    -> complete request bios

The blk-mq tag is particularly useful at the virtualization boundary.
The request's ordinary ``tag`` is unique only within one hardware context, so
``blk_mq_unique_tag()`` combines the hardware-context number in its upper bits
with that local tag in its lower bits.  Storvsc adds one to this value for the
VMBus transaction ID because transaction ID zero is reserved.  On completion
it subtracts one and uses ``scsi_host_find_tag()`` to recover the live
``scsi_cmnd``.  Thus the unique tag is simultaneously a blk-mq command-slot
identity and storvsc's wire correlation key; no raw guest pointer must be
echoed by the host for ordinary I/O.

The separation also explains why storvsc does not implement a blk-mq
``queue_rq`` callback directly.  Blk-mq provides generic CPU-to-queue dispatch,
tag allocation, scheduling, timeout, and completion machinery.  The SCSI core
adds shared host/target/LUN policy and SCSI error recovery.  Storvsc only has
to implement the transport-specific boundary: encode an already prepared SCSI
command for Hyper-V and report its result.

``scsi_queue_rq()`` is the SCSI request queue's blk-mq dispatch callback.  For
an ordinary command it:

1. checks the ``scsi_device``, target, and host queue state and accounts the
  command against their queue limits;
2. initializes the ``scsi_cmnd`` and the low-level driver's private area;
3. calls ``scsi_prepare_cmd()``, which invokes the bound upper-level driver's
  command initializer;
4. marks the blk-mq request started; and
5. calls ``scsi_dispatch_cmd()``.

For a disk, the upper-level initializer is ``sd_init_command()``.  It switches
on the block request operation.  A read or write enters
``sd_setup_read_write_cmnd()``, which allocates the SCSI scatter-gather tables,
checks device state, capacity, and logical-block alignment, converts the sector
range to a logical block address and block count, and selects an appropriate
READ or WRITE CDB format.  It also records transfer length, underflow, and
retry information in the ``scsi_cmnd``.

``scsi_dispatch_cmd()`` verifies that the device and host still exist, checks
that the CDB fits the host's ``max_cmd_len``, and finally calls::

  host->hostt->queuecommand(host, cmd)

For this host, that expression calls ``storvsc_queuecommand()``.  A zero return
means the low-level driver accepted the command and will eventually call
``scsi_done()``.  A queue-busy return means it did not accept the command;
``scsi_queue_rq()`` unwinds its busy accounting, returns a resource shortage to
blk-mq, and the request is dispatched again later.  This contract is why a
full VMBus ring can apply backpressure without completing or losing the SCSI
command.

Completion crosses the same boundary in reverse.  Storvsc records the VSP,
SRB, and target status in ``scsi_cmnd.result`` and calls ``scsi_done()``.
That function asks blk-mq to complete the request, whose SCSI ``.complete``
callback is ``scsi_complete()``.  ``scsi_decide_disposition()`` then classifies
the command result.  Depending on that disposition, the mid-layer:

* calls ``scsi_finish_command()`` for normal completion;
* reinserts the command for a retry;
* requeues it after a temporary device-busy condition; or
* gives it to the SCSI error handler for recovery.

Thus storvsc reports what happened at the Hyper-V transport and SCSI target,
but the SCSI mid-layer decides the generic recovery policy.  On successful
completion, the ``sd`` completion path accounts the transferred bytes and the
request ultimately returns through blk-mq to its bios.

Request submission
~~~~~~~~~~~~~~~~~~
The SCSI request queue installs ``scsi_queue_rq()`` as its ``blk-mq``
``queue_rq`` operation.  The submission path is:

.. code-block:: text

   struct request
       -> scsi_queue_rq()
       -> scsi_prepare_cmd()
       -> sd_init_command()
       -> sd_setup_read_write_cmnd()
       -> scsi_dispatch_cmd()
       -> storvsc_queuecommand()
       -> storvsc_do_io()
       -> vmbus_sendpacket_mpb_desc()
       -> hv_ringbuffer_write()
       -> Hyper-V storage VSP

``scsi_prepare_cmd()`` associates the request with a ``struct scsi_cmnd``.
For a disk read or write, ``sd_setup_read_write_cmnd()`` translates the block
sector and length into a SCSI READ or WRITE Command Descriptor Block (CDB).
``scsi_dispatch_cmd()`` then calls the low-level driver's ``queuecommand``
operation, which is ``storvsc_queuecommand()``.

Scatter-gather lists
~~~~~~~~~~~~~~~~~~~~
A logically contiguous I/O range is often not contiguous in physical memory.
For example, a 12-Kbyte write may come from three guest pages whose page frame
numbers are unrelated:

.. code-block:: text

   Logical write buffer

      bytes 0-4095       bytes 4096-8191     bytes 8192-12287
           |                    |                    |
           v                    v                    v
      guest PFN 0x120      guest PFN 0x9a0      guest PFN 0x441

Copying these pages into one physically contiguous temporary buffer would add
CPU and memory-bandwidth overhead.  Instead, the block and SCSI layers build
a scatter-gather list.  Each ``struct scatterlist`` entry describes a memory
region using a page, an offset within that page, and a length.  The entries are
ordered according to the logical byte stream even when the backing pages are
physically unrelated.

The name describes both data directions:

* A write gathers data from several memory regions into one device command.
* A read scatters data returned by one device command into several regions.

For a ``struct scsi_cmnd``, ``scsi_sglist()`` returns the first entry,
``scsi_sg_count()`` returns the number of entries, and ``scsi_bufflen()``
returns the total logical transfer length.

Before a device can access the memory, ``scsi_dma_map()`` passes the list
through the DMA API.  This creates device-visible DMA addresses and lengths.
The DMA API may translate addresses through an IOMMU, use a bounce buffer, or
coalesce adjacent entries.  Its return value is therefore the number of
DMA-mapped segments, which may differ from the original entry count.  A driver
uses ``sg_dma_address()`` and ``sg_dma_len()`` only after this mapping and
balances it with ``scsi_dma_unmap()`` after completion or failed submission.

The SCSI scatterlist is a generic Linux representation; it is not sent to
Hyper-V directly.  ``storvsc_queuecommand()`` walks the DMA-mapped segments
and expands their address ranges into ordered 4-Kbyte Hyper-V PFNs:

.. code-block:: text

   SCSI scatterlist
       -> scsi_dma_map()
       -> DMA address/length segments
       -> Hyper-V PFN array + first-page offset + total length

The resulting ``vmbus_packet_mpb_array`` tells the VSP where each portion of
the logical buffer resides.  Only this descriptor and the SCSI request are
placed in the VMBus ring.  The VSP reads from those pages for a write or writes
into them for a read, avoiding a full data copy through the ring.

``storvsc_queuecommand()`` constructs a ``struct vmscsi_request`` containing
the CDB, LUN address, transfer direction, transfer length, and SRB flags.  It
maps the SCSI scatterlist with ``scsi_dma_map()`` and converts the resulting
DMA ranges into an array of 4-Kbyte Hyper-V page frame numbers (PFNs).
``HV_HYP_PAGE_SIZE`` is used because Hyper-V's page size is 4 Kbytes even
when the guest kernel uses a larger page size.

SRB means SCSI Request Block in this protocol.  It is a Windows/Hyper-V
storage-stack envelope around a SCSI CDB, represented by
``struct vmscsi_request`` in storvsc.  The CDB specifies the device operation,
such as READ or INQUIRY.  The SRB additionally supplies the controller address,
data direction and length, queueing flags, timeout, and fields for transport
status and SCSI status.  The SRB does not contain the data pages themselves.

Keep the three result layers separate:

* ``vstor_packet.status`` reports the VSP operation result;
* ``vmscsi_request.srb_status`` reports how the host storage stack transported
  or processed the request; and
* ``vmscsi_request.scsi_status`` reports the target's SCSI result, such as
  GOOD or CHECK CONDITION.  Sense data explains a CHECK CONDITION.

An SRB transport status of success therefore does not by itself prove that
the target completed the SCSI command successfully.

The PFN array is a ``struct vmbus_packet_mpb_array``.  It describes the guest
memory that the VSP accesses for the data transfer; the data itself is not
copied into the VMBus ring.  The accompanying ``struct vstor_packet`` has an
operation of ``VSTOR_OPERATION_EXECUTE_SRB``.

``storvsc_do_io()`` selects a primary channel or subchannel, preferring a
channel associated with the issuing CPU and NUMA node when possible.  It
passes the multipage-buffer descriptor and ``vstor_packet`` to
``vmbus_sendpacket_mpb_desc()``.  The VMBus layer writes these descriptors to
the guest-to-host ring and signals Hyper-V when the ring transitions from
empty to non-empty.

The principal data structure transformations are:

.. code-block:: text

   struct bio
       -> struct request
       -> struct scsi_cmnd containing a SCSI CDB and scatterlist
       -> struct vmscsi_request in a struct vstor_packet
       +  VMBus multipage-buffer PFN array

Request correlation
~~~~~~~~~~~~~~~~~~~
Submission and completion are asynchronous, so each VSP response must identify
the original Linux command.  ``storvsc_do_io()`` passes the address of the
command's ``storvsc_cmd_request`` as the ``requestid`` argument to
``vmbus_sendpacket()`` or ``vmbus_sendpacket_mpb_desc()``.  This address is an
internal guest argument; it is not the normal transaction ID exposed to the
host.

While holding the outbound ring lock, ``hv_ringbuffer_write()`` copies the
packet into the ring and then invokes the channel's
``storvsc_next_request_id()`` callback.  For ordinary I/O, that callback
returns::

  transaction ID = blk_mq_unique_tag(scsi_cmd_to_rq(request->cmd)) + 1

A request's local tag identifies a slot only within one blk-mq hardware queue.
``blk_mq_unique_tag()`` combines the hardware-queue number and local tag::

  unique tag = (hardware queue number << 16) | local tag

For example, hardware queue 3 and local tag 42 produce unique tag 196650 and
VMBus transaction ID 196651.  Adding one is necessary because transaction ID
zero is reserved for unsolicited VSP messages; hardware queue 0 and local tag
0 would otherwise produce zero.  Initialization and reset operations use the
separate reserved values ``VMBUS_RQST_INIT`` and ``VMBUS_RQST_RESET``.

The VSP echoes the transaction ID in its completion.  The channel callback
performs the inverse lookup::

  unique tag = transaction ID - 1
  scsi_cmnd = scsi_host_find_tag(shost, unique tag)
  storvsc_cmd_request = scsi_cmd_priv(scsi_cmnd)

``scsi_host_find_tag()`` extracts the hardware-queue number and local tag,
looks up the corresponding ``struct request`` in that queue's tag table,
verifies that the request is still started, and returns its embedded
``scsi_cmnd``.  A missing or stale lookup is rejected as an incorrect
transaction ID.  The tag remains assigned while the command is outstanding;
after storvsc supplies the result to ``scsi_done()``, SCSI and blk-mq complete
the request and make the tag available for later reuse.

Request completion
~~~~~~~~~~~~~~~~~~
The VSP performs the operation and places a
``VSTOR_OPERATION_COMPLETE_IO`` response in the host-to-guest VMBus ring.
The Hyper-V synthetic interrupt controller signals the guest.  The VMBus
interrupt path identifies the channel and schedules its registered callback,
``storvsc_on_channel_callback()``.

The request transaction ID is derived from the SCSI request tag.  The
callback converts the returned transaction ID to a tag, finds the original
``struct scsi_cmnd``, and unmaps its DMA mappings.  It then follows this path:

.. code-block:: text

   Hyper-V synthetic interrupt
       -> VMBus channel callback dispatch
       -> storvsc_on_channel_callback()
       -> storvsc_on_receive()
       -> storvsc_on_io_completion()
       -> storvsc_command_completion()
       -> scsi_done()
       -> blk-mq completion
       -> bio_endio()

``storvsc_on_io_completion()`` copies the SCSI status, SRB status, sense data,
and transferred length from the VSP response.  The SCSI mid-layer interprets
the result and may retry or invoke error handling.  A final completion calls
``blk_mq_end_request()``, which propagates status to each bio and eventually
invokes its ``bi_end_io`` callback.

Completion, timeout, and teardown invariants
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
The VSP is outside the guest's trust boundary, so the receive path validates
host-provided metadata before using it.  The channel callback rejects short
packets, invalid uses of the unsolicited transaction ID zero, and transaction
IDs that do not identify a started SCSI request.  Completion handling caps the
sense-data length at ``STORVSC_SENSE_BUFFER_SIZE`` and the reported transfer
length at the submitted payload length.

Storvsc also adapts a few known host storage-stack behaviors.  For INQUIRY,
MODE SENSE, and MODE SENSE(10), and for MAINTENANCE IN and PERSISTENT RESERVE
IN on synthetic Fibre Channel, ``storvsc_host_mishandles_cmd()`` causes the
returned SCSI and SRB statuses to be normalized to success before normal
completion processing.  This is targeted protocol compatibility, not a
general suppression of device errors.

``storvsc_on_channel_callback()`` processes packets for at most approximately
``CALLBACK_TIMEOUT`` milliseconds, currently 2 ms, in one invocation.  On
reaching that budget it commits the current inbound-ring read position and
returns.  For a batched VMBus channel, ``vmbus_on_event()`` then checks whether
packets remain and reschedules the channel tasklet.  This bounds one callback's
CPU occupancy without losing queued completions.

Ordinary command timeout behavior is unusual.  ``storvsc_eh_timed_out()``
unconditionally returns ``SCSI_EH_RESET_TIMER`` because the Hyper-V host
guarantees a response even though Azure I/O latency may be unbounded.  The
SCSI layer therefore restarts the command timer instead of initiating normal
timeout error handling.  A host-reset handler exists for error recovery
requested through other paths, but repeated ordinary timeouts do not by
themselves escalate to that reset.

Teardown deliberately treats submission and completion differently.
``storvsc_dev_remove()`` first sets ``destroy``.  Outbound lookup then rejects
new requests, while inbound lookup continues accepting responses until
``num_outstanding_req`` reaches zero.  Each completion decrements that count
and wakes the drain waiter when the last request finishes.  Only after this
drain can teardown unpublish driver data, close channels, and free controller
state.  The invariant is: reject new work, but preserve the objects needed to
complete all work already visible to the VSP.

NVMe through virtual PCI
------------------------

Where NVMe enters the stack
~~~~~~~~~~~~~~~~~~~~~~~~~~~~
The generic block and blk-mq discussion above is not SCSI-specific.  Both an
``/dev/sdX`` disk and an ``/dev/nvmeXnY`` namespace receive bios, merge or
split them into requests, use software and hardware blk-mq contexts, allocate
tags, track timeouts, and eventually complete bios.  The paths split at each
disk's ``request_queue.mq_ops``:

.. code-block:: text

  filesystem, direct I/O, swap, or another bio submitter
                              |
                              v
                  bio -> generic block layer
                              |
                              v
                       struct request
                              |
                +-------------+-------------+
                |                           |
      SCSI disk request queue      NVMe namespace request queue
                |                           |
                v                           v
        scsi_queue_rq()               nvme_queue_rq()
                |                           |
       sd builds SCSI CDB          NVMe core builds command
                |                           |
     storvsc builds SRB/PFNs       nvme-pci builds PRPs/SGLs
                |                           |
       VMBus storage rings          PCI submission queue
                |                           |
       Hyper-V storage VSP         MMIO doorbell + controller

Everything above the fork is shared generic block code.  Everything in the
left branch from ``scsi_queue_rq()`` through ``sd``, ``scsi_cmnd``, SCSI error
handling, and ``storvsc`` is specific to SCSI devices.  The right branch uses
the NVMe core and an NVMe transport and does not instantiate those SCSI
objects.

The queues are created by different registration paths.  SCSI creates a
``scsi_device.request_queue`` from ``Scsi_Host.tag_set`` and that tag set
installs the SCSI blk-mq operations.  NVMe/PCI allocates admin and I/O tag sets
whose operations are ``nvme_mq_admin_ops`` and ``nvme_mq_ops``.  When the NVMe
core discovers a namespace, ``nvme_alloc_ns()`` calls ``blk_mq_alloc_disk()``
with the controller's I/O tag set.  The resulting namespace queue therefore
dispatches directly to the NVMe callbacks.

Their per-request memory also differs:

.. code-block:: text

  SCSI/storvsc request                 NVMe/PCI request
  +-------------------------+          +-------------------------+
  | struct request          |          | struct request          |
  +-------------------------+          +-------------------------+
  | struct scsi_cmnd        |          | struct nvme_iod         |
  +-------------------------+          |   struct nvme_request   |
  | storvsc_cmd_request     |          |   struct nvme_command   |
  +-------------------------+          |   DMA descriptor state  |
  | inline SCSI SG storage  |          +-------------------------+
  +-------------------------+

Both use ``blk_mq_rq_to_pdu()`` to reach request-private storage.  The SCSI
tag set sizes that storage for ``scsi_cmnd``, low-level-driver private data,
and scatterlist entries.  The NVMe/PCI tag set uses
``sizeof(struct nvme_iod)``; its first member is the common
``nvme_request``, followed by the native command and PCI DMA-mapping state.

Device discovery
~~~~~~~~~~~~~~~~
Hyper-V initially presents a passed-through PCI device as a VMBus device with
``HV_PCIE_GUID``.  The ``hv_pci`` driver binds and ``hv_pci_probe()`` opens a
VMBus channel to the vPCI VSP.  It negotiates the protocol, obtains the child
PCI function description and BAR requirements, creates a PCI host bridge,
and scans the resulting root bus.

The NVMe function then has both a VMBus identity and a normal Linux PCI
identity.  The generic PCI core matches its PCI class or device ID with the
``nvme`` PCI driver and calls ``nvme_probe()``.  ``nvme_probe()`` maps the
controller BAR, enables the PCI function, creates the admin queue, identifies
the controller and namespaces, creates I/O queues, and registers
``/dev/nvmeXnY`` block devices.

The vPCI VMBus channel is used for presentation, configuration support,
hotplug, and interrupt mapping.  It does not carry ordinary NVMe read and
write commands.  Once configured, the controller uses the same NVMe queue
interface that it would use on bare metal.

This is a different use of Hyper-V from storvsc.  Storvsc's VMBus channel is
the storage I/O transport, so every normal command crosses its shared rings.
For NVMe, the vPCI channel is the device-presentation and management path.
Ordinary I/O is expressed as native NVMe commands in DMA-backed queues; the
PCI driver rings an MMIO doorbell in the mapped controller BAR, and Hyper-V
routes the controller interrupt configured through vPCI.

How the two paths use VMBus differently
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
It is useful to separate the **control plane**, which discovers and configures
a device, from the **data plane**, which submits ordinary reads and writes.
Both paths use VMBus in their control plane, but only storvsc uses a VMBus
channel as its storage data plane.

For synthetic SCSI, VMBus is both planes:

.. code-block:: text

  control plane
    VMBus offer with HV_SCSI_GUID
      -> open primary channel
      -> negotiate storage protocol
      -> query controller properties
      -> create storage subchannels
      -> discover SCSI LUNs

  data plane for every command
    storvsc_queuecommand()
      -> vstor_packet containing Hyper-V SRB and SCSI CDB
      +  VMBus multipage-buffer descriptor containing guest PFNs
      -> VMBus outbound ring
      -> storage VSP

    storage VSP completion
      -> VMBus inbound ring
      -> storvsc_on_channel_callback()
      -> scsi_done()

For NVMe through vPCI, VMBus establishes the PCI environment but is outside
the normal NVMe command path:

.. code-block:: text

  vPCI control plane
    VMBus offer with HV_PCIE_GUID
      -> negotiate vPCI protocol
      -> query bus relations
      -> enter D0 and assign PCI resources
      -> create interrupt mappings
      -> report hot-add, eject, and invalidation events
      -> expose a PCI function to the Linux PCI core

  NVMe data plane for every command
    nvme_queue_rq()
      -> native NVMe command with PRP or SGL DMA addresses
      -> DMA-backed NVMe submission queue
      -> MMIO submission-queue doorbell
      -> NVMe controller

    controller writes NVMe completion queue
      -> configured MSI-X or MSI interrupt
      -> nvme_irq() / nvme_handle_cqe()
      -> nvme_complete_rq()

The ordinary NVMe submission path therefore contains no
``vmbus_sendpacket()`` call.  ``hv_pci`` does use that API for vPCI protocol
messages such as ``PCI_QUERY_PROTOCOL_VERSION``, ``PCI_QUERY_BUS_RELATIONS``,
``PCI_BUS_D0ENTRY``, resource assignment, and interrupt creation.  Its channel
callback receives replies plus asynchronous bus-relation, eject, and
invalidation notifications.  Those operations arrange the device around the
I/O queues; they do not carry NVMe submission or completion queue entries.

The memory descriptors illustrate the same distinction.  Storvsc places a
Hyper-V multipage-buffer descriptor beside the SRB in a VMBus packet.  The
descriptor names the guest pages that the storage VSP accesses, so the bulk
data is not copied through the ring.  NVMe places PCI DMA addresses in PRP or
SGL fields of a native NVMe command already stored in its submission queue.
The controller accesses those pages through its DMA domain; no VMBus storage
packet or Hyper-V PFN array is constructed by ``nvme-pci``.

Completion correlation is also independent:

* Storvsc places the blk-mq unique tag in the VMBus transaction ID.  The VSP
  echoes that ID in an inbound-ring completion packet.
* NVMe places a queue-local request tag plus generation bits in the NVMe
  command ID.  The controller returns it in a completion queue entry.
* vPCI control requests use separate VMBus transaction IDs to wake the
  ``hv_pci`` operation waiting for a protocol response.  Those IDs do not
  identify NVMe block requests.

This changes where pressure is observed.  A full storvsc outbound VMBus ring
can reject an ordinary SCSI command and cause blk-mq to retry it; storvsc may
also choose another storage subchannel.  An NVMe I/O is instead bounded by
blk-mq tags, the NVMe submission queue depth, DMA resources, and controller
state.  Congestion in the vPCI management channel is not the normal flow-
control mechanism for NVMe reads and writes.

.. list-table:: VMBus role in the two Hyper-V storage paths
   :header-rows: 1

   * - Question
     - Synthetic SCSI with storvsc
     - NVMe through vPCI
   * - Why is the VMBus device offered?
     - It is the synthetic storage controller.
     - It is a virtual PCI bus that exposes a child NVMe function.
   * - Does every normal I/O cross a VMBus ring?
     - Yes; command and page descriptors go out and completion comes back.
     - No; native NVMe queues, doorbells, DMA, and MSI carry normal I/O.
   * - What does the VMBus channel protocol describe?
     - Storage initialization, SCSI SRBs, PFNs, and storage completions.
     - PCI discovery, power, resources, interrupts, hotplug, and teardown.
   * - What identifies an ordinary block request?
     - VMBus transaction ID derived from the blk-mq unique tag.
     - NVMe command ID returned in an NVMe completion queue entry.
   * - What callback consumes ordinary completions?
     - ``storvsc_on_channel_callback()`` consumes VMBus packets.
     - ``nvme_irq()`` and ``nvme_handle_cqe()`` consume NVMe CQ entries.

End-to-end NVMe PCI path
~~~~~~~~~~~~~~~~~~~~~~~~
The complete path can be divided into device presentation, controller
bootstrap, I/O queue creation, namespace registration, request submission,
and completion.

Phase 1: expose a PCI function
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
The Hyper-V host first offers an ``HV_PCIE_GUID`` VMBus device.  This device
represents a virtual PCI bus, not the NVMe namespace itself.  ``hv_pci_probe()``
opens the VMBus channel, negotiates the vPCI protocol, obtains bus relations
and resource requirements, enters the bus into D0, reports the selected
resources, and creates a Linux PCI host bridge.

``pci_scan_root_bus_bridge()`` enumerates the child functions described by the
host.  When a child has PCI class ``PCI_CLASS_STORAGE_EXPRESS``, the normal PCI
core matches it with the ``nvme`` PCI driver and calls ``nvme_probe()``.  From
this point the NVMe driver operates on an ordinary ``struct pci_dev``.  It does
not call ``hv_pci`` for each command.

The important object relationship is::

  VMBus hv_device with HV_PCIE_GUID
   -> hv_pcibus_device
     -> Linux PCI host bridge
       -> pci_dev for the NVMe function
         -> nvme_dev
           -> nvme_ctrl

The vPCI objects continue to own PCI configuration, resource, hotplug, and
interrupt-routing state.  The ``nvme_dev`` owns the NVMe controller-specific
queues, BAR mapping, DMA pools, and reset work.

Phase 2: map the controller and bootstrap queue 0
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
``nvme_probe()`` maps BAR 0 and enables PCI memory access and bus mastering.
``nvme_pci_enable()`` reads the NVMe ``CAP`` register to learn properties such
as maximum queue depth and doorbell stride.  The fixed controller registers
are near the beginning of the BAR; queue doorbells begin at
``NVME_REG_DBS``.

NVMe cannot use an admin command to create its first queue, because no command
queue exists yet.  Queue ID 0 is therefore bootstrapped through controller
registers:

1. ``nvme_alloc_queue()`` allocates a completion queue and submission queue.
  They are normally DMA-coherent guest memory.  It initializes the software
  queue indices, phase bit, locks, DMA addresses, and doorbell pointer.
2. ``nvme_pci_configure_admin_queue()`` disables the controller and writes the
  admin queue attributes and DMA bases to ``AQA``, ``ASQ``, and ``ACQ``.
3. ``nvme_enable_ctrl()`` sets ``CC.EN`` and waits for ``CSTS.RDY``.
4. The driver requests an interrupt for queue 0 and marks the queue enabled.
5. ``nvme_alloc_admin_tag_set()`` creates a one-hardware-queue blk-mq tag set
  using ``nvme_mq_admin_ops`` and a ``struct nvme_iod`` payload per request.

On Hyper-V, requesting the MSI or MSI-X interrupt invokes the vPCI interrupt
domain.  ``hv_compose_msi_msg()`` sends a
``PCI_CREATE_INTERRUPT_MESSAGE*`` request over the vPCI VMBus channel and the
host returns the MSI address and data.  This VMBus exchange configures how a
later controller interrupt reaches the guest; the later interrupt itself is
not an NVMe completion packet on that channel.

The admin queue carries commands about the controller rather than normal
namespace traffic.  During initialization, ``nvme_init_ctrl_finish()`` uses it
to identify the controller and learn capabilities and limits.  Admin commands
are also used to negotiate the number of I/O queues, create and delete queue
pairs, identify namespaces, request asynchronous events, and abort timed-out
commands.

Phase 3: create native I/O queue pairs
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
``nvme_setup_io_queues()`` asks the controller how many I/O queues it can
support and reconciles that with CPUs, requested read/write/poll queues,
available vectors, BAR space, and driver limits.  For each queue ID greater
than zero:

1. ``nvme_alloc_queue()`` allocates the DMA-backed SQ and CQ memory.
2. ``adapter_alloc_cq()`` submits an NVMe Create I/O Completion Queue admin
  command, specifying the CQ DMA address, depth, and interrupt vector.
3. ``adapter_alloc_sq()`` submits Create I/O Submission Queue, specifying the
  SQ DMA address and associated completion queue.
4. ``queue_request_irq()`` installs ``nvme_irq()`` for an interrupt-driven
  queue.  A polled queue deliberately has no interrupt.
5. The queue is marked enabled and becomes available to blk-mq.

After queue creation, ``nvme_alloc_io_tag_set()`` installs ``nvme_mq_ops``.
Its hardware-queue count is the controller's number of I/O queues and its
request payload size is ``sizeof(struct nvme_iod)``.  The PCI driver's
``nvme_pci_map_queues()`` maps CPUs onto default, read, and optional poll
queues, using interrupt affinity where an interrupt exists.

The correspondence is direct for this transport::

  blk-mq I/O hctx 0 -> NVMe queue ID 1
  blk-mq I/O hctx 1 -> NVMe queue ID 2
  ...

Queue ID 0 is omitted because it belongs to the separate admin tag set.
Multiple blk-mq hardware contexts therefore drive independent native queue
pairs, reducing shared locks and allowing submissions and completions to stay
near the CPUs assigned to their interrupts.

Phase 4: discover namespaces and create block disks
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
Once the controller reaches ``NVME_CTRL_LIVE``, ``nvme_start_ctrl()`` schedules
namespace scanning.  Identify commands on the admin queue return namespace
IDs and properties such as capacity, logical block formats, metadata, and
feature support.

For each usable namespace, ``nvme_alloc_ns()`` allocates ``struct nvme_ns`` and
calls ``blk_mq_alloc_disk(ctrl->tagset, ...)``.  The resulting ``gendisk`` is
published as a device such as ``/dev/nvme0n1``.  Its request queue uses the
controller's I/O tag set, and its ``queuedata`` points to the namespace.  This
is what lets ``nvme_setup_cmd()`` recover the namespace ID and LBA format from
a generic block request.

One namespace is not one hardware queue.  All namespaces attached to a
controller normally share its I/O tag set and native queue pairs, while each
namespace has its own block disk, capacity, and request queue.

Phase 5: turn a block READ into an NVMe command
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
The generic block layer has already merged or split bios and selected an
``hctx`` before calling ``nvme_queue_rq()``.  The request identifies the
namespace through ``req->q->queuedata`` and the native queue through
``hctx->driver_data``.  Its private ``struct nvme_iod`` contains the common
NVMe request state, one ``struct nvme_command``, and PCI DMA descriptor state.

The submission path is:

.. code-block:: text

  blk-mq struct request
   -> nvme_queue_rq(hctx, request)
     -> nvme_prep_rq()
       -> nvme_setup_cmd(namespace, request)
         -> nvme_setup_rw(..., nvme_cmd_read)
       -> nvme_map_data()
     -> nvme_sq_copy_cmd()
     -> nvme_write_sq_db()

For a READ, ``nvme_setup_rw()`` fills the native 64-byte command with:

* the NVMe read opcode;
* the namespace identifier;
* starting logical block address;
* zero-based number of logical blocks;
* control, protection-information, and data-set-management fields; and
* a command ID derived from the blk-mq request tag.

Unlike the SCSI path, there is no intermediate CDB, SRB, or protocol conversion
by a host storage service.  The command placed in memory is the command format
defined by the NVMe specification.

Phase 6: describe the data with PRPs or SGLs
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
The command must also tell the controller where to DMA the read data.
``nvme_map_data()`` walks the block request's memory segments through the DMA
API.  The resulting DMA addresses are valid in the PCI device's DMA domain;
an IOMMU may translate them, so they are not generally interchangeable with
CPU physical addresses.

The PCI transport chooses one of two NVMe-native descriptions:

``PRP``
  ``PRP1`` names the first data page and may include an offset.  ``PRP2`` names
  either the second page or a DMA-coherent PRP list containing further page
  addresses.  Larger transfers may require chained PRP-list pages allocated
  from the driver's DMA pools.

``SGL``
  An NVMe scatter-gather descriptor can describe an address and length or
  point to a list of further descriptors.  The driver uses SGLs when the
  controller supports them and request layout or command type makes them
  suitable or necessary.

The selected pointer is written into the command's data-pointer fields.  The
``nvme_iod`` remembers allocated descriptor pages and DMA mappings so the
completion or failed-submission path can release exactly those resources.

Phase 7: publish the submission queue entry
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
Each ``nvme_queue`` maintains a circular submission queue in memory and a
software tail index.  Under ``nvmeq->sq_lock``, ``nvme_sq_copy_cmd()`` copies
the prepared command into the current SQ slot and advances the tail.
``nvme_write_sq_db()`` then publishes the new tail to the queue's submission
doorbell.

The doorbell is an MMIO register in BAR 0, not the submission queue itself.
It tells the controller that commands through the new tail are available in
DMA memory.  Required memory ordering ensures the SQ entry is visible before
the doorbell.  If the controller supports the optional doorbell buffer, the
driver can update a DMA-coherent shadow doorbell and avoid an MMIO write when
the controller's event-index rules permit.

The controller then:

1. observes the new SQ tail;
2. fetches the command from guest memory;
3. follows its PRP or SGL descriptors;
4. performs the namespace operation; and
5. DMA-writes a completion queue entry.

For a READ, the controller writes payload data into the mapped request pages.
The CPU does not copy that payload through a VMBus ring or through the BAR.

Phase 8: consume the completion queue entry
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
An interrupt-driven queue causes the controller to issue MSI-X or MSI after
writing one or more completion entries.  Hyper-V routes the interrupt using
the mapping established by vPCI.  ``nvme_irq()`` checks the completion queue
and ``nvme_handle_cqe()`` processes each entry whose phase bit indicates that
it is new.

The completion entry contains its submission queue ID, command ID, status,
result, and the controller's consumed SQ head.  The queue on which the entry
arrived selects the corresponding blk-mq tag table.  ``nvme_find_rq()`` splits
the command ID into a request tag and generation counter, finds the request,
and rejects a stale generation.

.. code-block:: text

  MSI-X or MSI
   -> nvme_irq()
   -> nvme_poll_cq()
   -> nvme_handle_cqe()
     -> queue-local command ID lookup
     -> nvme_pci_complete_rq()
       -> unmap DMA and free PRP/SGL resources
       -> nvme_complete_rq()
         -> finish, retry, fail over, or authenticate
         -> nvme_end_req()
           -> blk_mq_end_request()
           -> bio_endio()

After consuming entries, the driver advances the CQ head and rings the
completion doorbell so the controller knows which slots may be reused.  The
phase bit toggles each time the circular queue wraps, distinguishing a new
entry from an old entry still present in memory.

NVMe completion policy is in the NVMe core, not the SCSI error handler.
``nvme_decide_disposition()`` can finish the request, retry it, fail it over to
another path when native NVMe multipathing applies, or initiate authentication
handling for a fabrics controller.  A final outcome is converted to a block
status and returned through blk-mq to every bio in the request.

Polled I/O
^^^^^^^^^^
A poll queue follows the same command, DMA, SQ, and CQ formats but does not
request an interrupt.  The blk-mq ``poll`` callback invokes ``nvme_poll()`` to
inspect the completion queue from the submitting context.  This can avoid
interrupt latency for applications using polled I/O, at the cost of CPU time.
It is an alternative completion-notification method, not a different storage
protocol.

Timeout and controller reset
^^^^^^^^^^^^^^^^^^^^^^^^^^^^
Blk-mq calls ``nvme_timeout()`` when a request remains outstanding past its
deadline.  The PCI driver first checks controller state and may poll the CQ in
case the completion was written but its interrupt was lost.  Depending on the
request and controller state, it can issue an Abort admin command, extend the
timer, or schedule controller reset.

``nvme_reset_work()`` quiesces I/O, disables the old controller instance,
re-enables PCI and the controller, recreates queue 0, identifies the
controller again, recreates I/O queues, updates the blk-mq hardware-queue
count, and returns the controller to ``NVME_CTRL_LIVE``.  Queue ID 0 therefore
has a stable administrative **role**, but its memory and interrupt can be torn
down and recreated during reset.  Outstanding requests are synchronized with
this lifecycle by queue quiescing, controller state transitions, and blk-mq
tag iteration.

The reset boundary is also where vPCI may be involved again: PCI enablement,
BAR access, function-level reset, and interrupt allocation use the PCI
environment supplied by ``hv_pci``.  Nevertheless, successful namespace I/O
after reset resumes on native NVMe SQ/CQ pairs rather than a VMBus storage
channel.

Request submission
~~~~~~~~~~~~~~~~~~
The NVMe PCI request queue installs ``nvme_queue_rq()`` as its ``blk-mq``
``queue_rq`` operation:

.. code-block:: text

   struct request
       -> nvme_queue_rq()
       -> nvme_prep_rq()
       -> nvme_setup_cmd()
       -> nvme_setup_rw()
       -> nvme_map_data()
       -> nvme_sq_copy_cmd()
       -> nvme_write_sq_db()
       -> NVMe controller

``nvme_setup_rw()`` translates the request into a 64-byte
``struct nvme_command``.  It fills the opcode, namespace ID, starting logical
block address, block count, control flags, and command ID.  The command ID is
formed from the ``blk-mq`` request tag and a generation counter.  Unlike
storvsc's unique tag, an NVMe command ID does not encode the blk-mq hardware
context number because the completion already arrives on a particular NVMe
queue.  ``nvme_handle_cqe()`` chooses that queue's tag table and uses the
command ID to recover the request; the generation bits reject a stale
completion after a tag has been reused.

``nvme_map_data()`` maps the request's data segments with the DMA API and
describes them using NVMe Physical Region Page (PRP) entries or Scatter
Gather List (SGL) descriptors.  The DMA addresses are addresses in the PCI
device's DMA domain.  Their translation and accessibility depend on the
platform DMA and IOMMU configuration; they must not be assumed to be host
physical addresses.

The NVMe submission and completion queues are normally allocated in
DMA-coherent guest memory.  A controller memory buffer may be used for an I/O
submission queue when the controller supports it.  ``nvme_sq_copy_cmd()``
copies the command into the next submission queue entry, and
``nvme_write_sq_db()`` writes the queue tail to an MMIO doorbell in the
controller BAR.  The queue itself is not normally located in the BAR.

The principal data structure transformations are:

.. code-block:: text

   struct bio
       -> struct request
       -> struct nvme_command
       +  PRP entries or SGL descriptors
       -> NVMe submission queue entry

Request completion
~~~~~~~~~~~~~~~~~~
The controller writes a ``struct nvme_completion`` into the DMA-coherent
completion queue and generates an MSI-X or MSI interrupt.  The vPCI interrupt
setup performed by ``hv_pci`` allows Hyper-V to route that interrupt to a
guest vCPU, but completion processing is done by the normal ``nvme-pci``
driver:

.. code-block:: text

   NVMe MSI-X or MSI interrupt
       -> nvme_irq()
       -> nvme_poll_cq()
       -> nvme_handle_cqe()
       -> nvme_pci_complete_rq()
       -> nvme_complete_rq()
       -> blk_mq_end_request()
       -> bio_endio()

``nvme_handle_cqe()`` uses the completion queue entry's command ID to find
the original request.  The PCI driver unmaps the request's DMA mappings, and
the NVMe core interprets the completion status.  It may complete, retry, or
fail over the request.  Final completion propagates the result through
``blk-mq`` to the bios and filesystem.

Comparison
----------

.. list-table:: Hyper-V storage interfaces
   :header-rows: 1

   * - Property
     - Synthetic SCSI
     - NVMe through vPCI
   * - Guest block device
     - ``/dev/sdX``
     - ``/dev/nvmeXnY``
   * - ``blk-mq`` callback
     - ``scsi_queue_rq()``
     - ``nvme_queue_rq()``
   * - Guest protocol
     - SCSI CDB in a Hyper-V SRB
     - Native NVMe command
   * - Data description
     - Hyper-V PFN array
     - NVMe PRP or SGL
   * - Normal I/O transport
     - VMBus shared rings
     - PCI DMA queues and MMIO doorbells
   * - Completion interrupt
     - VMBus synthetic interrupt
     - MSI-X or MSI routed through vPCI
   * - Request correlation
     - VMBus transaction ID and SCSI tag
     - NVMe command ID and ``blk-mq`` tag
   * - SCSI mid-layer
     - Used
     - Not used

The two paths therefore meet at the generic block layer, but an NVMe request
does not traverse ``storvsc`` and a synthetic SCSI request does not traverse
the NVMe core.

Source entry points
-------------------
The main source files for following these paths are:

* ``block/blk-core.c`` and ``block/blk-mq.c`` for generic bio and request
  handling.
* ``drivers/scsi/sd.c`` for construction of SCSI disk commands.
* ``drivers/scsi/scsi_lib.c`` for SCSI ``blk-mq`` dispatch and completion.
* ``drivers/scsi/storvsc_drv.c`` for the synthetic SCSI VSC.
* ``drivers/hv/channel.c`` and ``drivers/hv/ring_buffer.c`` for VMBus packet
  transmission.
* ``drivers/pci/controller/pci-hyperv.c`` for vPCI discovery, configuration,
  hotplug, and interrupt mapping.
* ``drivers/nvme/host/core.c`` for NVMe command construction and generic
  completion handling.
* ``drivers/nvme/host/pci.c`` for NVMe PCI DMA mapping, queues, doorbells, and
  interrupts.