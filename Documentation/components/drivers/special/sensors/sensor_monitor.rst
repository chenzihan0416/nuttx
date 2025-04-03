==========================
Sensor Monitor (procfs)
==========================

Overview
========

The Sensor Monitor is a dynamic debugging tool implemented as a procfs interface
that allows users to monitor and control sensor logging at runtime. It provides
a flexible way to enable/disable sensor debug output without rebuilding the system,
making it particularly useful for debugging sensor-related issues in production
environments.

The monitor uses a hash table to store monitored sensor topics and their associated
log levels, allowing efficient lookup and modification of monitoring settings.

Features
========

- **Dynamic Configuration**: Add or remove monitored sensors at runtime
- **Log Level Control**: Set different log levels for different sensors
- **Batch Operations**: Add or remove multiple sensors in a single command
- **Procfs Interface**: Easy to use through standard file operations
- **Hash Table Backend**: Efficient storage and lookup using ``hsearch_r()``
- **Persistent Configuration**: Default sensor list can be configured at build time

Configuration
=============

To enable the Sensor Monitor, the following configuration options are required:

.. code-block:: kconfig

   CONFIG_SENSORS_MONITOR=y                    # Enable sensor monitor
   CONFIG_SENSORS_MONITOR_BUCKET=<size>        # Hash table bucket size
   CONFIG_SENSORS_MONITOR_LIST="<sensors>"     # Default monitored sensors

Configuration Options
---------------------

**CONFIG_SENSORS_MONITOR**
  Enable the sensor monitor procfs interface.

**CONFIG_SENSORS_MONITOR_BUCKET**
  Specifies the number of buckets in the hash table used to store monitored
  sensors. A larger value provides better performance for many sensors but
  uses more memory. Recommended values:
  
  - Small systems: 16-32
  - Medium systems: 64-128
  - Large systems: 256+

**CONFIG_SENSORS_MONITOR_LIST**
  A string containing space-separated sensor names to monitor by default.
  This list is applied during initialization.
  
  Example::
  
    CONFIG_SENSORS_MONITOR_LIST="sensor_accel sensor_gyro"

Usage
=====

The Sensor Monitor is accessed through the ``/proc/sensor_monitor`` file.

Reading Status
--------------

To view currently monitored sensors and their log levels:

.. code-block:: bash

   cat /proc/sensor_monitor

This displays:

1. Usage instructions
2. Command examples
3. List of currently monitored sensors with their log levels

Example output::

  Sensor procfs - Dynamic sensor debugging tool
  
  Usage:
    cat /proc/sensor_monitor - Show currently monitored topics
    echo <level> <topic> > /proc/sensor_monitor - Add topic(s)
    echo rm <topic> > /proc/sensor_monitor - Remove topic(s)
    ...
  
  sensor_accel            1
  sensor_gyro             2
  sensor_compass          1

Adding Sensors
--------------

**Add with Default Log Level (1)**

.. code-block:: bash

   echo sensor_accel > /proc/sensor_monitor

**Add with Specific Log Level**

.. code-block:: bash

   echo 2 sensor_accel > /proc/sensor_monitor

Log levels correspond to syslog levels (0-7), where:

- 0 = LOG_EMERG (Emergency)
- 1 = LOG_ALERT (Alert)
- 2 = LOG_CRIT (Critical)
- 3 = LOG_ERR (Error)
- 4 = LOG_WARNING (Warning)
- 5 = LOG_NOTICE (Notice)
- 6 = LOG_INFO (Informational)
- 7 = LOG_DEBUG (Debug)

**Add Multiple Sensors**

.. code-block:: bash

   echo sensor_accel sensor_gyro sensor_compass > /proc/sensor_monitor
   echo 2 sensor_accel sensor_gyro > /proc/sensor_monitor

Removing Sensors
----------------

**Remove Single Sensor**

.. code-block:: bash

   echo rm sensor_accel > /proc/sensor_monitor

**Remove Multiple Sensors**

.. code-block:: bash

   echo rm sensor_accel sensor_gyro > /proc/sensor_monitor

**Remove All Sensors**

.. code-block:: bash

   echo clean > /proc/sensor_monitor

Batch Operations
----------------

You can combine add and remove operations in a single command:

.. code-block:: bash

   echo "add 1 sensor_accel rm sensor_gyro" > /proc/sensor_monitor

This command:

1. Adds ``sensor_accel`` with log level 1
2. Removes ``sensor_gyro``

Implementation Details
======================

Architecture
------------

