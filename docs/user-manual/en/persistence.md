# Persistence

In this chapter we will describe how persistence works with Apache ActiveMQ and
how to configure it.

Apache ActiveMQ ships with a high performance journal. Since Apache ActiveMQ handles
its own persistence, rather than relying on a database or other 3rd
party persistence engine it is very highly optimised for the specific
messaging use cases.

An Apache ActiveMQ journal is an *append only* journal. It consists of a set of
files on disk. Each file is pre-created to a fixed size and initially
filled with padding. As operations are performed on the server, e.g. add
message, update message, delete message, records are appended to the
journal. When one journal file is full we move to the next one.

Because records are only appended, i.e. added to the end of the journal
we minimise disk head movement, i.e. we minimise random access
operations which is typically the slowest operation on a disk.

Making the file size configurable means that an optimal size can be
chosen, i.e. making each file fit on a disk cylinder. Modern disk
topologies are complex and we are not in control over which cylinder(s)
the file is mapped onto so this is not an exact science. But by
minimising the number of disk cylinders the file is using, we can
minimise the amount of disk head movement, since an entire disk cylinder
is accessible simply by the disk rotating - the head does not have to
move.

As delete records are added to the journal, Apache ActiveMQ has a sophisticated
file garbage collection algorithm which can determine if a particular
journal file is needed any more - i.e. has all its data been deleted in
the same or other files. If so, the file can be reclaimed and re-used.

Apache ActiveMQ also has a compaction algorithm which removes dead space from
the journal and compresses up the data so it takes up less files on
disk.

The journal also fully supports transactional operation if required,
supporting both local and XA transactions.

The majority of the journal is written in Java, however we abstract out
the interaction with the actual file system to allow different pluggable
implementations. Apache ActiveMQ ships with two implementations:

