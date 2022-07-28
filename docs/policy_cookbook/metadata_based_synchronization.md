# Metadata-based Synchronization

This example demonstrates how to use metadata to synchronize asynchronous tasks run by the Delay Server.

## How to do it ...

```python
acquire_metadata_lock(*collection, *lock_name)
{
    # TODO: Create an issue for this.
    # e.g. msiSetIfNotSet, msiTestAndSetMetadata, etc

    # Wait until we're able to add the metadata to the target collection.
    while (errorcode(msiModAVUMetadata('-C', *collection, 'add', *lock_name, 'metadata_lock', '')) < 0) {
        msiSleep('1');
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
