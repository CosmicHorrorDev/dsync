_Note: This is all just a WIP that is motivated by me wanting to bacup my QEMU image I use for work_

# Roadmap

- MVP - I'm largely just doing this work, so that I can replace `rdiff-backup` with a faster alternative for my usecase (a single large binary file backup).
  - This will require changes to `fast_rsync` to support streaming, and will likely involve changes to add threading support targeting large files
  - Even just the basic version is generally 5x faster than `rdiff-backup` for the same files
  - A major painpoint is also that I run out of storage when going far back in time since it apparently keeps all intermediate files on disk. This can instead be done by chaining `apply` calls in memory
- After this support will be added for backing up whole directories which can also directly lead into local rsync behavior (generating signatures and diffs in parallel for a full directory)
- After that support for dealing with a remote machine will be done for a single file, then whole directories
- Finally, basic features can be added to bring better feature parity with `rsync`

# Required `fast_rsync` changes

1. Need at least basic streaming support on `diff` output buffer and all of `apply`'s buffers
2. Blake2 support since I don't want to do extra work to ensure files are likely the same

# Nice-to-have `fast_rsync` changes

1. Very fast (de)serialization of signatures
  - Current likely usage would be `rkyv`
2. Multi-threaded `diff`
  - The rolling hash can be done in chunks (with chunk size a couple order of magnitude larger than block size, chunks will have to overlap a bit)
  - Matching indces can be sent to the cryptographic hash worker
  - Matches can still be done left-to-right by monitoring finished indices on the cryptographic hash worker
  - IndexedSignature would likely be a concurrent hashmap like dashmap (it's all read no write, so find an optimal one)
3. Multi-threaded signature generation
  - Much simpler than `diff`. Just go through chunks in parallel to generate the blocks hashes

# `dsync` Notes

## Local Only - Single File

Single file transfer just means copy the file if it doesn't exist. If it does exist and may be the same then do a checksum. If it obviously isn't the same then just do a full copy and replace it

## Local Only - Directory

If it doesn't exist then just recursively copy. If it does exist then ignore files that were the same and just copy over any modifed or new files

## Local to Remote - Single File

Try to determine if files have been copied, moved, etc by comparing sizes and then a hash of some small portions and a full hash (the full hash can just be a hash of the weak and strong hashes for the file to avoid re-doing work)

## Local to Remote - Directory

If there are duplicates in the directory then only send one over the network and copy the file over locally on the remote machine

It's possible that things could be sped up by spinning up a few ssh connections (not one connection per file though)
