# Photoshop 27.8 AutoRecover fails on a non-C scratch disk with NAME INVALID when Documents is directly below the drive root

On Windows 26H2, OS build 26340.9843, with Photoshop 27.8.0.13 and E: as the only scratch disk, AutoRecover was enabled but no recovery snapshots were generated. Illustrator recovery worked on the same computer.

Process Monitor captured Photoshop attempting to access:

    E:\PSAutoRecover\E:
    Result: NAME INVALID (0xC0000033)

The current user's Documents known-folder path was E:\Document. Changing only the Documents entry point changed Photoshop's attempted recovery subdirectory. This suggests that this recovery code path may derive a directory component from the Documents parent's name, producing E: when that parent is the drive root. This is a hypothesis supported by the local path trace and intervention, rather than a source-code finding.

Local intervention:

- Created E:\PSFolderCompat\Documents as a junction targeting the existing E:\Document.
- Updated the Personal values in the current user's User Shell Folders and Shell Folders registry keys to that alias, preserving their original value types and backups.
- Kept the original documents in place and retained E: as the scratch disk.

Photoshop then created snapshots under E:\PSAutoRecover\PSFolderCompat. After an actual crash, restarting Photoshop automatically reopened the recovered document with the unsaved layer movement intact. A subsequent fixture-only forced-exit test also recovered the exact layer state.

A temporary C: scratch-disk comparison had recovered successfully before this intervention; switching back to E: reproduced the failure. Simply rebuilding PSAutoRecover on E: did not resolve the invalid path.

Scope: this is one existing installation with pre-existing local binary modifications. Clean-install reproduction of the recovery issue remains unverified. The invalid filesystem path and before/after results are the concrete observations. The Documents alias also changes the Windows known-folder entry point for other applications, although its physical contents stay in their original directory.

Would engineering check how this recovery directory component is derived when the Documents known folder is directly below a volume root?

Related earlier discussion:
https://community.adobe.com/questions-712/auto-recover-still-not-working-when-c-is-not-the-scratch-disk-1082587