-   Java [NIO](http://en.wikipedia.org/wiki/New_I/O).

    The first implementation uses standard Java NIO to interface with
    the file system. This provides extremely good performance and runs
    on any platform where there's a Java 6+ runtime.

-   Linux Asynchronous IO

    The second implementation uses a thin native code wrapper to talk to
    the Linux asynchronous IO library (AIO). With AIO, Apache ActiveMQ will be
    called back when the data has made it to disk, allowing us to avoid
    explicit syncs altogether and simply send back confirmation of
    completion when AIO informs us that the data has been persisted.

    Using AIO will typically provide even better performance than using
    Java NIO.

    The AIO journal is only available when running Linux kernel 2.6 or
    later and after having installed libaio (if it's not already
    installed). For instructions on how to install libaio please see Installing AIO section.

    Also, please note that AIO will only work with the following file
    systems: ext2, ext3, ext4, jfs, xfs. With other file systems, e.g.
    NFS it may appear to work, but it will fall back to a slower
    synchronous behaviour. Don't put the journal on a NFS share!

    For more information on libaio please see [lib AIO](libaio.md).

    libaio is part of the kernel project.

The standard Apache ActiveMQ core server uses two instances of the journal:

-   Bindings journal.

    This journal is used to store bindings related data. That includes
    the set of queues that are deployed on the server and their
    attributes. It also stores data such as id sequence counters.

    The bindings journal is always a NIO journal as it is typically low
    throughput compared to the message journal.

    The files on this journal are prefixed as `activemq-bindings`. Each
    file has a `bindings` extension. File size is `1048576`, and it is
    located at the bindings folder.

-   JMS journal.

    This journal instance stores all JMS related data, This is basically
    any JMS Queues, Topics and Connection Factories and any JNDI
    bindings for these resources.

    Any JMS Resources created via the management API will be persisted
    to this journal. Any resources configured via configuration files
    will not. The JMS Journal will only be created if JMS is being used.

    The files on this journal are prefixed as `activemq-jms`. Each file
    has a `jms` extension. File size is `1048576`, and it is located at
    the bindings folder.

-   Message journal.

    This journal instance stores all message related data, including the
    message themselves and also duplicate-id caches.

    By default Apache ActiveMQ will try and use an AIO journal. If AIO is not
    available, e.g. the platform is not Linux with the correct kernel
    version or AIO has not been installed then it will automatically
    fall back to using Java NIO which is available on any Java platform.

    The files on this journal are prefixed as `activemq-data`. Each file
    has a `hq` extension. File size is by the default `10485760`
    (configurable), and it is located at the journal folder.

For large messages, Apache ActiveMQ persists them outside the message journal.
This is discussed in [Large Messages](large-messages.md).

Apache ActiveMQ can also be configured to page messages to disk in low memory
situations. This is discussed in [Paging](paging.md).

If no persistence is required at all, Apache ActiveMQ can also be configured
not to persist any data at all to storage as discussed in the Configuring
the broker for Zero Persistence section.

## Configuring the bindings journal

The bindings journal is configured using the following attributes in
`activemq-configuration.xml`

-   `bindings-directory`

    This is the directory in which the bindings journal lives. The
    default value is `data/bindings`.

-   `create-bindings-dir`

    If this is set to `true` then the bindings directory will be
    automatically created at the location specified in
    `bindings-directory` if it does not already exist. The default value
    is `true`

## Configuring the jms journal

The jms config shares its configuration with the bindings journal.

## Configuring the message journal

The message journal is configured using the following attributes in
`activemq-configuration.xml`

-   `journal-directory`

    This is the directory in which the message journal lives. The
    default value is `data/journal`.

    For the best performance, we recommend the journal is located on its
    own physical volume in order to minimise disk head movement. If the
    journal is on a volume which is shared with other processes which
    might be writing other files (e.g. bindings journal, database, or
    transaction coordinator) then the disk head may well be moving
    rapidly between these files as it writes them, thus drastically
    reducing performance.

    When the message journal is stored on a SAN we recommend each
    journal instance that is stored on the SAN is given its own LUN
    (logical unit).

-   `create-journal-dir`

    If this is set to `true` then the journal directory will be
    automatically created at the location specified in
    `journal-directory` if it does not already exist. The default value
    is `true`

-   `journal-type`

    Valid values are `NIO` or `ASYNCIO`.

    Choosing `NIO` chooses the Java NIO journal. Choosing `AIO` chooses
    the Linux asynchronous IO journal. If you choose `AIO` but are not
    running Linux or you do not have libaio installed then Apache ActiveMQ will
    detect this and automatically fall back to using `NIO`.

-   `journal-sync-transactional`

    If this is set to true then Apache ActiveMQ will make sure all transaction
    data is flushed to disk on transaction boundaries (commit, prepare
    and rollback). The default value is `true`.

-   `journal-sync-non-transactional`

    If this is set to true then Apache ActiveMQ will make sure non
    transactional message data (sends and acknowledgements) are flushed
    to disk each time. The default value for this is `true`.

-   `journal-file-size`

    The size of each journal file in bytes. The default value for this
    is `10485760` bytes (10MiB).

-   `journal-min-files`

    The minimum number of files the journal will maintain. When Apache ActiveMQ
    starts and there is no initial message data, Apache ActiveMQ will
    pre-create `journal-min-files` number of files.

    Creating journal files and filling them with padding is a fairly
    expensive operation and we want to minimise doing this at run-time
    as files get filled. By pre-creating files, as one is filled the
    journal can immediately resume with the next one without pausing to
    create it.

    Depending on how much data you expect your queues to contain at
    steady state you should tune this number of files to match that
    total amount of data.

-   `journal-max-io`

    Write requests are queued up before being submitted to the system
    for execution. This parameter controls the maximum number of write
    requests that can be in the IO queue at any one time. If the queue
    becomes full then writes will block until space is freed up.

    When using NIO, this value should always be equal to `1`

    When using AIO, the default should be `500`.

    The system maintains different defaults for this parameter depending
    on whether it's NIO or AIO (default for NIO is 1, default for AIO is
    500)

    There is a limit and the total max AIO can't be higher than what is
    configured at the OS level (/proc/sys/fs/aio-max-nr) usually at
    65536.

-   `journal-buffer-timeout`

    Instead of flushing on every write that requires a flush, we
    maintain an internal buffer, and flush the entire buffer either when
    it is full, or when a timeout expires, whichever is sooner. This is
    used for both NIO and AIO and allows the system to scale better with
    many concurrent writes that require flushing.

    This parameter controls the timeout at which the buffer will be
    flushed if it hasn't filled already. AIO can typically cope with a
    higher flush rate than NIO, so the system maintains different
    defaults for both NIO and AIO (default for NIO is 3333333
    nanoseconds - 300 times per second, default for AIO is 500000
    nanoseconds - ie. 2000 times per second).

    > **Note**
    >
    > By increasing the timeout, you may be able to increase system
    > throughput at the expense of latency, the default parameters are
    > chosen to give a reasonable balance between throughput and
    > latency.