The Sensor Monitor consists of the following components:

**Procfs Interface**
  Implements standard file operations (open, close, read, write) for
  ``/proc/sensor_monitor``.

**Hash Table Storage**
  Uses POSIX ``hsearch_r()`` family functions to store sensor-topic mappings:
  
  - **Key**: Sensor name (string)
  - **Value**: Log level (integer stored as pointer)

**Command Parser**
  Parses space-separated commands from write operations and executes
  corresponding actions.

Data Structures
---------------

**struct sensor_monitor_buffer_s**

Used for formatting output during read operations:

.. code-block:: c

   struct sensor_monitor_buffer_s
   {
     FAR char *buffer;      /* Output buffer pointer */
     size_t   totalsize;    /* Total bytes written */
     size_t   buflen;       /* Remaining buffer size */
     off_t    offset;       /* Current file offset */
   };

**Hash Table Entry**

Each monitored sensor is stored as an ``ENTRY``:

.. code-block:: c

   ENTRY item;
   item.key = "sensor_accel";           /* Sensor name */
   item.data = (void *)(intptr_t)1;     /* Log level */

Key Functions
-------------

**sensor_monitor_initialize()**

Initializes the sensor monitor subsystem:

1. Allocates and initializes the hash table
2. Sets the custom ``free_entry`` callback
3. Processes the default sensor list from ``CONFIG_SENSORS_MONITOR_LIST``
4. Registers the procfs entry

.. code-block:: c

   int sensor_monitor_initialize(void);

**sensor_monitor_level()**

Retrieves the log level for a specific sensor:

.. code-block:: c

   int sensor_monitor_level(FAR const char *name);

**Parameters:**

- ``name``: Sensor name to query

**Returns:**

- Log level (0-7) if sensor is monitored
- ``LOG_EMERG`` (0) if sensor is not monitored

**Usage Example:**

.. code-block:: c

   int level = sensor_monitor_level("sensor_accel");
   if (level >= LOG_INFO)
     {
       syslog(LOG_INFO, "Accel data: x=%d y=%d z=%d\n", x, y, z);
     }

**sensor_monitor_add()**

Adds a sensor to the monitoring table:

.. code-block:: c

   static int sensor_monitor_add(FAR const char *name, int level);

**Parameters:**

- ``name``: Sensor name
- ``level``: Log level (0-7)

**Returns:**

- ``OK`` on success
- ``-ENOMEM`` if allocation fails

**Behavior:**

- If sensor already exists, updates its log level
- If sensor is new, allocates memory and adds to hash table

**sensor_monitor_remove()**

Removes a sensor from the monitoring table:

.. code-block:: c

   static int sensor_monitor_remove(FAR const char *token);

**Parameters:**

- ``token``: Sensor name to remove

**Returns:**

- ``OK`` on success
- ``-ENOENT`` if sensor not found

**sensor_monitor_clean()**

Removes all monitored sensors:

.. code-block:: c

   static void sensor_monitor_clean(void);

Destroys and recreates the hash table, effectively clearing all entries.

**sensor_monitor_print()**

Callback function used with ``hforeach_r()`` to print each monitored sensor:

.. code-block:: c

   static void sensor_monitor_print(FAR ENTRY *item, FAR void *args);

Integration with Sensor Drivers
================================

Sensor drivers can query the monitor to determine if they should output
debug information:

.. code-block:: c

   #include <nuttx/sensors/sensor.h>
   
   void my_sensor_process_data(FAR struct sensor_data_s *data)
   {
     int level = sensor_monitor_level("sensor_accel");
     
     if (level >= LOG_INFO)
       {
         syslog(LOG_INFO, "Sensor data: x=%d y=%d z=%d\n",
                data->x, data->y, data->z);
       }
     
     if (level >= LOG_DEBUG)
       {
         syslog(LOG_DEBUG, "Detailed sensor state: ...\n");
       }
   }

This allows drivers to:

1. Check if their sensor is being monitored
2. Adjust verbosity based on the configured log level
3. Avoid unnecessary logging overhead when not monitored

Memory Management
=================

**Hash Table**

The hash table is allocated during initialization and persists throughout
system operation. Size is determined by ``CONFIG_SENSORS_MONITOR_BUCKET``.

**Sensor Names**

Sensor names are dynamically allocated using ``strdup()`` when added to the
hash table. The custom ``free_entry`` callback ensures proper cleanup:

.. code-block:: c

   static void sensor_monitor_free_entry(FAR ENTRY *entry)
   {
     kmm_free(entry->key);
   }

