# Requirements for NVM Configuration Service

NOTE: See [embedded-services#267] for more discussion.

[embedded-services#267]: https://github.com/OpenDevicePartnership/embedded-services/issues/267

We will need the ability to store non-volatile data for use with configuration or similar data.

## Order of magnitude numbers

The following numbers should be considered "typical"/"expected" for users of the configuration
service:

* Approximately 4KiB of "value" storage, excluding keys and serialization/storage overhead
* Approximately 100-128KiB of assigned disk space reserved for configuration, assume 8x the
  "resident" storage (important for estimating "free space" usable for wear leveling) after
  considering key storage, serialization overhead, disk format overhead, etc.
* Approximately 50-100 records
* Typical "value"s stored are 16-128B each, with some outliers
* Expected storage device is modern NOR flash, with:
    * 100,000 erase cycles per page
    * 4KiB erase sector
    * 32-64MHz, QSPI interface
        * read: 15-30MiB/s peak
            * Full `O(n)` 128k scan: 4-8ms
            * Quick `O(log(n))` scan (5 pages): 0.64-1.28ms
            * Single `O(1)` 4k read: 128-256us
        * sector (4k) erase: 50ms typ, 500ms max (end of lifespan), 8-80KiB/s
            * Full 128k erase: 1.6-16.0s
            * Might be faster with larger (64k/128k) erases
        * page (<= 256B) write: 500us typ, 5ms max (end of lifespan), 50-500KiB/s
            * Full 128k write: 0.26-2.56s

Assuming 128KiB of total space, 4KiB of resident values, and estimating a 4x overhead of "disk used"
to "values" to account for upper bound of serialization, keys, and disk overhead, this means that
we have an 8x "excess" of storage. This means we should estimate approximately 800k lifetime writes
of full configuration values.

With an upper product lifespan of 20 years (the retention lifespan of the flash part), this would
mean that we have an acceptable page erase interval of once every 13 minutes for the full duration
of the product.

* `20 x 365.25 x 24 x 60` = 10,519,200 "lifetime minutes"
* `8 x 100000` = 800,000 "lifetime page erases" (including excess storage space)
* `10519200 / 800000` = 13.15 "minutes per page erase"

These numbers are still vaguely "ideal", e.g. they assume all writes will efficiently utilize page
space. This would be derated if writes were less efficient, e.g. if there is write amplification
or sub-page writes. Averaging "one erase per hour" is probably reasonable, if latency is acceptable.

## Hard Requirements

This service will need the following qualities ("Hard Requirements"):

* Compatible with external NOR flash, for example SPI/QSPI flash parts
  * Should be compatible with other storage technologies
* Support for basic create/read/update/delete operations
* Ability to perform write/update operations in a power-loss-safe method without corruption
  and minimal data loss (e.g. only unsynced changes at power loss are lost, no older/existing data
  is lost)
* Support for concurrent/shared firmware access
* Support for "addressable" storage, e.g. a file path, storage key, etc.
* Ability to mitigate impact of memory wear
* Compatible with low-power operation
* Ability to support "roll forward" and "roll backwards" operations associated with a firmware
  update or rollback.
    * This may entail a "schema change" or "migration" of stored data
    * In the event of a firmware rollback, it should be possible to use data stored prior to the
      "migration"
    * In the event of a firmware update, it should be possible to use data stored prior to the
      "migration"
* Handling of Serialization/Deserialization steps, e.g. provide a "rust data" interface, translating
  to/from the "wire format" stored at rest

## Potential requirements

The following are items that have been discussed, but may not be hard requirements, or may require
additional scoping. This is to be discussed.

* Ability to "secure erase" individual files/records
    * This would be to ensure certain data could not be retrieved after deletion,
      but before the physical storage page has been re-used and fully erased
    * How would this feature interact with roll forward or roll back abilities?
    * Is this feature also power-loss-safe? e.g. loss of power AFTER "soft"
      delete occurs, but before "secure" delete occurs?
* Ability to handle "bad block" detection, vs "general wear leveling"
    * NOR flash is less susceptible than other mediums like NAND flash for random bad blocks
    * Do we need to handle resiliency in the face of individual bad blocks, or only adopt a
      general wear-leveling pattern to avoid changes to common locations?
    * littlefs handles this by reading-back after every write, and picking a new location if
      the write fails. This is less efficient than keeping a running table of bad blocks,
      but also less complex)
* It should be possible to load all necessary configuration at boot, with a target boot time of
  100-500ms.
    * For example, if we have 64 items, and loading is always `O(n)`, this means we must perform
      64 x full scans: 256-512ms
    * For example, if we have 64 items, and loading is always `O(log(n))`, this means we must perform
      64 x quick scans: 41-82ms
    * For example, if we have 64 items, and we perform an initial `O(n)` scan followed by
      64 x `O(1)` single reads: 12-24ms
    * These numbers do NOT account for effects of "caching", if relevant to the underlying storage
      library. Caching of keys and values could be performed, potentially hydrated in an initial
      linear `O(n)` scan, speeding later reads/writes at the cost of more RAM usage for cache.
    * These number do NOT account for any initial "mounting" or `fsck`-like repair operations which
      may be necessary, nor the time required for any `mkfs`-like initialization on first boot.
* Ideally after initial "hydration", we should limit or eliminate the need to re-load values from
  flash.
    * This design choice means that we would prefer statically reserving enough RAM space to store
      ALL active configuration items, rather than an "ephemeral" configuration where only the
      currently needed configuration items are "live"/"resident" in RAM/cache.
* Care should be taken to avoid unnecessary writes, and likely support some kind of intelligent
  or explicit limitiation of "flushes"
    * The first aspect of this is avoiding "low efficiency" writes whenever possible, for example if
      writing a single 16B value occupies a whole 4K page (with no ability to later append to the
      same page).
    * The second aspect of this is "debouncing" writes to flash, for example if a value is written
      four times in 100ms, we would prefer to not to "flush" these writes to flash, even if it could
      be done in a "high efficiency" manner, where all four writes would be made to the same page
      if possible.
    * This "flushing" behavior could be provided automatically by the underlying storage library,
      or could be exposed to the application, allowing for manually flushing when reasonable.
    * It should be possible to "force" a flush when necessary, even if doing so would be
      sub-optimal, for example in the case of a shutdown event.

## Unknown Qualities

The following are unknown items that may help guide decisions made for implementation.

### How broad of a scope is "Configuration Storage"?

Note: This is now resolved, see "Order of magnitude numbers" above.

### What kind of access patterns are we expecting?

Note: This is now resolved, see "Order of magnitude numbers" above.

### How to implement storage, and what "layers" do we expect?

Depending on answers to previous questions, I can see three main ways to implement the configuration
storage service:

1. A specific implementation that handles things from the high level API down to the low level flash
   operations. It will need to handle metadata, performing reads/erases of specific physical flash
   locations, etc.
2. Separate the "configuration API" from the "disk storage layer". This means that the configuration
   subsystem doesn't need to know "how" data is stored, and instead thinks in terms of "files" or
   "records. It would be built on top of some kind of storage layer, like:
    * a) A general flash filesystem ("ffs") that provides "files" and potentially "folders", e.g.
      littlefs, spiffs, pebblefs, etc.
    * b) A general key:value store ("kvs" that provides "records", acting similar to a
      `HashMap<Vec<u8>, Vec<u8>>`, e.g. sequential-storage::Map, ekv, etc.

If we ONLY need configuration storage (in some scope-able usage pattern), it's likely most
reasonable to go with option 1: we can tightly integrate the operation of the storage API with the
on-disk format, allowing us to have a smaller code footprint, and ideally fewer "integration seams"
that can lead to bugs.

If we are likely to need storage for other purposes, it might make sense to extract the common "disk
storage layer", and use a common ffs/kvs layer, to reduce total code size and "moving parts", and
implement configuration as a layer on top of that.

However this "sharing" is to be taken with a grain of salt: if our use cases are different enough,
e.g. a "large, append-only log store" and a "small, write-once-read-many config store", then
"sharing" the ffs/kvs layer may actually end up worse off than have two smaller bespoke
implementations. This "false sharing" case should be avoided, as it will make the ffs/kvs layer more
complicated than it would otherwise need to be.