-   `journal-buffer-size`

    The size of the timed buffer on AIO. The default value is `490KiB`.

-   `journal-compact-min-files`

    The minimal number of files before we can consider compacting the
    journal. The compacting algorithm won't start until you have at
    least `journal-compact-min-files`

    The default for this parameter is `10`

-   `journal-compact-percentage`

    The threshold to start compacting. When less than this percentage is
    considered live data, we start compacting. Note also that compacting
    won't kick in until you have at least `journal-compact-min-files`
    data files on the journal

    The default for this parameter is `30`

## An important note on disabling disk write cache.

> **Warning**
>
> Most disks contain hardware write caches. A write cache can increase
> the apparent performance of the disk because writes just go into the
> cache and are then lazily written to the disk later.
>
> This happens irrespective of whether you have executed a fsync() from
> the operating system or correctly synced data from inside a Java
> program!
>
> By default many systems ship with disk write cache enabled. This means
> that even after syncing from the operating system there is no
> guarantee the data has actually made it to disk, so if a failure
> occurs, critical data can be lost.
>
> Some more expensive disks have non volatile or battery backed write
> caches which won't necessarily lose data on event of failure, but you
> need to test them!
>
> If your disk does not have an expensive non volatile or battery backed
> cache and it's not part of some kind of redundant array (e.g. RAID),
> and you value your data integrity you need to make sure disk write
> cache is disabled.
>
> Be aware that disabling disk write cache can give you a nasty shock
> performance wise. If you've been used to using disks with write cache
> enabled in their default setting, unaware that your data integrity
> could be compromised, then disabling it will give you an idea of how
> fast your disk can perform when acting really reliably.
>
> On Linux you can inspect and/or change your disk's write cache
> settings using the tools `hdparm` (for IDE disks) or `sdparm` or
> `sginfo` (for SDSI/SATA disks)
>
> On Windows you can check / change the setting by right clicking on the
> disk and clicking properties.

## Installing AIO

The Java NIO journal gives great performance, but If you are running
Apache ActiveMQ using Linux Kernel 2.6 or later, we highly recommend you use
the `AIO` journal for the very best persistence performance.

It's not possible to use the AIO journal under other operating systems
or earlier versions of the Linux kernel.

If you are running Linux kernel 2.6 or later and don't already have
`libaio` installed, you can easily install it using the following steps:

Using yum, (e.g. on Fedora or Red Hat Enterprise Linux):

    yum install libaio

Using aptitude, (e.g. on Ubuntu or Debian system):

    apt-get install libaio

## Configuring Apache ActiveMQ for Zero Persistence

In some situations, zero persistence is sometimes required for a
messaging system. Configuring Apache ActiveMQ to perform zero persistence is
straightforward. Simply set the parameter `persistence-enabled` in
`activemq-configuration.xml` to `false`.

Please note that if you set this parameter to false, then *zero*
persistence will occur. That means no bindings data, message data, large
message data, duplicate id caches or paging data will be persisted.

## Import/Export the Journal Data

You may want to inspect the existent records on each one of the journals
used by Apache ActiveMQ, and you can use the export/import tool for that
purpose.
you can export the journal as a text file by using this command:

`java -cp activemq-tools-jar-with-dependencies.jar export-journal <JournalDirectory> <JournalPrefix> <FileExtension> <FileSize> <FileOutput>`

To import the file as binary data on the journal (Notice you also
require netty.jar):

`java -cp activemq-tools-jar-with-dependencies.jar import-journal <JournalDirectory> <JournalPrefix> <FileExtension> <FileSize> <FileInput>`

-   JournalDirectory: Use the configured folder for your selected folder. Example: ./activemq/data/journal

-   JournalPrefix: Use the prefix for your selected journal, as discussed above

-   FileExtension: Use the extension for your selected journal, as discussed above

-   FileSize: Use the size for your selected journal, as discussed above

-   FileOutput or FileInput: text file that will contain the exported data

See [Tools](tools.md) for more information.
