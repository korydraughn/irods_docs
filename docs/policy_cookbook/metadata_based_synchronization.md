# Metadata-based Synchronization

This example demonstrates how to use metadata to synchronize asynchronous tasks run by the Delay Server.

## How to do it ...

```python
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

acquire_metadata_lock(*collection, *lock_name)
{
    # TODO: Create an issue for this.
    # e.g. msiSetIfNotSet, msiTestAndSetMetadata, etc

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
    
}
```
