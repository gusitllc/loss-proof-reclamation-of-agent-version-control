# Loss-proof reclamation of agent version-control workspaces using permanent-reference durability proofs and an object-borrowing archive

**Defensive Publication** - published to establish prior art.
**Publisher:** Gus IT LLC.

## Abstract

A sweeper reclaims disk space from many version-control clones and worktrees created by automated coding agents without ever deleting work that exists nowhere else. For each workspace it computes the set of local work as every commit reachable from any local reference, any on-disk reference log and any stash entry, and subtracts only commits proven held by a permanent reference on the hosting service: an ancestor of the default branch, or the head or an ancestor of the head of a merged change request. Everything else is written to a verified archive built in a temporary bare repository that borrows the workspace's objects, so the workspace repository itself is never written. Deletion follows a fixed sequence of live re-proof, atomic rename claim, fingerprint re-check, inventory scan, archive and removal, and any error leaves the workspace in place.

## Technical Field

Software development tooling; safe garbage collection of distributed version-control working copies on machines shared by many automated coding agents.

## Background

Fleets of automated coding agents create large numbers of clones and linked worktrees, and machines run out of disk. Common cleanup scripts delete a workspace when its working tree is clean and its branch is pushed, or when the branch's change request has merged. Those tests miss several kinds of work that exist only locally: commits reachable only from reference logs after a reset or rebase, stash entries, edits hidden by index flags that make the working tree look clean, tags and custom references, hand-written hook scripts, valuable ignored files such as local environment files, and dangling objects from a dropped stash. They also trust movable evidence: a remote branch tip can be force-pushed or deleted, and an open change request's head can move, so a commit that is 'on the server' today may be gone tomorrow. Finally, a workspace can change between the moment it is classified and the moment it is deleted, for example when an agent resumes work in it.

## Summary of the Disclosure

Disclosed is a reclamation method in three parts. A fact collector computes each workspace's local work from all local references, reference logs read directly from disk, and stash entries, and fails closed: any unexpected answer from the version-control tool makes the workspace report-only. A proof chain asks the hosting service whether each local tip is held, and only proofs anchored to permanent references count as durable; the durable set is subtracted from local work with a reachability walk, and the remainder must be archived before deletion. An archiver builds a bundle of the remainder inside a temporary bare repository whose alternates file points at the workspace's object store, saves staged and unstaged changes as binary patches, proves the staged patch applies, and copies untracked, valuable ignored and hand-made administrative files. Removal runs only after a live re-proof and an atomic rename claim followed by re-checks on the claimed copy.

## Detailed Description

Candidates are the child directories of configured root directories. Loose files and directories that are not repositories are reported and never touched; protected paths, and any path that contains a protected path or a root, are kept. For each repository the collector first confirms that the version-control tool resolves the directory to itself rather than to a parent, refuses repository shapes the rule cannot reason about (a non-default object hash, an alternative reference storage format, ancestry grafts, or repository-local configuration that would run filters or include other files), and then reads: the symbolic head, every reference with its object, the stash list, the porcelain status, the list of ignored entries, the index flags that hide edits, linked worktrees, submodule data, and the reference log files read straight from the administrative directory so that nothing is truncated and an unborn head is covered. Hand-made entries in the administrative directory, such as non-sample hooks, are recorded. Ignored entries are kept as valuable unless they match a configured regenerable set, and a regenerable-looking directory that contains a nested repository or an environment file is still valuable. A fingerprint is computed over all of these facts. Any command that does not answer cleanly makes the workspace report-only.

For each repository group the sweeper asks the hosting service which change requests exist for the branches seen and runs a proof chain per distinct tip. Durable proofs are tried first: the tip is the head of a merged change request, whose head reference is frozen at merge; the tip is an ancestor of the default branch; or the tip is an ancestor of the merged change request's head. Non-durable proofs follow: the head or an ancestor of an open or closed change request, or the tip or an ancestor of the same-named remote branch. A not-found answer for the default branch is a negative proof; any unanswered step makes the tip unproven, which blocks deletion and is never treated as negative. Only durable proofs are cached between runs.

Local work is the set of commits reachable from all reference tips, stash entries and reference-log commits, minus commits reachable from durably proven tips. The default branch's remote-tracking tip joins the durable set once it is itself proven on the default branch, and remaining unique tips are proven against the default branch up to a bound before the subtraction is repeated. Separate walks distinguish tips not covered by any proof (unpushed) from history held only by reference logs.

