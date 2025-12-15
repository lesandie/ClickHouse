# ObjectStorageQueue auxiliary keeper change rationale

This note explains each line that was added or modified when enabling keeper-path awareness for `StorageObjectStorageQueue` and related classes.

## src/Storages/ObjectStorageQueue/ObjectStorageQueueIFileMetadata.h
- Line 5: Include the forward declaration for `Context` so metadata objects can hold a context without pulling in heavy headers.
- Lines 67-68: Add `keeper_name` and `context` parameters to the constructor so each file metadata instance knows which Keeper connection pool to use.
- Lines 161-162: Store the keeper name and context pointer as members for later Keeper lookups.

## src/Storages/ObjectStorageQueue/ObjectStorageQueueIFileMetadata.cpp
- Lines 134-135: Extend the constructor signature to accept the keeper name and context.
- Lines 143-144: Initialize the new keeper/context members so they can be reused throughout the object lifetime.
- Line 186: Acquire a ZooKeeper client through `getDefaultOrAuxiliaryZooKeeper` using the stored keeper name and context when cleaning up processing nodes in the destructor.
- Line 401: Use the keeper-aware ZooKeeper client when resetting processing state to ensure the correct Keeper cluster is addressed.
- Line 515: Validate processing cleanup in debug builds with a Keeper session tied to the correct keeper name/context.
- Line 544: Mirror the keeper-aware validation in the failed-path finalization debug checks.
- Line 590: Read retry metadata from the appropriate Keeper cluster when marking a file as failed with retries.

## src/Storages/ObjectStorageQueue/ObjectStorageQueueMetadata.h
- Line 11: Forward declare `Context` to allow passing context pointers without full includes.
- Lines 61-62: Extend the metadata constructor to capture the keeper name and context for all per-table metadata operations.
- Line 87: Add the keeper name to the `syncWithKeeper` signature so the correct Keeper cluster is used during metadata synchronization.
- Lines 154-158: Provide overloads of `getZooKeeper` that accept context and keeper name, plus an instance method that reuses stored context/keeper details.
- Line 187: Record the keeper name as a member so the metadata object knows which Keeper pool to target.
- Line 191: Keep a weak reference to the context for reuse when creating new Keeper sessions.

## src/Storages/ObjectStorageQueue/ObjectStorageQueueMetadata.cpp
- Lines 135-136: Extend the constructor signature to accept keeper name and context so metadata objects are tied to a specific Keeper cluster.
- Line 146: Store the keeper name for later Keeper lookups.
- Line 157: Save the provided context into the weak pointer used when resolving Keeper clients.
- Lines 169-185: Update `getZooKeeper` to choose the default or auxiliary Keeper via `getDefaultOrAuxiliaryZooKeeper`, while honoring fault-injection settings on the resolved context.
- Lines 188-194: Add an instance `getZooKeeper` that reuses the stored keeper name and weak context.
- Lines 255-285: When constructing file metadata instances, pass the keeper name and context to ensure per-file operations use the right Keeper cluster.
- Line 460: Capture keeper name in `syncWithKeeper` so metadata initialization also respects auxiliary Keeper selection.
- Lines 492-503: Use the keeper-aware `getZooKeeper` for creating ancestors and reading/writing metadata during synchronization.

## src/Storages/ObjectStorageQueue/ObjectStorageQueueOrderedFileMetadata.cpp
- Line 65: Verify bucket ownership in debug builds using the keeper-aware ZooKeeper client.
- Line 97: Release bucket locks through the correct Keeper cluster during cleanup.
- Lines 156-158: Accept keeper name and context in the ordered file metadata constructor.
- Lines 168-169: Forward keeper name and context to the base class constructor so processing paths use the right Keeper.
- Line 214: Read the processed-node state from the Keeper cluster associated with the table.
- Line 243: Check bucket existence with the keeper-aware client in debug builds.
- Line 254: Acquire bucket locks via the appropriate Keeper connection.
- Line 290: Use the keeper-aware client when setting file processing state.
- Line 310: Perform multi-read checks against the selected Keeper cluster.
- Line 345: Consult the correct Keeper for failed-node existence when reconciling state.
- Line 396: Run multi-op commits through the keeper-aware client while setting processing state.
- Line 510: Migrate bucket metadata using the Keeper session tied to the configured keeper name.
- Line 569: Refresh the Keeper client after session loss using the same keeper selection.
- Line 667: Load failed-node markers with the keeper-aware client when filtering pending files.

## src/Storages/ObjectStorageQueue/ObjectStorageQueueUnorderedFileMetadata.cpp
- Line 21: Accept keeper name in the unordered file metadata constructor so per-file metadata operations know which Keeper to use.
- Line 22: Accept context for retrieving Keeper sessions when needed.
- Lines 33-34: Forward keeper name and context to the base class constructor.
- Line 44: Create processing requests using the keeper-aware ZooKeeper client.
- Line 79: Use keeper-aware Keeper sessions when executing processing multi-requests and retries.
- Line 149: Fetch processed/failed markers via the chosen Keeper cluster when filtering out files.

## src/Storages/ObjectStorageQueue/ObjectStorageQueueSource.cpp
- Line 268: Obtain the ZooKeeper client from `files_metadata`, which now encapsulates keeper selection.
- Line 277: Use the keeper-aware `files_metadata` client for multi-read checks during retries.
- Line 1363: Acquire the ZooKeeper client for commit operations from `files_metadata` so commits go to the correct Keeper cluster.

## src/Storages/ObjectStorageQueue/StorageObjectStorageQueue.h
- Lines 76-80: Introduce `KeeperPathInfo` to bundle keeper name with the Keeper path chosen for the table.
- Lines 82-87: Return `KeeperPathInfo` from `chooseZooKeeperPath` so callers receive both keeper name and path.
- Lines 103-105: Store keeper name and a composite key (`zookeeper_key`) for distinguishing metadata on different Keeper clusters.

## src/Storages/ObjectStorageQueue/StorageObjectStorageQueue.cpp
- Lines 250-252: Capture both keeper name and path from `chooseZooKeeperPath`, and build a composite `zookeeper_key` to disambiguate metadata per Keeper cluster.
- Line 321: Log both Keeper path and keeper name for clarity in server logs.
- Line 324: Pass keeper name into `syncWithKeeper` to ensure metadata initialization targets the selected Keeper cluster.
- Lines 329-338: Construct metadata with keeper name and context so per-table metadata uses the right Keeper.
- Lines 359-364: Use `zookeeper_key` in the metadata factory to avoid collisions across Keeper clusters and to reuse keeper-aware metadata objects.
- Lines 373-377: Remove metadata factory entries using `zookeeper_key`, ensuring cleanup matches the correct Keeper cluster.
- Line 444: Remove metadata via `zookeeper_key` during shutdown to avoid touching tables on other Keeper clusters.
- Line 1384: Return keeper-aware ZooKeeper sessions from `getZooKeeper` so callers use the selected Keeper cluster.
- Lines 1440-1443: Preserve keeper name when serializing settings, embedding it in `keeper_path` unless the default keeper is used.
- Lines 1500-1515: Parse the keeper name (if provided) from settings and default paths while choosing the Keeper path.
- Lines 1523-1527: Update the return type of `chooseZooKeeperPath` to include keeper name along with the normalized Keeper path.

