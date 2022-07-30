# Metadata-based Synchronization

These examples demonstrate how to synchronize rules using metadata.

- live systems will have multiple rules being executed
- these techniques help to sync any number of rules
- \*one technique relies on the fact that iRODS doesn't allow duplicate AVUs

## Example 1: Controlling execution order

### How to do it ...

```
is_metadata_attached_to_collection(*collection, *attribute_name)
{
    *attached = false;

    foreach (*row in select COLL_NAME where COLL_NAME = '*collection' and META_COLL_ATTR_NAME = '*attribute_name') {
        *attached = true;
    }

    *attached;
}

wait_for_metadata_signal(*collection, *attribute_name)
{
    while (!is_metadata_attached_to_collection(*collection, *attribute_name)) {
        msiSleep('1', '0');
    }
}

synchronized_delay_rules_example()
{
    *number_of_delay_rules = 5;

    # Schedule multiple delay rules.
    for (*i = 0; *i < *number_of_delay_rules; *i = *i + 1) {
        *signum = *i;

        delay("<INST_NAME>irods_rule_engine_plugin-irods_rule_language-instance</INST_NAME>") {
            wait_for_metadata_signal('/tempZone/home/rods', 'irods::signal_*signum');

            # Log a message to let the admin know the metadata signal was received.
            writeLine("serverLog", "Received signal! (irods::signal_*signum)");

            # Set metadata for the next delay rule.
            *next_signum = *signum + 1;
            writeLine("serverLog", "Signaling next delay rule (irods::signal_*next_signum) ...");
            msiModAVUMetadata('-C', '/tempZone/home/rods', 'add', 'irods::lock_*next_signum', 'unused', '');
        }
    }
}
```

## Example 2: Locks with acquire-release semantics

### How to do it ...

```
acquire_metadata_lock(*collection, *lock_name)
{
    # TODO: Create an issue for this.
    # e.g. msiSetIfNotSet, msiTestAndSetMetadata, etc

    # TODO: This needs to incorporate a timer. Without a timer, the locks can result in a deadlock.
    # One way to handle this is by changing this to have "try" semantics. Basically, try to get the lock
    # and if you can't, return and let the caller decide what to do next. This gives the caller a chance
    # to stop after some number of attempts.
    # 
    # Wait until we're able to add the metadata to the target collection.
    while (errorcode(msiModAVUMetadata('-C', *collection, 'add', *lock_name, 'metadata_lock', '')) < 0) {
        msiSleep('1', '0');
    }

    # At this point, we know we've "acquired" the lock.
}

release_metadata_lock(*collection, *lock_name)
{
    # Try to remove the metadata lock from the collection. The use of "errorcode()" allows us
    # to ignore failures. If the metadata does not exist, it's okay.
    errorcode(msiModAVUMetadata('-C', *collection, 'rm', *lock_name, 'metadata_lock', ''));
}

synchronized_delay_rules_example()
{
    *number_of_delay_rules = 5;

    # Schedule multiple delay rules.
    for (*i = 0; *i < *number_of_delay_rules; *i = *i + 1) {
        *task_id = *i;

        delay("<INST_NAME>irods_rule_engine_plugin-irods_rule_language-instance</INST_NAME>") {
            # Looping helps to simulate contention on the locking mechanism.
            for (*j = 0; *j < *iterations; *j = *j + 1) {
                acquire_metadata_lock('/tempZone/home/rods', 'irods::mutex');
                writeLine("serverLog", "(Delay Rule #*task_id) Acquired lock.");

                # Simulate work.
                msiSleep('2', '0');

                # Release the lock so that other delay rules can enter the critical section.
                writeLine("serverLog", "(Delay Rule #*task_id) Released lock.");
                release_metadata_lock('/tempZone/home/rods', 'irods::mutex');
            }
        }
    }
}
```