Classification lists every reason a workspace may not be deleted yet, from uncommitted, stashed or hidden changes and valuable ignored files through lent objects, linked worktrees and administrative files to unproven or unpushed tips, reference-log-only history and a missing or open change request. A workspace whose change request is merged or closed and whose only reasons are reference-log history or administrative files is archived and then removed; a clean merged one is removed directly. A workspace touched within a recent-activity window is kept unless its owner declared it done and did not touch it again. Other blocked workspaces are archived and removed only after a configured period blocked, and a failed archive is not retried within a back-off window.

The archive is built from the claimed copy. A temporary bare repository is initialised, its alternates file is pointed at the workspace's object directory and its commit-graph use is disabled, so the bundle reads the workspace's objects without writing to its repository. Shallow boundaries are carried only where a parent is genuinely absent. The bundle plan names every non-durable local reference, every unique tip, every stash entry and every dangling commit; dangling trees and blobs are wrapped in synthetic commits inside the temporary repository. Planned references already on the default branch are left out and listed for restoration by name. The bundle is created, size-capped, verified, and its heads are checked to equal every kept reference at its exact commit; the parents of any carried shallow boundary must be proven on the default branch, or the archive fails. Staged and unstaged changes are written as full-index binary patches using plumbing commands that ignore user diff configuration, and the staged patch is checked to apply against the head in a throwaway index file. Untracked, valuable ignored and changed tracked files and administrative extras are copied and size-checked. A manifest with restore steps is written into a fresh directory; any failure removes the partial archive.

Applying a decision runs in order: a kill-switch check; a fingerprint re-check at the original path and a live, uncached re-proof of every tip and default-branch commit the plan relied on; a journal entry; an atomic rename into a trash directory under the root, which fails while another process holds the workspace and leaves it for the next run; on the claimed copy, a fingerprint re-check, a new-borrower check, an inventory scan refusing nested repositories, unrepresentable names and overlong paths, and a dangling-object count that turns a removal into an archive; the archive if required; a journal entry; and removal. Any finding or error renames the workspace back; if its path is taken, the claim is kept and reported.

A mutation table pairs each defect found in review with a one-edit source change that re-introduces it and the scenario test that must then fail. A runner requires a clean tree, runs each test unmutated first, applies the edit, expects failure and restores the source; a stale anchor fails the run.

## Disclosed Features (each disclosed alone and in combination)

1. A method defines the local work of a version-control workspace as every commit reachable from any local reference, any reference log read from disk, and any stash entry.
2. The method treats a commit as durably held only when it is an ancestor of the default branch, the head of a merged change request, or an ancestor of that head, and treats remote branch tips and open change-request heads as held but not durable.
3. The method treats an unanswered proof query as unproven, which blocks deletion, and never as a negative proof, and caches only durable proofs.
4. A system bundles all local work not durably held from a temporary bare repository that borrows the workspace's objects through an alternates file, so that no write is made to the workspace repository.
5. The system verifies that each planned reference is present in the bundle at its exact commit and that any shallow boundary it carries can be restored from the default branch, failing the archive otherwise.
6. The system saves staged and unstaged changes as binary patches, proves the staged patch applies in a throwaway index, and copies untracked, valuable ignored, changed tracked and hand-made administrative files.
7. A method deletes a workspace only after a fingerprint re-check, a live re-proof that bypasses the cache, an atomic rename into a trash directory, and a second fingerprint re-check on the renamed copy.
8. The method scans the claimed copy for nested repositories, unrepresentable names and new object borrowers, and archives rather than deletes when dangling objects are present.
9. The method converts any error from the version-control tool into a report-only outcome and restores the workspace to its original path after any failure following the claim.
10. A system pins each defect found in review to a scenario test through a table of one-edit mutations and fails when a mutation survives or no longer applies.

## Variations

- The durable set may include other references the hosting service guarantees immutable, such as protected release tags, while excluding any reference that can be force-updated.
- The archive may be uploaded to object storage or pushed to a dedicated archive repository instead of written to local disk.
- Copied files may be verified by content digest rather than by size.
- The claim may use a filesystem lock or an exclusive open handle instead of a rename where renames are not atomic.
- Proof queries may be batched through a graph query interface rather than issued per tip.
- The mutation check may be generated from a mutation-testing tool and filtered to mutations matching documented review findings.

## Note on omitted parameters

The recent-activity window, the stale-blocked period, the archive retry back-off, archive size caps, dangling-object and scan caps, concurrency levels and the proof bound are implementation choices and are not stated.