**Buffer Management**

Read operations use temporary stack buffers (``NAME_MAX`` size) for
formatting output before copying to user space.

Error Handling
==============

**Write Operation Errors**

- Invalid sensor name: Silently ignored (allows flexible command syntax)
- Allocation failure: Returns ``-ENOMEM``
- Sensor not found (for remove): Returns ``-ENOENT``

**Read Operation**

- Always succeeds
- Returns help text and current monitoring status

**Initialization Errors**

- Hash table allocation failure: Returns ``-ENOMEM``
- Procfs registration failure: Propagates error from ``procfs_register()``

Examples
========

Example 1: Basic Monitoring
----------------------------

.. code-block:: bash

   # Enable monitoring for accelerometer with default level
   echo sensor_accel > /proc/sensor_monitor
   
   # Check status
   cat /proc/sensor_monitor
   
   # Output shows:
   # sensor_accel            1

Example 2: Multi-Sensor Setup
------------------------------

.. code-block:: bash

   # Monitor multiple sensors with different levels
   echo 2 sensor_accel > /proc/sensor_monitor
   echo 3 sensor_gyro > /proc/sensor_monitor
   echo 1 sensor_compass > /proc/sensor_monitor
   
   # Verify
   cat /proc/sensor_monitor

Example 3: Dynamic Adjustment
------------------------------

.. code-block:: bash

   # Start with basic monitoring
   echo sensor_accel sensor_gyro > /proc/sensor_monitor
   
   # Increase verbosity for accelerometer
   echo 7 sensor_accel > /proc/sensor_monitor
   
   # Remove gyro monitoring
   echo rm sensor_gyro > /proc/sensor_monitor
   
   # Add new sensor
   echo 2 sensor_baro > /proc/sensor_monitor

Example 4: Cleanup
------------------

.. code-block:: bash

   # Remove all monitored sensors
   echo clean > /proc/sensor_monitor
   
   # Verify empty
   cat /proc/sensor_monitor

Best Practices
==============

1. **Use Appropriate Log Levels**
   
   - Use lower levels (0-3) for errors and critical events
   - Use higher levels (4-7) for verbose debugging
   - Default level 1 is suitable for basic monitoring

2. **Sensor Naming Convention**
   
   - Use descriptive names matching driver names
   - Common prefix: ``sensor_``
   - Examples: ``sensor_accel``, ``sensor_gyro_0``, ``sensor_temp_cpu``

3. **Performance Considerations**
   
   - Configure adequate bucket size for your sensor count
   - Rule of thumb: buckets ≥ number of sensors / 2
   - Avoid excessive logging in interrupt context

4. **Production Use**
   
   - Set ``CONFIG_SENSORS_MONITOR_LIST`` for critical sensors
   - Use procfs to enable additional debugging when issues occur
   - Clean up after debugging to reduce overhead

5. **Driver Integration**
   
   - Always check monitor level before expensive operations
   - Use appropriate syslog levels matching monitor levels
   - Consider performance impact of frequent lookups

Limitations
===========

1. **Name Length**: Sensor names are limited to ``NAME_MAX`` characters
2. **Memory**: Each monitored sensor requires memory for:
   
   - Hash table entry
   - Duplicated sensor name string
   - Hash table overhead

3. **Concurrency**: No explicit locking in current implementation
   (relies on kernel mutual exclusion)

4. **Persistence**: Monitoring configuration is lost on reboot
   (except default list from ``CONFIG_SENSORS_MONITOR_LIST``)

Troubleshooting
===============

**Problem: echo command has no effect**

- Check if procfs is mounted: ``mount | grep proc``
- Verify write permissions on ``/proc/sensor_monitor``
- Check for error messages in system log

**Problem: Sensor not appearing in list**

- Ensure correct sensor name (case-sensitive)
- Check hash table size (``CONFIG_SENSORS_MONITOR_BUCKET``)
- Verify memory availability

**Problem: High memory usage**

- Reduce ``CONFIG_SENSORS_MONITOR_BUCKET``
- Remove unused sensors with ``echo rm`` or ``echo clean``
- Review monitored sensor count

See Also
========

- :doc:`sensors_uorb` - Main sensor framework documentation
- ``nuttx/include/nuttx/sensors/sensor.h`` - Sensor header file
- ``nuttx/drivers/sensors/sensor_monitor.c`` - Implementation
- ``drivers/fs/procfs/`` - Procfs filesystem implementation
